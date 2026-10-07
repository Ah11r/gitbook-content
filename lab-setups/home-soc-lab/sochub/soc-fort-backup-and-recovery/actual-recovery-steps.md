# Actual Recovery Steps

## SOC Fort Recovery on Linode

Recovery of the SOCFortress CoPilot lab stack (originally on GCP) onto a Linode server, from the private GitHub backup. This note records what we did, the commands used, the outputs you pasted, and the fixes.

### 1. Context

* GCP free credits ended, so the environment was backed up to a private GitHub repo (`soc-fort-copilot`, branch `soc-platform-build`) and restored on Linode.
* Stack: 17 services on one Ubuntu 24.04 VM (CoPilot, Wazuh 4.14.5, Graylog 6.1, DFIR-IRIS v2.4.19, Grafana, InfluxDB, Fluent Bit), all on the `soc-network` Docker bridge.
* The backup holds **config only**: `docker-compose.yml`, `.env.gpg`, `wazuh-certificates/`, `config/filebeat.yml`, `config/fluent-bit.conf`. **No historical data** (Wazuh alerts, Graylog logs, CoPilot records, IRIS cases). The rebuild starts with empty databases.
* The recovery guide's 16 steps were followed on Linode. Everything below is what came up _after_ that.

### 2. Concerns raised at the start

1. Repo must be **private**, and must not contain plaintext `.env` or keys. (`.env` is GPG-encrypted; the Wazuh CA/admin keys are in the repo, which is fine only while it stays private.)
2. Default credentials (`Admin1234!`, `admin/admin` on the indexer, stock `wazuh-wui`) are dangerous on a public IP.
3. The backup does not include Docker volumes, so old data is gone.
4. `:latest` images (CoPilot, Grafana) may have drifted since the GCP build.
5. Rotate or revoke the GitHub Personal Access Token if no longer needed.

### 3. Linode Cloud Firewall

Review of the first rule set found problems:

| Problem                         | Fix                                                            |
| ------------------------------- | -------------------------------------------------------------- |
| Admin UIs open to all IPv4/IPv6 | Restrict to your own public IP (`/32`)                         |
| 55000 (Wazuh API) exposed       | Not needed externally, CoPilot uses the Docker network. Remove |
| 8086 (InfluxDB) exposed         | Not needed externally. Remove (see section 9)                  |
| 8444 in rule                    | The dashboard publishes on **8443**. Typo                      |
| 1515 missing                    | Needed for agent registration                                  |
| Port 80 open                    | Only redirects to 443. Remove or restrict                      |
| SSH source `105.160.0.0/14`     | Broad. Use a `/32` if the IP is stable                         |

Final layout (lab):

* Admin ports **443, 3000, 4433, 8443, 9001**: source = your own IP.
* Agent ports **1514, 1515** (and 12201 if needed): source = your own network.
* **Outbound:** leave the default Accept with no rules. The Cloud Firewall is stateful, and the server needs outbound for image pulls, apt, feeds and git.
* **Default inbound policy:** Drop.
* Keep the Lish console handy in case a rule locks you out.

Docker publishes some ports on all interfaces that nothing needs externally (MySQL 3306, Wazuh Indexer 9200, CoPilot backend 5000, MinIO 9002 and 5555). The Cloud Firewall drops them, but as a second layer, bind them to `127.0.0.1` in the compose file later. Docker-published ports bypass `ufw`, so rely on the Cloud Firewall.

### 4. SSH freezing and tmux

**Cause:** an idle SSH session is dropped by NAT or the ISP, so the terminal freezes with no error.

Escape a hung session: press `Enter`, then `~` then `.`

Client keepalives, in `~/.ssh/config` on the Ubuntu host:

```
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 4
```

Run long jobs inside tmux on the Linode:

```bash
sudo apt install -y tmux
tmux new -s soc
# after a disconnect:
tmux attach -t soc
```

Optional server side (`/etc/ssh/sshd_config`): `ClientAliveInterval 60`, `ClientAliveCountMax 3`, then `sudo systemctl restart ssh`. Test from a second terminal before closing the first.

