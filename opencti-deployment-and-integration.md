# OpenCTI Deployment & Integration

**Environment:** Docker-based OpenCTI Community Edition (v7.260701.0), Ubuntu VM **Related systems:** Wazuh SIEM , NVD/CVE

### 1. Base Installation

* I deployed OpenCTI via Docker Compose following the official installation guide, Docker method, on Ubuntu. Below are the steps
  * Installed Docker Engine + Compose plugin via the official Docker apt repo.
  * Cloned the OpenCTI Docker repo, copied `.env.sample` → `.env`.
  * Generated required secrets:
    * &#x20;`OPENCTI_ADMIN_TOKEN` (uuidgen),
    * &#x20;`OPENCTI_ENCRYPTION_KEY` (openssl rand -base64 32),&#x20;
    * `PLATFORM_REGISTRATION_TOKEN` (openssl rand -hex 32),&#x20;
    * strong passwords for MinIO, RabbitMQ, OpenSearch,&#x20;
    * &#x20;`OPENCTI_HEALTHCHECK_ACCESS_KEY`.
* For this deployment, I skipped **XTM One** (Filigran's AI/agentic orchestration layer) since it wasn't needed for a first install and requires more resources. This is however the next thing to explore.

#### Issues During Base Installation:

Upon spinninig up the setup, the OpenCTI container was "unhealthy"

* Root cause: The `OPENCTI_HEALTHCHECK_ACCESS_KEY` contained a `+` character, which broke URL query-string parsing in the healthcheck (`GET /health?health_access_key=...`).
* Fix: I regenerated the key using `openssl rand -hex 32` and recreated the container.&#x20;

### 2. Connectors Installed

| Connector                                                          | Type                | Notes                                                                                                                                |
| ------------------------------------------------------------------ | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| MITRE ATT\&CK                                                      | External import     | Default                                                                                                                              |
| AlienVault OTX                                                     | External import     | Needs API key; initially set to pull since Jan 2025 → caused 2.1M message backlog; later trimmed to a 60-day window and queue purged |
| URLhaus (abuse.ch)                                                 | External import     | No key needed                                                                                                                        |
| ThreatFox (abuse.ch)                                               | External import     | No key needed                                                                                                                        |
| CVE / NVD                                                          | External import     | Needs NVD API key; `CVE_PULL_HISTORY=false`                                                                                          |
| VirusTotal                                                         | Internal enrichment | Manual trigger (not automatic)                                                                                                       |
| MalwareBazaar Recent Additions                                     | External import     | Needs abuse.ch Auth-Key                                                                                                              |
| RansomwareLive                                                     | External import     | No key needed                                                                                                                        |
| Wazuh (community connector, `misje/opencti-wazuh-connector:0.3.0`) | Internal enrichment | This enriches Wazuh alerts                                                                                                           |

The Common setup pattern for each connector is: &#x20;

* generate a UUID (`uuidgen`),&#x20;
* add a service block to `docker-compose.yml`,&#x20;
* bring up the service `docker compose up -d <service>`.

### 3. Wazuh Integration

#### Direction 1: Wazuh queries OpenCTI&#x20;

**Goal:** When a Wazuh alert fires (matching configured rule groups), Wazuh looks up the relevant IOC against OpenCTI in real time and attaches the result to the alert.&#x20;

**Components:**

I used  custom scripts on the **Wazuh DR server**. `/var/ossec/integrations/`&#x20;

* `custom-opencti.py` / `custom-opencti` scripts from `github.com/anyam17/wazuh-opencti-integration`,&#x20;
* OpenCTI API token stored in `/var/ossec/integrations/opencti.env`, permissions locked to `root:wazuh`, `640`.&#x20;
* I added the following integration block `ossec.conf` integration block:

```xml
  <integration>
    <name>custom-opencti</name>
    <group>sysmon,suricata,web,syscheck,virustotal,sshd</group>
    <hook_url>http://172.16.20.66:8080/graphql</hook_url>
    <alert_format>json</alert_format>
  </integration>
```

* I added custom detection rules added at `/var/ossec/etc/rules/opencti_rules.xml` with rule IDs `110000`–`110025`. These are tiered by match type and confidence score i.e rule  `110020` into a 75–89 "high confidence" tier and a new `110025` 90–100 "critical" tier).

