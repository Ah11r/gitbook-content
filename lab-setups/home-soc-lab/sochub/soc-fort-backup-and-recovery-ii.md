# SOC Fort Backup & Recovery II

## SOC Fort: Backup & Recovery Guide v2

Rebuild of the SOCFortress CoPilot lab stack from the private GitHub repo `soc-fort-copilot` (branch `soc-platform-build`) onto a fresh Ubuntu 24.04 VM. v2 folds in everything found while recovering onto Linode. Follow it in order and the problems below do not recur.

> Conventions. Project folder: `/home/root/soc-platform/CoPilot` (`$PROJ`). `~` is `/root` for the root user, so backups in `~` live in `/root`. `YOUR_LINODE_IP` is the server's public address. No passwords appear in this guide: keep them in a password manager.

### 1. What v1 got wrong or missed

| #  | Gap in v1                                                                               | Effect                                                             | Fix in v2                                |
| -- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------- |
| 1  | Admin ports open to the world; wrong dashboard port (8444 vs 8443); 1515 missing        | Exposure, agents cannot enrol                                      | Section 3                                |
| 2  | IRIS certificate never backed up                                                        | `iris-nginx` crash loop ("No such file")                           | Section 5, step 9                        |
| 3  | Grafana OpenSearch plugin not in `GF_INSTALL_PLUGINS`                                   | Dashboards: "Datasource ... was not found"                         | Section 4, change B                      |
| 4  | `config/fluent-bit.conf` committed empty                                                | Nothing forwarded to Graylog                                       | Section 4, change E; Section 5, step 11  |
| 5  | Manager log volume mounted at `/var/wazuh-manager/logs`, real path is `/var/ossec/logs` | Fluent Bit saw an empty folder                                     | Section 4, change A                      |
| 6  | Repo's `config/filebeat.yml` is the stock file, not the patched one                     | After a manager recreate: x509 errors, no alerts reach the indexer | Section 4, change F; Section 5, step 8   |
| 7  | Two OpenSearch stores (Graylog's own vs the Wazuh Indexer)                              | Customer index invisible to CoPilot and Grafana                    | Single store: Section 4, changes C and D |
| 8  | Indexer certificate valid only for an IP (`SAN: IP only`)                               | Graylog cannot verify the name `wazuh-indexer`                     | Section 5, step 6                        |
| 9  | Customer groups lost whenever the manager container is recreated                        | No customer label, no routing                                      | Section 5, step 12                       |
| 10 | Most manager volumes mounted at wrong `/var/wazuh-manager/...` paths                    | Manager config is not persistent (open)                            | Section 9                                |

### 2. What the backup contains

Commit to the private repo:

* `docker-compose.yml` with all the Section 4 changes already applied
* `.env.gpg` (re-encrypt after every change to `.env`)
* `wazuh-certificates/` including the **re-issued** `wazuh-indexer.pem` (SAN fix). Contains private keys and `root-ca.key`: the repo must stay private
* `config/fluent-bit.conf` (Section 5, step 11)
* `config/filebeat.yml`: the **patched** version (copy from the running manager: `docker cp copilot-wazuh-manager-1:/etc/filebeat/filebeat.yml config/filebeat.yml`). It contains the indexer login: private repo only
* `config/wazuh-groups/<Group>/agent.conf` for each customer group (copy from the manager: `/var/ossec/etc/shared/<Group>/agent.conf`)

Not in the repo, and why:

* Docker volume data: Wazuh alerts, Graylog messages, MongoDB, MySQL, Grafana, IRIS cases. Every rebuild starts with empty databases
* IRIS certificate pair: cheap to regenerate (step 9)
* Graylog truststore `cacerts-wazuh`: derived file in a volume, regenerate (step 7)
* GPG passphrase and every password: password manager only

### 3. Linode and firewall

Plan used: 4 vCPU / 16 GB RAM, Ubuntu 24.04, 50 GB disk.

Linode Cloud Firewall (default inbound policy: Drop, outbound: Accept with no rules):

| Rule      | Ports                                                       | Source                        |
| --------- | ----------------------------------------------------------- | ----------------------------- |
| SSH       | TCP 22                                                      | your own IP (`/32` if stable) |
| Admin UIs | TCP 443, 3000, 4433, 8443, 9001                             | your own IP only              |
| Agents    | TCP 1514, 1515 (add TCP/UDP 12201 and UDP 514 only if used) | your agents' networks         |

Do not open 55000 (Wazuh API), 9200 (indexer), 8086 (InfluxDB), 3306 or 5000: containers reach each other over the Docker network. Docker-published ports bypass `ufw`, so rely on the Cloud Firewall. Keep the Lish console available in case a rule locks you out.

Terminal hygiene:

```bash
# ~/.ssh/config on your workstation
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 4
# on the server
sudo apt install -y tmux && tmux new -s soc      # reattach: tmux attach -t soc
```

### 4. Compose and config changes (already in the repo copy)

**A. Manager logs volume** (the manager block, the `wazuh-logs` line):

```yaml
      - wazuh-logs:/var/ossec/logs        # was /var/wazuh-manager/logs
```

**B. Grafana plugin** (Grafana environment):

```yaml
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource,grafana-opensearch-datasource
```

(`grafana-simple-json-datasource` is Angular and is refused by this Grafana version; it is harmless and can be removed.)

**C. Graylog store address** (Graylog environment): credentials come from `.env`, never from the compose file:

```yaml
      - GRAYLOG_ELASTICSEARCH_HOSTS=https://${GRAYLOG_INDEXER_USER}:${GRAYLOG_INDEXER_PASSWORD}@wazuh-indexer:9200
```

**D. Graylog truststore** (Graylog environment, new line). The image's entrypoint appends `GRAYLOG_SERVER_JAVA_OPTS` to its defaults. `changeit` is Java's public default store password, not a secret:

```yaml
      - GRAYLOG_SERVER_JAVA_OPTS=-Djavax.net.ssl.trustStore=/usr/share/graylog/data/config/cacerts-wazuh -Djavax.net.ssl.trustStorePassword=changeit
```

**E. Fluent Bit config file** goes in `config/fluent-bit.conf` (content in step 11).

**F. Filebeat patch**: `config/filebeat.yml` must be the patched file: `hosts: [https://wazuh-indexer:9200]`, `username`/`password` set, `ssl.verification_mode: none`.

**G. New `.env` variables** (values in the password manager; use only URL-safe characters, otherwise URL-encode):

```
GRAYLOG_INDEXER_USER=admin
GRAYLOG_INDEXER_PASSWORD=<indexer admin password>
```

`.env` is git-ignored (`.gitignore` line 12). Only `.env.gpg` is committed.

> Known, unfixed: Graylog's heap line `GRAYLOG_JAVA_OPTS=-Xms512m -Xmx1g` is **not read** by this image (it reads `GRAYLOG_SERVER_JAVA_OPTS`), so the cap is ignored. Not changed yet.

### 5. Recovery procedure

Estimated time: 90 to 120 minutes. Work in tmux. After each step, check its result before the next.

1. **Provision the VM** (Section 3 plan) and create the firewall.
2. **Install Docker Engine and Compose v2** `[v1]`.
3. **Clone the repo, branch `soc-platform-build`** `[v1]`.
4. **Decrypt `.env`** `[v1]`.
5. **Set the server address** in `.env` (`SERVER_HOST`: the compose file uses it for Graylog's external URL) `[v1]`; **create required directories** `[v1]`.
6. **Indexer certificate.** The repo should already hold the re-issued `wazuh-indexer.pem`. If you ever need to regenerate it (new CA, or a name change), sign it with the existing CA and **keep the subject** identical (`CN=wazuh-indexer`); only the SAN list changes. Staging first, never in place:

```bash
PROJ=/home/root/soc-platform/CoPilot
mkdir -p ~/cert-backup ~/cert-staging && chmod 700 ~/cert-backup ~/cert-staging
cp -a $PROJ/wazuh-certificates/. ~/cert-backup/
cd ~/cert-staging
openssl req -new -key $PROJ/wazuh-certificates/wazuh-indexer-key.pem \
  -subj "/C=US/L=California/O=Wazuh/OU=Wazuh/CN=wazuh-indexer" -out indexer.csr
printf "subjectAltName=DNS:wazuh-indexer,DNS:localhost,IP:127.0.0.1\nbasicConstraints=CA:FALSE\nkeyUsage=digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment\n" > san.ext
openssl x509 -req -in indexer.csr -CA $PROJ/wazuh-certificates/root-ca.pem \
  -CAkey $PROJ/wazuh-certificates/root-ca.key -CAserial ~/cert-staging/ca.srl -CAcreateserial \
  -days 3000 -sha256 -extfile san.ext -out wazuh-indexer.pem
openssl verify -CAfile $PROJ/wazuh-certificates/root-ca.pem wazuh-indexer.pem
diff <(openssl x509 -in wazuh-indexer.pem -noout -pubkey) <(openssl pkey -in $PROJ/wazuh-certificates/wazuh-indexer-key.pem -pubout) && echo "cert matches key"
# swap in place (same inode: the file is bind-mounted into the container), then restart ONLY the indexer
cat ~/cert-staging/wazuh-indexer.pem > $PROJ/wazuh-certificates/wazuh-indexer.pem
docker restart copilot-wazuh-indexer-1
```

Keep `-days` so the leaf expires **before** the CA (the CA ends May 2036). The original SAN listed a GCP internal IP (`10.128.0.2`); it is no longer needed. Verify the name check exactly as Graylog will do it (Section 6, check 3).

7. **Graylog truststore** (a copy of Java's default store plus your CA; lives in the `graylog-data` volume, so redo it on every fresh deploy). Start the stack once so the volume exists, then:

```bash
docker cp $PROJ/wazuh-certificates/root-ca.pem copilot-graylog-1:/tmp/wazuh-root-ca.pem
docker exec -u root copilot-graylog-1 sh -c '
cp /opt/java/openjdk/lib/security/cacerts /usr/share/graylog/data/config/cacerts-wazuh &&
keytool -importcert -noprompt -alias wazuh-root-ca -file /tmp/wazuh-root-ca.pem \
  -keystore /usr/share/graylog/data/config/cacerts-wazuh -storepass changeit &&
chown graylog:graylog /usr/share/graylog/data/config/cacerts-wazuh &&
chmod 644 /usr/share/graylog/data/config/cacerts-wazuh'
docker compose up -d --force-recreate --no-deps graylog
```

Compare the CA fingerprint printed by `openssl x509 -in root-ca.pem -noout -fingerprint -sha256` with `keytool -list ... -alias wazuh-root-ca`: they must match.

8. **Validate and start the stack** `[v1]` (`docker compose config --quiet && docker compose up -d`). **Restore the Filebeat patch** (after the manager exists, and again after every recreate of `wazuh-manager`):

```bash
docker cp $PROJ/config/filebeat.yml copilot-wazuh-manager-1:/etc/filebeat/filebeat.yml
docker exec copilot-wazuh-manager-1 filebeat test output      # expect: talk to server... OK
docker restart copilot-wazuh-manager-1                        # restart keeps the file; a recreate wipes it
```

Then **Wazuh Indexer security init** `[v1]` if the indexer returns 503. **Fix CoPilot connector null URLs** `[v1]` (twice, as in v1: before and after seeding). **Fix IRIS database** `[v1]`.

9. **IRIS certificate** (it is not in the repo). The compose file mounts `./iris-web-official/certificates/web_certificates/` at `/www/certs`:

```bash
mkdir -p iris-web-official/certificates/web_certificates
SERVER_IP=YOUR_LINODE_IP
openssl req -x509 -nodes -newkey rsa:4096 -days 365 \
  -keyout iris-web-official/certificates/web_certificates/iris.key \
  -out iris-web-official/certificates/web_certificates/iris.crt \
  -subj "/CN=iris" -addext "subjectAltName=IP:$SERVER_IP,DNS:iris"
chmod 644 iris-web-official/certificates/web_certificates/iris.crt iris-web-official/certificates/web_certificates/iris.key
docker compose up -d iris-nginx
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:4433     # expect 302
```

IRIS admin password: `docker logs copilot-iris-app-1 2>&1 | grep -i administrator | tail -3`.

10. **Passwords.** CoPilot admin (login username is `admin`, not the email). Never run `docker exec -it` inside `$(...)`: the prompt text gets captured into the hash (a bcrypt hash must be exactly 60 characters):

```bash
read -s -p "New CoPilot admin password: " NEWPW; echo
HASH=$(docker exec -e PW="$NEWPW" copilot-copilot-backend-1 python3 -c "import bcrypt,os; print(bcrypt.hashpw(os.environ['PW'].encode(), bcrypt.gensalt(12)).decode())")
unset NEWPW; echo -n "$HASH" | wc -c                           # must print 60
docker exec copilot-copilot-mysql-1 mysql -u root -p$(grep MYSQL_ROOT_PASSWORD .env | cut -d= -f2) copilot \
  -e "UPDATE user SET password='$HASH' WHERE username='admin'; SELECT username, LENGTH(password) FROM user WHERE username='admin';"
```

```
Grafana and InfluxDB read their init passwords only on first boot; afterwards use their CLIs:
```

```bash
docker exec copilot-grafana-1 grafana cli admin reset-admin-password "<new password>"
docker exec -e PW="<new password>" copilot-influxdb-1 sh -c 'influx user password --name admin --password "$PW"'
```

```
Graylog: edit `GRAYLOG_PASSWORD`, `GRAYLOG_NETWORK_PASSWORD` and `GRAYLOG_ROOT_PASSWORD_SHA2` (SHA-256 of the password) in `.env` together, never `GRAYLOG_PASSWORD_SECRET`; then `docker compose up -d --force-recreate graylog` (a plain `restart` does not re-read `.env`). Choose passwords without `$`, `#`, quotes, backslashes or spaces.

Decision recorded: the Wazuh indexer `admin` and `wazuh-wui` default passwords were deliberately left unchanged (lab; their ports are not exposed). Revisit before any production use.
```

11\. **Fluent Bit config.** The repo copy is `config/fluent-bit.conf`:

```
[SERVICE]
    Flush         1
    Log_Level     info
    Parsers_File  parsers.conf

[INPUT]
    Name              tail
    Path              /var/ossec/logs/alerts/alerts.json
    Tag               wazuh.alerts
    Parser            json
    Read_from_Head    On
    Refresh_Interval  5

[OUTPUT]
    Name           tcp
    Match          wazuh.*
    Host           graylog
    Port           5555
    Format         json_lines
    json_date_key  false
```

```
Validate: `docker exec copilot-fluent-bit-1 /fluent-bit/bin/fluent-bit -c /fluent-bit/etc/fluent-bit.conf --dry-run` (expect "configuration test is successful"), then `docker compose restart fluent-bit`. Set `Read_from_Head Off` once the flow is confirmed, otherwise every restart re-sends the whole alert file.
```

12\. **Customer and groups.** Reconnect connectors `[v1]` (Wazuh-Indexer, Wazuh-Manager, Graylog, Grafana, InfluxDB). For InfluxDB: URL `http://influxdb:8086`, extra data `socplatform,copilot` (`org,bucket`), API token created in the InfluxDB UI (custom token, read/write on the `copilot` bucket). **Full rebuild:** re-provision the customer in CoPilot (Customers, Provision): it recreates the Graylog index, stream (rule `agent_labels_customer` equals the customer code), pipeline link, Wazuh groups and the Grafana org with dashboards (per v1; not re-tested on the new build). **Manager-only recreate:** the groups are lost but the Graylog and Grafana side survives; restore them:

```bash
for g in Linux_sirhiira Mac_sirhiira Windows_sirhiira; do
  docker exec copilot-wazuh-manager-1 /var/ossec/bin/agent_groups -a -g $g -q
  docker cp $PROJ/config/wazuh-groups/$g/agent.conf copilot-wazuh-manager-1:/var/ossec/etc/shared/$g/agent.conf
  docker exec copilot-wazuh-manager-1 chown wazuh:wazuh /var/ossec/etc/shared/$g/agent.conf
  docker exec copilot-wazuh-manager-1 chmod 660 /var/ossec/etc/shared/$g/agent.conf
  docker exec copilot-wazuh-manager-1 /var/ossec/bin/verify-agent-conf -f /var/ossec/etc/shared/$g/agent.conf
done
docker exec copilot-wazuh-manager-1 /var/ossec/bin/agent_groups -l
```

```
Group naming in this build uses underscores and `Mac_`, not the docs' `Linux-<code>` / `macOS-<code>`. Do not copy `merged.mg` (the manager rebuilds it). The Mac group logs a harmless warning about a missing macOS CIS policy file.
```

### 6. Verification checklist

1. All containers `Up`; Graylog `healthy`: `docker compose ps`.
2. Filebeat: `docker exec copilot-wazuh-manager-1 filebeat test output` ends in `talk to server... OK` (the "certificate chain verification is disabled" warning is expected while the patch is used).
3. Name and CA check, exactly what Graylog does, no verification disabled:

```bash
read -s -p "Indexer admin password: " IPW; echo
curl -s --cacert $PROJ/wazuh-certificates/root-ca.pem --resolve wazuh-indexer:9200:127.0.0.1 \
  -u "admin:$IPW" "https://wazuh-indexer:9200/_cluster/health?pretty"; unset IPW        # status: green
```

4. Graylog log shows `Connected to (Elastic/Open)Search version <Elasticsearch:7.10.2>` and no `PKIX` / `Unauthorized` (the indexer introduces itself as 7.10.2 on purpose: `compatibility.override_main_response_version: true`; do **not** change it, Filebeat depends on it). Graylog 6.1 with this indexer (OpenSearch 2.19.5) is not officially supported but works; known community-reported risk: range aggregations ignoring filters.
5. Customer index lives in the Wazuh Indexer: `curl -sk -u admin "https://localhost:9200/_cat/indices/wazuh-sirhiira*?v"` shows `wazuh-sirhiira_0` (and Graylog's `graylog_0`, `gl-events_0`, `gl-system-events_0`).
6. Synthetic end-to-end test (bypasses Fluent Bit; the event carries the customer label). Expected: one document in `wazuh-sirhiira_0` and one in `graylog_0`:

```bash
cd $PROJ; TS=$(date -u +%Y-%m-%dT%H:%M:%S.000+0000)
printf '{"timestamp":"%s","rule":{"level":3,"description":"SOCHub synthetic test event","id":"99999","groups":["test"]},"agent":{"id":"999","name":"synthetic-test","labels":{"customer":"sirhiira"}},"manager":{"name":"wazuh-manager"},"id":"synthetic.%s","full_log":"synthetic test event, safe to delete","decoder":{"name":"synthetic"},"location":"manual-test"}\n' "$TS" "$(date +%s)" > /dev/tcp/127.0.0.1/5555 && echo sent
sleep 20; P=$(grep -m1 '^GRAYLOG_INDEXER_PASSWORD=' .env | cut -d= -f2-)
curl -sk -u "admin:$P" "https://localhost:9200/wazuh-sirhiira*,graylog_*/_search?q=agent_name:synthetic-test&size=2&filter_path=hits.total,hits.hits._index&pretty"; unset P
```

7. CoPilot: Platform, Log Management, Index Management, filtered to the customer, lists `wazuh-sirhiira_0`. Grafana (customer org), `EDR - _SUMMARY`: last hour shows the event (Agents 1, Events 1).

### 7. Known discrepancies and status

| #  | Item                                                                                                                                                                                                                                      | Status                                                                                                  |
| -- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| 1  | Firewall exposure and wrong ports                                                                                                                                                                                                         | Solved                                                                                                  |
| 2  | IRIS cert missing                                                                                                                                                                                                                         | Solved (step 9)                                                                                         |
| 3  | Default passwords                                                                                                                                                                                                                         | Solved for CoPilot, Grafana, Graylog, InfluxDB. Wazuh indexer and `wazuh-wui` left default by choice    |
| 4  | Grafana OpenSearch plugin                                                                                                                                                                                                                 | Solved                                                                                                  |
| 5  | Empty `fluent-bit.conf`                                                                                                                                                                                                                   | Solved                                                                                                  |
| 6  | Manager logs mount path                                                                                                                                                                                                                   | Solved (logs only)                                                                                      |
| 7  | Filebeat patch lost on recreate                                                                                                                                                                                                           | Solved by restoring from the repo copy                                                                  |
| 8  | Wazuh groups lost on recreate                                                                                                                                                                                                             | Solved (step 12)                                                                                        |
| 9  | Two OpenSearch stores                                                                                                                                                                                                                     | Solved: Graylog writes to the Wazuh Indexer. `opensearch-graylog` is still running and is to be retired |
| 10 | Other manager volume paths                                                                                                                                                                                                                | Open (Section 9)                                                                                        |
| 11 | Backup repo stale                                                                                                                                                                                                                         | Fix by committing the files in Section 2                                                                |
| 12 | Misc: InfluxDB and Grafana connectors use the public IP, 8086 firewall rule, `Read_from_Head On`, an InfluxDB token that appeared in a screenshot, Event Shipper placeholder `graylog_host`, IRIS connector not present in CoPilot's list | Parked                                                                                                  |
| 13 | Graylog lookup tables reference missing files (`network_ports.csv`, `software_vendors.csv`, `nist_800_53_to_cui.csv` in `/etc/graylog/`)                                                                                                  | Open. Likely effect: missing enrichment fields                                                          |
| 14 | Graylog heap cap not applied                                                                                                                                                                                                              | Open                                                                                                    |
| 15 | CoPilot Deploy Agent wizard cannot generate commands (Section 8)                                                                                                                                                                          | Open                                                                                                    |
| 16 | Each event stored twice: customer index and `graylog_0` (two streams)                                                                                                                                                                     | Noted, not changed. Not known whether this is by design                                                 |

### 8. Agent enrolment: Deploy Agent wizard (unresolved)

Wizard error for the customer: missing `repo_url`, `repo_username`, `repo_password`, `windows_edr_installer`, `wazuh_domain`, `customer_meta_wazuh_registration_port`, `customer_meta_wazuh_log_ingestion_port`, `customer_meta_wazuh_auth_password`. Five map to the Customers, Default Settings dialog (EDR Repo URL, EDR Repo Username, EDR Repo Password, Windows EDR Installer, Wazuh Domain). The three `customer_meta_*` values live in the `customers_meta` table and are populated by the Provision request (`provision.py`, around lines 177-182). The public SOCFortress docs do not describe these fields, and the repo's `backend/app` code does not contain the error text (it may be in the frontend, or the running image is newer than the repo). Provisioning also checks ports for collisions between customers, which suggests a per-customer Wazuh worker design that a single-manager lab does not have. The schema example lists registration `1514` and log ingestion `1515`, which looks reversed from Wazuh's real ports (`1515` enrolment, `1514` events): do not copy it without checking.

Next, read-only: find the error text in the frontend (`grep -rIl "Cannot generate EDR" . --exclude-dir=node_modules --exclude-dir=.git`), compare `git log -1` with the image build date, and inspect the stored values (`DESCRIBE customers_meta; SELECT ... FROM customers_meta;`, printing the password length only).

Fallback that works with Wazuh's documented variables (also what Group Policy or Intune would use), with an administrator PowerShell and an agent version not newer than the manager (4.14.5):

```
msiexec.exe /i wazuh-agent-4.14.5-1.msi /q WAZUH_MANAGER="YOUR_LINODE_IP" WAZUH_AGENT_GROUP="Windows_sirhiira" WAZUH_AGENT_NAME="<hostname>"
```

Test connectivity first from the endpoint: `Test-NetConnection YOUR_LINODE_IP -Port 1515` and `-Port 1514` (both must be `True`; verified from the author's Windows machine). Windows group config enrolled endpoints get: file-integrity monitoring of system folders and `c:\users\*` in real time, many registry keys, and the Application, Security, System, PowerShell and Sysmon event logs. Check on the server: `docker exec copilot-wazuh-manager-1 /var/ossec/bin/agent_control -l`.

Production-minded to-do: enrolment password on the manager, a DNS name for the Wazuh domain, an HTTPS installer repo with credentials if the wizard needs one, restricted source networks.

### 9. Not done yet (planned)

* **Stage 5 cleanup:** stop and remove `opensearch-graylog` once agent data is verified; remove the Filebeat `verification_mode: none` patch and verify against the CA (the new certificate allows it; not tested); `Read_from_Head Off`; fix the Graylog heap flag; handle the missing lookup CSV files.
* **Phase 2 persistence:** the manager's `etc`, `queue`, `multigroups`, `api/configuration` and `active-response` volumes use `/var/wazuh-manager/...` paths, so that data lives in the container, not in volumes. Check the vendor layout before changing. Caution: mounting a non-empty volume over `/var/ossec/etc` can stop Docker seeding the image's default config, so use fresh volumes and keep the old ones until the manager is healthy.
* **Optional:** upgrade Graylog to 6.2 or later (a Graylog staff member said 6.2+ aims to support OpenSearch 2.19.3+); Graylog itself reports 7.1.9 as current. Replace the Indexer `admin` account in Graylog's store address with a dedicated limited user.

### 10. Lessons learned

* The backup held config only, and the compose file did not match the real images (mount paths, plugin). Restores fail quietly where a container works from its own filesystem and only breaks when recreated.
* `docker exec -it` inside `$(...)` pollutes captured output; use `-e VAR=...` without `-it`. Never hide database errors with `2>/dev/null` on a write.
* Grafana and InfluxDB read their init passwords only at first boot: use their CLIs afterwards.
* `docker compose restart` does not re-read `.env`; use `up -d --force-recreate <service>`.
* A single-file bind mount follows the inode: overwrite with `cat new > old`, never replace the file.
* A volume's contents are only seeded from the image when the volume is empty.
* Hostnames in TLS certificates matter: `wazuh-indexer` must be in the SAN list for any verifying client (Graylog's Java does verify).
* Change one thing at a time; keep a rollback copy (`~/docker-compose.yml.bak*`, `~/cert-backup`, `~/manager-backup`).

### 11. Prompt to start the next chat

> I am rebuilding my self-hosted SOCFortress CoPilot lab (Wazuh 4.14.5, Graylog 6.1, DFIR-IRIS 2.4.19, Grafana, InfluxDB, Fluent Bit; 17 containers on one Ubuntu 24.04 VM on Linode). I have the v2 recovery guide (pasted below). Work step by step: give one step at a time, explain what each command does and why, wait for my output before continuing, and refer to docs.socfortress.co and the Wazuh docs when unsure. Unresolved: the CoPilot Deploy Agent wizard (needs EDR repo and customer meta values), Stage 5 cleanup, and manager volume persistence. Project folder is `/home/root/soc-platform/CoPilot`. \[paste this guide]