**tmux scrolling:** the mouse wheel was sending shell history, not scrolling.

* Enter scroll mode with `Ctrl+b` then `[`. Use PgUp/PgDn, and `q` to quit.
* Enable the mouse and a bigger scrollback:

```bash
echo "set -g mouse on" >> ~/.tmux.conf
echo "set -g history-limit 50000" >> ~/.tmux.conf
tmux source-file ~/.tmux.conf
```

* With the mouse on, hold **Shift** while dragging to copy to the local clipboard (`Ctrl+Shift+C`).
* For long output: `docker logs <container> --tail 20 2>&1 | tee ~/file.log`

### 5. Post-recovery status check

```bash
cd ~/soc-platform/CoPilot
docker compose ps
```

Result: all containers `Up` (Graylog `healthy`) **except** `copilot-iris-nginx-1`, which was `Restarting (1)`.

Notable published ports from the output:

* Wazuh Dashboard `0.0.0.0:8443->5601`
* Graylog 9001, 12201 (tcp/udp), 5044, 5555
* Wazuh Manager 1514-1515, 514/udp, 55000
* Wazuh Indexer 9200, MySQL 3306, CoPilot backend 5000, MinIO 9002

### 6. Fix: IRIS nginx restart loop (missing certificate)

Diagnosis:

```bash
docker logs copilot-iris-nginx-1 --tail 20
```

Output (repeating, once a minute):

```
[emerg] 1#1: cannot load certificate "/www/certs/iris.crt": BIO_new_file() failed
(SSL: error:80000002:system library::No such file or directory:calling fopen(/www/certs/iris.crt, r)
```

The error is `No such file`, not `Permission denied` (which was the Issue 8 error on the GCP build). The cert was never in the backup, and the recovery guide has no step that creates it.

Find where the compose file mounts the certs:

```bash
sed -n '310,345p' docker-compose.yml
find . -maxdepth 4 \( -name "iris.crt" -o -name "iris.key" \) 2>/dev/null
ls -la data iris-web-official
```

Findings:

* Compose mounts `./iris-web-official/certificates/web_certificates/:/www/certs/:ro`
* `CERT_FILENAME=iris.crt`, `KEY_FILENAME=iris.key`
* `find` returned nothing. The files do not exist on this server.
* The folder `iris-web-official/certificates/web_certificates/` existed but was empty.

Fix: generate a new self-signed cert (safe, since it is only self-signed):

```bash
ls -laR iris-web-official/certificates

SERVER_IP=YOUR_LINODE_IP
openssl req -x509 -nodes -newkey rsa:4096 -days 365 \
  -keyout iris-web-official/certificates/web_certificates/iris.key \
  -out iris-web-official/certificates/web_certificates/iris.crt \
  -subj "/CN=iris" \
  -addext "subjectAltName=IP:$SERVER_IP,DNS:iris"

chmod 644 iris-web-official/certificates/web_certificates/iris.crt \
          iris-web-official/certificates/web_certificates/iris.key

docker compose up -d iris-nginx
sleep 10
docker compose ps iris-nginx
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:4433
```

Result: `Up 22 seconds (healthy)`, `curl` returned `302` (login redirect).

Notes:

* The key is set to `644` because nginx runs as a non-root user (same lesson as the original Issue 8).
* The browser shows a self-signed certificate warning. That is normal.
* **Add this step to the recovery guide.** Do not commit the IRIS key or cert to the repo. They are cheap to regenerate.
* IRIS admin password: `docker logs copilot-iris-app-1 2>&1 | grep -i "administrator" | tail -3`. You retrieved it and logged in successfully.

### 7. CoPilot admin password reset

Your first attempt used `docker exec -it ... getpass` inside `$(...)`, and the login failed.

Diagnosis:

```bash
docker exec copilot-copilot-mysql-1 mysql \
  -u root -p$(grep MYSQL_ROOT_PASSWORD .env | cut -d= -f2) copilot \
  -e "SELECT username, LENGTH(password) AS len FROM user WHERE username='admin';" 2>/dev/null
```

Output: `admin 73`. A valid bcrypt hash is exactly **60** characters. With `-it`, the `Password:` prompt text was captured into `$HASH` along with the hash, so the stored value was not a valid hash.

Fix (no `-it`, password passed as an environment variable):

```bash
read -s -p "New CoPilot admin password: " NEWPW; echo
HASH=$(docker exec -e PW="$NEWPW" copilot-copilot-backend-1 python3 -c "import bcrypt,os; print(bcrypt.hashpw(os.environ['PW'].encode(), bcrypt.gensalt(12)).decode())")
unset NEWPW
echo -n "$HASH" | wc -c        # must print 60

docker exec copilot-copilot-mysql-1 mysql \
  -u root -p$(grep MYSQL_ROOT_PASSWORD .env | cut -d= -f2) copilot \
  -e "UPDATE user SET password='$HASH' WHERE username='admin'; SELECT username, LENGTH(password) AS len FROM user WHERE username='admin';"
```

Result: `len` = 60, login worked. **Login username is `admin`, not the email.** No restart is needed, because CoPilot checks the DB at each login. Do not use `2>/dev/null` on the `UPDATE`, since it hides errors.

### 8. Grafana admin password

Grafana reads its init password only on first boot, so change it with the CLI (no restart):

```bash
read -s -p "New Grafana admin password: " GPW; echo
docker exec copilot-grafana-1 grafana cli admin reset-admin-password "$GPW"
unset GPW
```

Then log in at `http://YOUR_LINODE_IP:3000` and update the CoPilot **Grafana** connector with the new password. You verified this successfully.

### 9. Graylog admin password

Findings from `.env` (values hidden):

* `GRAYLOG_ROOT_PASSWORD_SHA2` is the admin login hash. It reaches the container via the compose file.
* `GRAYLOG_PASSWORD` and `GRAYLOG_NETWORK_PASSWORD` are the plain-text passwords used by CoPilot.
* `GRAYLOG_PASSWORD_SECRET` is an encryption/salt secret, **not a login password. Never change it.**

Checks (no secrets printed):

```bash
cp .env ~/env.bak && chmod 600 ~/env.bak
get() { grep -m1 "^$1=" .env | cut -d= -f2-; }
[ "$(get GRAYLOG_PASSWORD)" = "$(get GRAYLOG_NETWORK_PASSWORD)" ] && echo "PASSWORD == NETWORK_PASSWORD" || echo "PASSWORD differs from NETWORK_PASSWORD"
[ "$(printf %s "$(get GRAYLOG_PASSWORD)" | sha256sum | cut -d' ' -f1)" = "$(get GRAYLOG_ROOT_PASSWORD_SHA2)" ] && echo "SHA2 matches PASSWORD" || echo "SHA2 does not match PASSWORD"
```

Result: `PASSWORD == NETWORK_PASSWORD` and `SHA2 matches PASSWORD`. All three values change together.