**Quick win:**&#x20;

I simulated sshd failure log using an observed IP in OpenCTI `217.60.241.73` .This produced a correctly matched, fully documented Note/Sighting in OpenCTI with the real alert JSON, TLP:AMBER marking, and "Confirmed By Other Sources" confidence see the screenshot below.

**NOTE:** this integration script is explicitly marked "proof of concept" by its author. I also noted it requires alot of rule tuning to correctly get an alert. Therefore, I explored the below option.

### Direction 2: OpenCTI queries Wazuh (retroactive threat hunting)

**Goal:** When a new/updated Indicator lands in OpenCTI from any feed, search Wazuh's **log history** for that IOC.&#x20;

This catches any matches Wazuh's own real-time rules might have missed, including historical ones.

**Connector:** `ghcr.io/misje/opencti-wazuh-connector:0.3.0`  --  Deployed on the DR

#### Setup Approach

* I created a **dedicated, read-only OpenSearch user** (`cti_connector`) on the Wazuh Indexer, role scoped to `read` on `wazuh-alerts-*` index pattern, `cluster_composite_ops_ro` (or `cluster_monitor`) cluster permission.
  * Verified read access works, write access is blocked.

#### During this config, I hit some Ls

* Root-level settings (not under `opencti`/`connector` namespaces) needed the `WAZUH_` prefix — initially wrote `MAX_TLP`/`APP_URL` without it, causing `Field required` validation errors on `max_tlp`/`app_url` since the connector never saw them.
* `Indicator` was initially included in `CONNECTOR_SCOPE`, causing a recurring (harmless but noisy) error: `ValueError: Indicator is not based on any observables` — because Indicators don't carry a directly searchable value; only their linked Observable does. Removed `Indicator` from scope; Observable-type enrichment alone covers the same ground successfully.
* `CONNECTOR_AUTO=true` only fires on entity **creation/update**, not continuously — existing entities are never automatically re-checked. This led to building the re-enrichment script (Section 6).

#### Quick Win:

I did a real end-to-end test:&#x20;

* Injected a synthetic sshd log line for IP `217.60.241.73` into `/var/ossec/logs/active-responses.log` on the OpenCTI server (I installed a Wazuh agent on the server for monitoring),&#x20;
* This was picked in OpenCTI, under observables. A Sighting Note was created with matched field `data.srcip`, rule `5760`, TLP:AMBER, "Confirmed By Other Sources" confidence.

To enhance the enrichment, I added a Vulnerability/CVE correlation

* Confirmed Wazuh's native **Vulnerability Detection module** on the manager is already enabled (`/var/ossec/etc/ossec.conf`):
* Added `Vulnerability` to the Wazuh connector's `CONNECTOR_SCOPE`, plus `WAZUH_VULNERABILITY_INCIDENT_CVSS3_SCORE_THRESHOLD=7.0` to auto-create Incidents for confirmed, severe (CVSS ≥7) vulnerabilities actually present on real hosts.



### 4. Re-Enrichment Script (state-tracked, scheduled)

**Problem:** The enrichment connectors in OpenCTI don't have a native periodic re-check. They only fire once, at entity creation.&#x20;

**Solution:** Custom Python script (`wazuh_reenrich.py`) using `pycti`, with **state tracking**:

* Stores the last run's timestamp in a local state file (`~/.wazuh_reenrich_state`).
* Each run queries OpenCTI server-side for observables with `created_at` or `updated_at` **after** the last run's timestamp only.
* The first run (no state file yet) falls back to a configurable lookback window (`LOOKBACK_HOURS`, set to 2).
* State file is only updated after a fully successful run, so a crashed run doesn't silently skip its window.
* Triggers `ask_for_enrichment` against the Wazuh connector for each matching observable, with a configurable delay (`ENRICH_DELAY_SECONDS=0.3`) between calls.

**Quick win:** first run correctly found 0 IPv4/IPv6 (none new) and 32 Domain-Name observables changed in the lookback window, and began triggering enrichment successfully.

