---
description: >-
  This guide documents the backup of the SOC Platform configuration to a private
  GitHub repository and the steps to recover from it on a new VM.
---

# SOC Fort Backup & Recovery

#### What Was Backed Up

The backup lives on branch `soc-platform-build` of the private repo:\
`https://github.com/SirHiira/soc-fort-copilot`

| File/Directory           | Purpose                                   |
| ------------------------ | ----------------------------------------- |
| `docker-compose.yml`     | Full 17-service stack definition          |
| `.env.gpg`               | All secrets and passwords (GPG encrypted) |
| `wazuh-certificates/`    | Wazuh SSL certificates (9 files)          |
| `config/filebeat.yml`    | Wazuh Filebeat patched config             |
| `config/fluent-bit.conf` | Fluent Bit forwarder config               |

> **NOTE:** Historical data (Wazuh alerts, Graylog logs, CoPilot records) is NOT included. The platform rebuilds fresh with empty databases.

#### Backup Procedure

**Prerequisites**

* GitHub account: SirHiira
* Private repo: soc-fort-copilot
* GitHub Personal Access Token with `repo` scope

**Steps**

**1. Encrypt the .env file**

bash

```bash
cd /home/sir.hiira/soc-platform/CoPilot
gpg --symmetric --cipher-algo AES256 --output .env.gpg .env
```

Store the GPG passphrase in a password manager — it cannot be recovered if lost.

**2. Stage and commit**

bash

```bash
git add .env.gpg
git add docker-compose.yml
git add wazuh-certificates/
git add config/filebeat.yml
git add config/fluent-bit.conf
git add .gitignore
git commit -m "SOC Platform backup $(date +%Y-%m-%d)"
```

**3. Push to private repo**

bash

```bash
# Set remote with token
git remote set-url origin https://SirHiira:YOUR_TOKEN@github.com/SirHiira/soc-fort-copilot.git

# Push
git push origin soc-platform-build

# Clean up token
git remote set-url origin https://github.com/SirHiira/soc-fort-copilot.git
```

**Updating the Backup**

Run this whenever you make significant config changes:

bash

```bash
cd /home/sir.hiira/soc-platform/CoPilot
gpg --symmetric --cipher-algo AES256 --output .env.gpg .env
git add -A
git commit -m "Config update $(date +%Y-%m-%d)"
git push origin soc-platform-build
```

***

#### Recovery Procedure

Estimated time: **60-90 minutes**

**Step 1 — Provision a New VM**

| Setting      | Value                             |
| ------------ | --------------------------------- |
| Provider     | GCP or any cloud/on-prem          |
| Machine type | e2-standard-4 (4 vCPU, 16 GB RAM) |
| OS           | Ubuntu 24.04 LTS                  |
| Boot disk    | 50 GB SSD                         |

**Step 2 — Install Docker**

bash

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
sudo apt-get install -y git gnupg

# Verify
docker --version
docker compose version
```

**Step 3 — Clone the Backup Repo**

bash

```bash
mkdir -p /home/$USER/soc-platform
cd /home/$USER/soc-platform

# Clone (enter GitHub token when prompted for password)
git clone -b soc-platform-build \
  https://github.com/SirHiira/soc-fort-copilot.git CoPilot

cd CoPilot
```

**Step 4 — Decrypt the .env File**

bash

```bash
# Enter your GPG passphrase when prompted
gpg --decrypt .env.gpg > .env

# Verify
head -3 .env
```

**Step 5 — Update the Server IP**

bash

<pre class="language-bash"><code class="lang-bash"><strong>NEW_IP=$(curl -s ifconfig.me)
</strong>echo "New IP: $NEW_IP"

sed -i "s/SERVER_HOST=.*/SERVER_HOST=$NEW_IP/" .env

grep SERVER_HOST .env
</code></pre>

**Step 6 — Create Required Directories**

bash

```bash
mkdir -p data/copilot-backend-data/logs
mkdir -p data/data/minio-data
mkdir -p config/wazuh-dashboard
```

**Step 7 — Validate and Start the Stack**

bash

```bash
# Validate compose file
docker compose config --quiet && echo "✅ Valid" || echo "❌ Errors"