Update `.env` with Python (special characters can't break the edit):

```bash
read -s -p "New Graylog admin password: " GLPW; echo
export GLPW
python3 - <<'EOF'
import os, hashlib
pw = os.environ["GLPW"]
sha = hashlib.sha256(pw.encode()).hexdigest()
new = {"GRAYLOG_PASSWORD": pw, "GRAYLOG_NETWORK_PASSWORD": pw, "GRAYLOG_ROOT_PASSWORD_SHA2": sha}
lines = open(".env").read().split("\n")
out = []
for l in lines:
    k = l.split("=", 1)[0]
    out.append(f"{k}={new[k]}" if k in new else l)
open(".env", "w").write("\n".join(out))
EOF
unset GLPW
grep -c -E "^GRAYLOG_(PASSWORD|NETWORK_PASSWORD|ROOT_PASSWORD_SHA2)=" .env    # must print 3
```

Avoid `$`, `#`, quotes, backslashes and spaces in the password (Docker Compose interpolates them).

Recreate the container (a plain `restart` does not re-read `.env`):

```bash
docker compose up -d --force-recreate graylog
sleep 60
docker compose ps graylog
curl -s -o /dev/null -w "%{http_code}\n" -u admin http://localhost:9001/api/system    # expect 200
```

Then update the CoPilot **Graylog** connector. You confirmed all of this worked.

### 10. InfluxDB

Reset the admin password with the CLI:

```bash
read -s -p "New InfluxDB admin password: " IPW; echo
docker exec -e PW="$IPW" copilot-influxdb-1 sh -c 'influx user password --name admin --password "$PW"'
unset IPW
```

(Minimum 8 characters.) The CoPilot connector uses an **API token**, not the password.

Create the token in the InfluxDB UI (`http://YOUR_LINODE_IP:8086`): **Load Data → API Tokens → Generate API Token → Custom**, with Read + Write on the `copilot` bucket. Save it in your password manager.

CoPilot **InfluxDB connector** values:

| Field         | Value                                                       |
| ------------- | ----------------------------------------------------------- |
| Connector URL | `http://influxdb:8086` (Docker hostname, not the public IP) |
| API Key       | the token you generated                                     |
| Extra data    | `socplatform,copilot` (format `org,bucket`)                 |

The org and bucket came from the compose file (`DOCKER_INFLUXDB_INIT_ORG` and `DOCKER_INFLUXDB_INIT_BUCKET`) and the recovery guide. You can confirm them in the UI (profile icon for the org, **Load Data → Buckets** for the bucket). The connector verified.

You had to open 8086 in the firewall to reach the UI. **Remove that rule now** if the connector uses `http://influxdb:8086`.

### 11. Decision: Wazuh default passwords

You chose **not to change any further passwords**. Wazuh indexer `admin` and the API user `wazuh-wui` remain on defaults.

Rationale (lab): 9200 and 55000 are not exposed in the firewall, the dashboard (8443) is restricted to your IP, and the databases are empty. This stops being acceptable if you open those ports, share the network, or add real data.

Why the indexer `admin` password is the risky one: Filebeat in the manager (patched config), the dashboard and the CoPilot connectors likely depend on it, and changing it without updating them would silently break the stack. Do not change it casually.

### 12. Remaining housekeeping

* [ ] Delete the firewall rule for **8086** (InfluxDB connector uses the Docker hostname).
* [ ] Delete the InfluxDB API token that appeared in a screenshot and generate a fresh one for the connector. Avoid sharing screenshots with the key field visible.
* [ ] **Re-encrypt `.env` to `.env.gpg` and push it**, because the backup still holds the old Graylog password.
* [ ] Add the **IRIS certificate generation** step (section 6) to the recovery guide.
* [ ] Do **not** add the new passwords to `docker-compose.yml`. It goes to GitHub. Stale defaults are still present in the Grafana and InfluxDB environment variables (ignored after first init, but misleading).
* [ ] Later hardening: bind 3306, 9200, 5000, 9002 and 5555 to `127.0.0.1` in the compose file.
* [ ] Revoke or rotate the GitHub Personal Access Token if it is no longer needed.
* [ ] Still pending from the guide: re-provision customers and agents (Step 16), and re-apply the Filebeat patch whenever `wazuh-manager` is recreated.

### 13. Lessons learned

* The backup needs three additions: the IRIS cert step, a re-encrypted `.env.gpg` after each password change, and a note that data is not backed up (add volume exports if you want history).
* `docker exec -it` inside `$(...)` can pollute the captured output. Use `-e VAR=...` without `-it`.
* Never hide database errors with `2>/dev/null` on writes.
* Env-var passwords for Grafana and InfluxDB only apply at first init. Use their CLIs afterwards.
* A plain `docker compose restart` does not re-read `.env`. Use `up -d --force-recreate <service>`.
* Check published ports with `docker compose ps` against the firewall rules.