**Deployment:** intended to run via `cron` every 2 hours; uses the existing user admin `OPENCTI_TOKEN`&#x20;

***

## Infrastructure Issues Diagnosed & Fixed

#### a) RabbitMQ healthcheck timeouts (early, recurring theme)

* `rabbitmq-diagnostics ping` was intermittently taking 36–51s against a 30s timeout. I diagnosed this as a likely DNS resolution stalling inside the container which is a known Docker/RabbitMQ pattern.
* Fix. To work around the timeout, I added  `--no-deps`  to the config to unblock dependent container startups.

#### b) Docker-internal DNS resolution failures (later, confirmed pattern)

* CVE connector repeatedly failed with `ConnectionTimeoutError` / `NameResolutionError`  when reaching to `services.nvd.nist.gov` from inside its container. The same request succeeded fine from the host.
* **Fix:** I added explicit DNS servers to Docker's daemon config:

```json
  // /etc/docker/daemon.json
  { "dns": ["8.8.8.8", "1.1.1.1"] }
```

* Restarted Docker daemon (restarts all containers) and confirmed via `/etc/resolv.conf` inside a container showing `ExtServers: [8.8.8.8 1.1.1.1]`.



#### c) Elasticsearch `version_conflict_engine_exception` on CVE connector "Work" tracking documents

* Recurring `DATABASE_ERROR: Update indexing fail` in CVE connector logs, traced to Elasticsearch rejecting concurrent writes to a single shared `work_...` progress-tracking document — caused by the CVE connector's `asyncio.TaskGroup` firing many concurrent progress updates at once.
* Confirmed via GitHub search: a known, longstanding class of issue in OpenCTI's history&#x20;
* **Important distinction:** this only affects progress/telemetry tracking, not the actual CVE/Vulnerability data ingestion itself (which flows via RabbitMQ → workers, unaffected) and hence I did not chase further. (it's a known OpenCTI core-engine limitation, not fixable from the connector/config side, and doesn't block real data flow.)

#### &#x20;d) Redis Memory Incident (Login Slowness)

**Symptom:** I encountered a sluggish login i.e it took 5–10 minutes after entering credentials.

**Root cause:** The `stream.opencti` (OpenCTI's internal Redis event stream) had grown to **2,000,002 entries / \~7.68GB**, which is OpenCTI's own default cap. This was just too large for this VM's RAM. Combined with Elasticsearch's \~4GB heap, this exhausted RAM (15GB total) and fully filled swap (4GB), causing severe thrashing.

**Fix:**

1. I changed the default cap to 500000: Added `REDIS__TRIMMING=500000` to `.env`.
2. Manually trimmed immediately for relief:

```bash
   docker compose exec redis redis-cli XTRIM stream.opencti MAXLEN 500000
```

3\. Restarted OpenCTI to pick up the new setting.

**Result:** Stream dropped to 500,024 entries (\~2.6GB); overall RAM usage dropped from 13Gi → 8.7Gi used; swap relieved from full → 1.9Gi free. Login became fast again.

**NOTE:** To fully clear Swap memory, I had to do a full VM restart.

### Enterprise Edition Gating Discovered

I wished to integrate an SSO authentication mechanism but later confirmed via OpenCTI's own migration docs: as of **v7.260224.0**, all SSO/authentication strategies beyond local username/password (SAML, OpenID, Auth0, and proxy-header auth) became **Enterprise Edition–only** features.&#x20;

Temporal Solution: LDAP

### FortiSIEM Integration

This is the next thing in line. Here is what I am considering

* **Native path :** TAXII 2.1 feed, one-way OpenCTI → FortiSIEM.&#x20;
  * Create a TAXII Collection in OpenCTI &#x20;
  * Configure FortiSIEM's to pull from the collection's feed URL on a schedule.`Resources → Malware IPs/URLs/Domains/Hash → OpenCTI [type]`&#x20;
* **Real-time,  "FortiSIEM queries OpenCTI per alert" integration:**&#x20;
  * This is  _not_ a native FortiSIEM feature (unlike Wazuh's integrator daemon) and would require custom middleware (webhook receiver + OpenCTI GraphQL client + FortiSIEM Incident Update API).&#x20;