# Pull images (5-10 minutes)
docker compose pull

# Start everything
docker compose up -d

# Wait for initialisation
sleep 120

# Check status
docker compose ps
```

All containers should show `Up`. Expected restarts on first boot: `iris-app` (database migration) and `copilot-backend` (connector seeding).

**Step 8 — Initialise Wazuh Indexer Security**

bash

```bash
docker exec -it copilot-wazuh-indexer-1 bash -c "
  export JAVA_HOME=/usr/share/wazuh-indexer/jdk && \
  /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
    -cd /usr/share/wazuh-indexer/config/opensearch-security/ \
    -p 9200 -cn wazuh-indexer -h wazuh-indexer -nhnv \
    -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem \
    -cert /usr/share/wazuh-indexer/config/certs/admin.pem \
    -key /usr/share/wazuh-indexer/config/certs/admin-key.pem
"
```

Expected output: `Done with success`

**Step 9 — Fix CoPilot Connector Null URL**

bash

```bash
docker exec copilot-copilot-mysql-1 mysql \
  -u root \
  -p$(grep MYSQL_ROOT_PASSWORD .env | cut -d= -f2) \
  copilot \
  -e "ALTER TABLE connectors MODIFY connector_url varchar(256) NULL;" 2>/dev/null
```

**Step 10 — Fix IRIS Database**

bash

```bash
sleep 30

docker exec copilot-iris-db-1 psql -U iris -d iris_db -c "
  CREATE EXTENSION IF NOT EXISTS pgcrypto;
  CREATE USER iris_admin WITH PASSWORD 'iris_admin_password';
  ALTER USER iris_admin WITH SUPERUSER;
" 2>/dev/null

# Restart IRIS app to trigger migration
docker compose up -d --force-recreate iris-app iris-worker
sleep 60

# Confirm IRIS started
docker logs copilot-iris-app-1 2>&1 | grep -i "IRIS IS READY"
```

**Step 11 — Apply Filebeat Patch**

bash

```bash
docker exec copilot-wazuh-manager-1 bash -c "
  cp /etc/filebeat/filebeat.yml /tmp/filebeat.yml &&
  sed -i 's|hosts:.*|hosts: [\"https://wazuh-indexer:9200\"]|' /tmp/filebeat.yml &&
  sed -i 's/#username:/username: \"admin\"/' /tmp/filebeat.yml &&
  sed -i 's/#password:/password: \"admin\"/' /tmp/filebeat.yml &&
  sed -i 's/#ssl.verification_mode:/ssl.verification_mode: none/' /tmp/filebeat.yml &&
  cp /tmp/filebeat.yml /etc/filebeat/filebeat.yml"

docker exec copilot-wazuh-manager-1 bash -c \
  "pkill -f filebeat; sleep 2; nohup filebeat -e > /var/log/filebeat.log 2>&1 &"

# Verify
docker exec copilot-wazuh-manager-1 filebeat test output 2>&1 | tail -5
```

Expected: `talk to server... OK`

**Step 12 — Fix Null Connector URLs (post-seeding)**

bash

```bash
docker exec copilot-copilot-mysql-1 mysql \
  -u root \
  -p$(grep MYSQL_ROOT_PASSWORD .env | cut -d= -f2) \
  copilot \
  -e "UPDATE connectors SET connector_url='http://localhost' WHERE connector_url IS NULL;" 2>/dev/null
```

**Step 13 — Open GCP Firewall Ports**

bash

```bash
gcloud compute firewall-rules create soc-platform-ports \
  --allow tcp:80,tcp:443,tcp:8443,tcp:9001,tcp:4433,tcp:3000,tcp:8086 \
  --source-ranges 0.0.0.0/0 \
  --description "SOC Platform UI ports"
```

**Step 14 — Verify All Services**

bash

```bash
curl -sk -o /dev/null -w "%{http_code}" https://localhost:443 && echo " CoPilot"
curl -sk -o /dev/null -w "%{http_code}" -u admin:admin https://localhost:9200 && echo " Wazuh Indexer"
curl -sk -o /dev/null -w "%{http_code}" https://localhost:8443 && echo " Wazuh Dashboard"
curl -sk -o /dev/null -w "%{http_code}" http://localhost:9001 && echo " Graylog"
curl -sk -o /dev/null -w "%{http_code}" https://localhost:4433 && echo " IRIS"
curl -sk -o /dev/null -w "%{http_code}" http://localhost:3000 && echo " Grafana"
```

All should return `200` or `302`.

**Step 15 — Reconnect CoPilot Connectors**

Log into CoPilot and go to **Connectors**. Update and verify:

| Connector     | URL                           | Username  | Password/Token                      |
| ------------- | ----------------------------- | --------- | ----------------------------------- |
| Wazuh-Indexer | `https://wazuh-indexer:9200`  | admin     | admin                               |
| Wazuh-Manager | `https://wazuh-manager:55000` | wazuh-wui | `MyS3cr37P450r.*-`                  |
| Graylog       | `http://graylog:9001`         | admin     | GRAYLOG\_PASSWORD from .env         |
| Grafana       | `http://grafana:3000`         | admin     | GrafanaAdmin1234!                   |
| InfluxDB      | `http://influxdb:8086`        | —         | fresh token + `socplatform,copilot` |

**Get fresh InfluxDB token:**

bash

```bash
docker exec copilot-influxdb-1 influx auth list \
  --user admin --hide-headers 2>/dev/null | awk '{print $4}'
```

**Step 16 — Re-provision Customers and Agents**

Since the database is fresh:

1. Go to **Customers → Add Customer** and recreate your customers
2. Go through the **Provision** workflow for each customer
3. Re-enrol agents via **Agents → Deploy Agent**
4. Move agents into correct Wazuh groups

***

#### Service Access URLs

Replace `YOUR_IP` with the VM's external IP.

| Service         | URL                    | Username      | Default Password         |
| --------------- | ---------------------- | ------------- | ------------------------ |
| CoPilot         | `https://YOUR_IP`      | admin         | reset via DB (see below) |
| Wazuh Dashboard | `https://YOUR_IP:8443` | admin         | admin                    |
| Graylog         | `http://YOUR_IP:9001`  | admin         | from .env                |
| DFIR-IRIS       | `https://YOUR_IP:4433` | administrator | from iris-app logs       |
| Grafana         | `http://YOUR_IP:3000`  | admin         | GrafanaAdmin1234!        |
| InfluxDB        | `http://YOUR_IP:8086`  | admin         | InfluxAdmin1234!         |

**Reset CoPilot admin password after rebuild:**

bash

```bash
HASH=$(docker exec copilot-copilot-backend-1 python3 -c \
  "import bcrypt; print(bcrypt.hashpw(b'YourNewPassword!', bcrypt.gensalt(12)).decode())")

#Alternative method. Note this command requires you to type the password in a hidden prompt
HASH=$(docker exec -it copilot-copilot-backend-1 python3 -c \
"import bcrypt,getpass; print(bcrypt.hashpw(getpass.getpass().encode(), bcrypt.gensalt(12)).decode())")


docker exec copilot-copilot-mysql-1 mysql \
  -u root \
  -p$(grep MYSQL_ROOT_PASSWORD .env | cut -d= -f2) \
  copilot \
  -e "UPDATE user SET password='$HASH' WHERE username='admin';" 2>/dev/null
```

**Get IRIS admin password:**

bash

```bash
docker logs copilot-iris-app-1 2>&1 | grep -i "administrator" | tail -3
```

***

#### Important Notes

* The Filebeat patch (Step 11) must be reapplied every time the `wazuh-manager` container is recreated
* Customer Wazuh groups (`Linux_hiirasir` etc.) are recreated automatically during customer provisioning
* The InfluxDB API token changes on every fresh deployment — always retrieve it fresh
* If Wazuh Dashboard shows errors after rebuild, re-run Step 8 (securityadmin)
* Keep your GPG passphrase safe — without it the `.env.gpg` backup is useless
