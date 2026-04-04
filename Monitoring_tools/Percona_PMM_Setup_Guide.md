# Percona PMM Setup Guide
## OS Metrics + PostgreSQL (Patroni) Metrics Monitoring
### Rocky Linux | 3 Client Nodes → 1 Monitoring Server

---

## Server Inventory

| Role | Hostname | IP Address | OS |
|------|----------|------------|----|
| PMM Monitoring Server | pmm-server | `192.168.109.128` | Rocky Linux |
| PostgreSQL Node 1 | pg-node-1 | `192.168.109.133` | Rocky Linux |
| PostgreSQL Node 2 | pg-node-2 | `192.168.109.134` | Rocky Linux |
| PostgreSQL Node 3 | pg-node-3 | `192.168.109.135` | Rocky Linux |

---

## What Will Be Monitored

| Metric Category | Source | Dashboard in PMM |
|----------------|--------|------------------|
| CPU, RAM, Disk, Network | node_exporter (auto by PMM Agent) | OS → Node Summary |
| PostgreSQL performance, connections, locks | PMM PostgreSQL agent | PostgreSQL → Overview |
| Slow queries, query analytics | pg_stat_statements | Query Analytics (QAN) |
| Replication lag, Patroni role (leader/replica) | pg_stat_replication | PostgreSQL → Overview |

---

## Architecture

<img width="1408" height="768" alt="Gemini_Generated_Image_6sc4oe6sc4oe6sc4" src="https://github.com/user-attachments/assets/4411c7b9-50b8-4adb-b113-582e786d382b" />


> PMM Agents on each client node collect OS and PostgreSQL metrics and push them to the PMM Server at `192.168.109.128`. No data is pulled from the server side — agents initiate the connection outbound.

---

## ══════════════════════════════════════════════
## PART 1 — PMM SERVER SETUP
## ▶ Run ONLY on: 192.168.109.128
## ══════════════════════════════════════════════

---

### Step 1.1 — Update System

```bash
# ▶ SERVER: 192.168.109.128

sudo dnf update -y
sudo dnf install -y curl wget vim net-tools
```

---

### Step 1.2 — Install Docker Engine

```bash
# ▶ SERVER: 192.168.109.128

# Remove any old Docker packages
sudo dnf remove -y docker docker-common docker-selinux docker-engine

# Install required plugin
sudo dnf install -y dnf-plugins-core

# Add Docker CE repository for Rocky Linux / CentOS
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker CE
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify
docker --version
sudo systemctl status docker
```

---

### Step 1.3 — Open Firewall Ports on PMM Server

```bash
# ▶ SERVER: 192.168.109.128

sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

---

### Step 1.4 — Set SELinux to Permissive

```bash
# ▶ SERVER: 192.168.109.128

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Verify
getenforce
# Expected: Permissive
```

---

### Step 1.5 — Deploy PMM Server as Docker Container

```bash
# ▶ SERVER: 192.168.109.128

# Create a persistent volume for PMM data (metrics, dashboards, config)
docker volume create pmm-data

# Pull the latest PMM Server 2.x image
docker pull percona/pmm-server:2

# Start the PMM Server container
docker run -d \
  --name pmm-server \
  --restart always \
  -p 80:80 \
  -p 443:443 \
  -v pmm-data:/srv \
  percona/pmm-server:2

# Verify it is running
docker ps

# Watch startup logs (wait until you see "Started" messages)
docker logs pmm-server --tail=40
```

---

### Step 1.6 — Access PMM Web UI

Open a browser and go to:

```
https://192.168.109.128
```

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin` (you will be asked to change this on first login) |

> **Set a strong new password immediately.** You will need it in every `pmm-admin config` command on the client nodes. In this guide it is referred to as `YOUR_PMM_PASSWORD`.

---

### Step 1.7 — Confirm PMM Server is Reachable

```bash
# ▶ SERVER: 192.168.109.128

curl -sk https://192.168.109.128/ping
# Expected response: {"data":{}}
```

---

## ══════════════════════════════════════════════
## PART 2 — PMM CLIENT SETUP
## ▶ Run on ALL THREE: 192.168.109.133 | .134 | .135
## ══════════════════════════════════════════════

> Steps 2.1, 2.2, 2.3, and 2.6 are **identical on all three nodes** — run the same command on each.
> Step 2.4 has a **different command per node** — read carefully.

---

### Step 2.1 — Update System

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

sudo dnf update -y
sudo dnf install -y curl wget vim net-tools
```

---

### Step 2.2 — Add Percona Repository

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

# Install Percona release manager
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm

# Enable PMM2 client repo
sudo percona-release enable pmm2-client

# Confirm repo is listed
sudo dnf repolist | grep pmm
```

---

### Step 2.3 — Install PMM Client

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

sudo dnf install -y pmm2-client

# Verify
pmm-admin --version
```

---

### Step 2.4 — Register Each Node with the PMM Server

> Each node must use its own unique `--node-name` and `--node-address`.
> Run **only the matching block** on each node.

**On pg-node-1 (192.168.109.133):**

```bash
# ▶ SERVER: 192.168.109.133 ONLY

sudo pmm-admin config \
  --server-url=https://admin:admin@192.168.109.128 \
  --server-insecure-tls \
  192.168.109.133 generic pg-node-1
```

**On pg-node-2 (192.168.109.134):**

```bash
# ▶ SERVER: 192.168.109.134 ONLY

sudo pmm-admin config \
  --server-url=https://admin:admin@192.168.109.128 \
  --server-insecure-tls \
  192.168.109.134 generic pg-node-2
```

**On pg-node-3 (192.168.109.135):**

```bash
# ▶ SERVER: 192.168.109.135 ONLY

sudo pmm-admin config \
  --server-url=https://admin:admin@192.168.109.128 \
  --server-insecure-tls \
  192.168.109.135 generic pg-node-3
```

---

### Step 2.5 — Enable and Start PMM Agent

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

sudo systemctl enable pmm-agent
sudo systemctl start pmm-agent
sudo systemctl status pmm-agent

# Confirm it is connected to PMM Server
pmm-admin status
```

Expected output (node name differs per node):

```
Agent ID  : /agent_id/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Node ID   : /node_id/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Node name : pg-node-1          ← unique per node
PMM Server:
  URL     : https://192.168.109.128/
  Version : 2.x.x
Connected : true               ← must be true
```

If `Connected: false`, check firewall and reachability first:

```bash
# ▶ SERVER: 192.168.109.133  (or .134 / .135)

curl -sk https://192.168.109.128/ping
# Must return: {"data":{}}
```

---

### Step 2.6 — Open Required Firewall Ports on Client Nodes

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

# PMM Agent communication ports (agent → server)
sudo firewall-cmd --permanent --add-port=42000-42005/tcp

# PostgreSQL port (PMM agent connects locally)
sudo firewall-cmd --permanent --add-port=5432/tcp

sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

---

### Step 2.7 — Set SELinux to Permissive on Client Nodes

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Verify
getenforce
# Expected: Permissive
```

---

## ══════════════════════════════════════════════
## PART 3 — POSTGRESQL CONFIGURATION FOR PMM
## ▶ Run on ALL THREE: 192.168.109.133 | .134 | .135
## ══════════════════════════════════════════════

---

### Step 3.1 — Create the PMM Monitoring User in PostgreSQL

Run this on the **Patroni primary node only**. The user will replicate to the other two nodes automatically.

First, identify which node is the primary:

```bash
# ▶ SERVER: 192.168.109.133  (or any node)

patronictl -c /etc/patroni/patroni.yml list
# Look for the node showing "Leader" in the Role column
```

Then on the **Leader node**, run:

```bash
# ▶ SERVER: PRIMARY / LEADER NODE ONLY

sudo -u postgres psql
```

Inside the psql prompt:

```sql
-- Create dedicated monitoring user
CREATE USER pmm_user WITH PASSWORD 'PMMStr0ngP@ss!';

-- Grant full monitoring access (PostgreSQL 10+)
GRANT pg_monitor TO pmm_user;

-- Confirm the user exists
\du pmm_user

-- Exit
\q
```

---

### Step 3.2 — Allow PMM User in pg_hba.conf

This must be done on **all three nodes** because Patroni can promote any replica to primary:

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

# Find the exact path of pg_hba.conf on your system
sudo -u postgres psql -c "SHOW hba_file;"
# Common path on Rocky Linux with Patroni: /var/lib/pgsql/data/pg_hba.conf

sudo vi /var/lib/pgsql/data/pg_hba.conf
```

Add these two lines **before** any `reject` or `ident` rules:

```
# PMM monitoring user — localhost only
host    all         pmm_user    127.0.0.1/32    md5
host    all         pmm_user    ::1/128         md5
```

Reload PostgreSQL config on each node (no restart needed):

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

sudo -u postgres psql -c "SELECT pg_reload_conf();"
# Expected: pg_reload_conf
#           ─────────────────
#           t
```

---

### Step 3.3 — Enable Query Statistics in postgresql.conf via patronictl

Use `patronictl edit-config` to safely apply parameters across all nodes. This avoids editing `postgresql.conf` manually on each node:

```bash
# ▶ SERVER: 192.168.109.133  (any node with patronictl access)

patronictl -c /etc/patroni/patroni.yml edit-config
```

In the editor that opens, add or merge the following under `postgresql.parameters`:

```yaml
postgresql:
  parameters:
    track_activities: 'on'
    track_counts: 'on'
    track_io_timing: 'on'
    track_functions: 'all'
    shared_preload_libraries: 'pg_stat_statements'
    pg_stat_statements.track: 'all'
    pg_stat_statements.max: 10000
```

Save and exit. Then apply with a rolling restart (Patroni promotes safely, no downtime):

```bash
# ▶ SERVER: 192.168.109.133

# Reload non-restart parameters immediately
patronictl -c /etc/patroni/patroni.yml reload postgres

# Rolling restart for shared_preload_libraries (one node at a time)
patronictl -c /etc/patroni/patroni.yml restart postgres

```

---

### Step 3.4 — Enable pg_stat_statements Extension

Run on the **primary node only** — replication propagates it to replicas:

```bash
# ▶ SERVER: PRIMARY / LEADER NODE ONLY

sudo -u postgres psql -d postgres \
  -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"

# Verify extension is active
sudo -u postgres psql -d postgres -c "\dx pg_stat_statements"

# Confirm it is collecting query data
sudo -u postgres psql -d postgres \
  -c "SELECT count(*) FROM pg_stat_statements;"
```

---

### Step 3.5 — Test PMM User Can Log in on All Nodes

```bash
# ▶ SERVER: 192.168.109.133
psql -h 127.0.0.1 -U pmm_user -d postgres -p 5432 -c "SELECT version();"

# ▶ SERVER: 192.168.109.134
psql -h 127.0.0.1 -U pmm_user -d postgres -p 5432 -c "SELECT version();"

# ▶ SERVER: 192.168.109.135
psql -h 127.0.0.1 -U pmm_user -d postgres -p 5432 -c "SELECT version();"
```

All three should return the PostgreSQL version string with no errors. If asked for a password, enter `PMMStr0ngP@ss!` to confirm it works.

---

## ══════════════════════════════════════════════
## PART 4 — ADD POSTGRESQL SERVICE TO PMM
## ▶ Each command runs on its own node ONLY
## ══════════════════════════════════════════════

---

### Step 4.0 — Add PostgreSQL on cluster level (Haproxy) 

```bash
# ▶ SERVER: 192.168.109.133 ONLY

sudo pmm-admin add postgresql \
  --username=pmm_user \
  --password='PMMStr0ngP@ss!' \
  --host=192.168.109.136 \
  --port=5000 \
  --service-name=postgresql-cluster \
  --query-source=pgstatements
```

---


### Step 4.1 — Add PostgreSQL on pg-node-1

```bash
# ▶ SERVER: 192.168.109.133 ONLY

sudo pmm-admin add postgresql \
  --username=pmm_user \
  --password='PMMStr0ngP@ss!' \
  --host=192.168.109.133 \
  --port=5432 \
  --service-name=postgresql-node-1
```

---

### Step 4.2 — Add PostgreSQL on pg-node-2

```bash
# ▶ SERVER: 192.168.109.134 ONLY

sudo pmm-admin add postgresql \
  --username=pmm_user \
  --password='PMMStr0ngP@ss!' \
  --host=192.168.109.134 \
  --port=5432 \
  --service-name=postgresql-node-2
```

---

### Step 4.3 — Add PostgreSQL on pg-node-3

```bash
# ▶ SERVER: 192.168.109.135 ONLY

sudo pmm-admin add postgresql \
  --username=pmm_user \
  --password='PMMStr0ngP@ss!' \
  --host=192.168.109.135 \
  --port=5432 \
  --service-name=postgresql-node-3
```

---

### Step 4.4 — Verify All Services Are Running

```bash
# ▶ SERVER: 192.168.109.133
pmm-admin list
```

Expected output for pg-node-1:

```
Service type   Service name           Address          Status
PostgreSQL     postgresql-node-1      127.0.0.1:5432   RUNNING

Agent type           Status
pmm_agent            RUNNING
postgresql_agent     RUNNING
node_exporter        RUNNING    ← OS metrics collected automatically
```

```bash
# ▶ SERVER: 192.168.109.134
pmm-admin list

# ▶ SERVER: 192.168.109.135
pmm-admin list
```

> `node_exporter` is started automatically by PMM Agent — it handles all OS metrics (CPU, RAM, Disk, Network) with no extra configuration needed.

---

## ══════════════════════════════════════════════
## PART 5 — VERIFY ON PMM SERVER
## ▶ Run on: 192.168.109.128
## ══════════════════════════════════════════════

---

### Step 5.1 — Check Registered Nodes via API (if this donot show anything dont warry go to step 5.3)

```bash
# ▶ SERVER: 192.168.109.128

curl -sk -u admin:YOUR_PMM_PASSWORD \
  https://192.168.109.128/v1/inventory/Nodes \
  | python3 -m json.tool | grep node_name
```

Expected output:

```json
"node_name": "pg-node-1",
"node_name": "pg-node-2",
"node_name": "pg-node-3",
```

---

### Step 5.2 — Check Registered Services via API (if this donot show anything dont warry go to step 5.3)

```bash
# ▶ SERVER: 192.168.109.128

curl -sk -u admin:YOUR_PMM_PASSWORD \
  https://192.168.109.128/v1/inventory/Services \
  | python3 -m json.tool | grep service_name
```

Expected output:

```json
"service_name": "postgresql-node-1",
"service_name": "postgresql-node-2",
"service_name": "postgresql-node-3",
```

---

### Step 5.3 — PMM Web UI — Dashboards to Check

Open `https://192.168.109.128` in a browser and verify:

| Dashboard | Menu Path | What You Should See |
|-----------|-----------|---------------------|
| Node Summary | PMM → OS → Node Summary | CPU, RAM, Disk, Network for all 3 nodes |
| PostgreSQL Overview | PMM → PostgreSQL → PostgreSQL Overview | postgresql-node-1, 2, 3 with active metrics |
| Query Analytics | PMM → Query Analytics | Top queries from all 3 nodes |
| PMM Inventory | PMM → Configuration → PMM Inventory | 3 nodes, 3 PostgreSQL services |

---

## ══════════════════════════════════════════════
## PART 6 — TROUBLESHOOTING
## ══════════════════════════════════════════════

---

### PMM Agent Shows `Connected: false`

```bash
# ▶ SERVER: affected node (192.168.109.133 / .134 / .135)

# Step 1 — Test network connectivity to PMM Server
curl -sk https://192.168.109.128/ping
# Must return: {"data":{}}
# If this fails, check firewall on 192.168.109.128 (ports 80, 443)

# Step 2 — Check agent logs for errors
sudo journalctl -u pmm-agent -n 50 --no-pager

# Step 3 — Restart agent
sudo systemctl restart pmm-agent

# Step 4 — Re-register the node (replace X and IP accordingly)
sudo pmm-admin config \
  --server-url=https://admin:YOUR_PMM_PASSWORD@192.168.109.128 \
  --server-insecure-tls \
  --node-name=pg-node-X \
  --node-address=192.168.109.1XX \
  --force
```

---

### PostgreSQL Service Shows No Data in PMM

```bash
# ▶ SERVER: affected node

# Step 1 — Confirm pmm_user can connect
psql -h 127.0.0.1 -U pmm_user -d postgres -c "SELECT 1;"

# Step 2 — Confirm pg_stat_statements is loaded
sudo -u postgres psql -c \
  "SELECT name, setting FROM pg_settings WHERE name = 'shared_preload_libraries';"
# Must show: pg_stat_statements in the setting value

# Step 3 — Confirm extension exists
sudo -u postgres psql -d postgres -c "\dx pg_stat_statements"

# Step 4 — If pg_hba.conf was changed, reload PostgreSQL
sudo -u postgres psql -c "SELECT pg_reload_conf();"

# Step 5 — Remove and re-add the PostgreSQL service
sudo pmm-admin remove postgresql postgresql-node-X
sudo pmm-admin add postgresql \
  --username=pmm_user \
  --password='PMMStr0ngP@ss!' \
  --host=127.0.0.1 \
  --port=5432 \
  --service-name=postgresql-node-X \
  --query-source=pgstatements
```

---

### OS Metrics Not Showing in PMM

```bash
# ▶ SERVER: affected node

# Check node_exporter is listed and RUNNING
pmm-admin list | grep node_exporter

# If missing, add it manually
sudo pmm-admin add node-exporter

# Check PMM Agent is running
sudo systemctl status pmm-agent
```

---

### PMM Server Container Not Starting

```bash
# ▶ SERVER: 192.168.109.128

# Check Docker is running
sudo systemctl status docker

# Check container status
docker ps -a | grep pmm-server

# Read logs for errors
docker logs pmm-server --tail=50

# Check available disk space (PMM needs space for metrics)
df -h

# Restart container
docker restart pmm-server
```

---

## PART 7 — PORTS REFERENCE

| Port | Service | Server | Direction |
|------|---------|--------|-----------|
| 80 | PMM Web UI (HTTP redirect to HTTPS) | 192.168.109.128 | Inbound |
| 443 | PMM Web UI (HTTPS) + Agent endpoint | 192.168.109.128 | Inbound |
| 5432 | PostgreSQL | .133, .134, .135 | Local only |
| 42000-42005 | PMM Agent ↔ PMM Server | .133, .134, .135 | Outbound to .128 |

---

## PART 8 — MAINTENANCE

### Update PMM Client (Run After Any Percona Release)

```bash
# ▶ SERVER: 192.168.109.133
# ▶ SERVER: 192.168.109.134
# ▶ SERVER: 192.168.109.135

sudo dnf update pmm2-client -y
sudo systemctl restart pmm-agent
pmm-admin --version
```

---

### Update PMM Server Container

```bash
# ▶ SERVER: 192.168.109.128

# Pull newer image
docker pull percona/pmm-server:2

# Stop and remove old container (pmm-data volume is preserved)
docker stop pmm-server
docker rm pmm-server

# Start fresh container using the same data volume
docker run -d \
  --name pmm-server \
  --restart always \
  -p 80:80 \
  -p 443:443 \
  -v pmm-data:/srv \
  percona/pmm-server:2

# Verify
docker ps
docker logs pmm-server --tail=20
```

---

### Backup PMM Data

```bash
# ▶ SERVER: 192.168.109.128

sudo mkdir -p /backup/pmm

docker run --rm \
  -v pmm-data:/data \
  -v /backup/pmm:/backup \
  alpine tar czf /backup/pmm-data-$(date +%F).tar.gz -C /data .

ls -lh /backup/pmm/
```

---

## FINAL CHECKLIST

### PMM Server — 192.168.109.128
- [ ] Docker installed and service is running
- [ ] `docker ps` shows `pmm-server` container as `Up`
- [ ] `https://192.168.109.128` loads the PMM login page
- [ ] Admin password changed from default `admin`
- [ ] Firewall allows ports 80 and 443
- [ ] SELinux set to Permissive

### pg-node-1 — 192.168.109.133
- [ ] PMM Client installed (`pmm-admin --version` returns a version)
- [ ] Registered with `--node-name=pg-node-1`
- [ ] `pmm-admin status` shows `Connected: true`
- [ ] `pmm_user` created in PostgreSQL with `pg_monitor` role
- [ ] `pg_hba.conf` updated with pmm_user localhost entry
- [ ] `pg_stat_statements` extension enabled in postgres database
- [ ] `postgresql-node-1` service added via `pmm-admin add postgresql`
- [ ] `pmm-admin list` shows `postgresql_agent` and `node_exporter` as `RUNNING`
- [ ] Firewall allows ports 42000-42005 and 5432
- [ ] SELinux set to Permissive

### pg-node-2 — 192.168.109.134
- [ ] PMM Client installed and registered (`--node-name=pg-node-2`)
- [ ] `pmm-admin status` shows `Connected: true`
- [ ] `pmm_user` access configured in `pg_hba.conf`
- [ ] `postgresql-node-2` service added and `RUNNING`
- [ ] `node_exporter` running for OS metrics
- [ ] Firewall and SELinux configured

### pg-node-3 — 192.168.109.135
- [ ] PMM Client installed and registered (`--node-name=pg-node-3`)
- [ ] `pmm-admin status` shows `Connected: true`
- [ ] `pmm_user` access configured in `pg_hba.conf`
- [ ] `postgresql-node-3` service added and `RUNNING`
- [ ] `node_exporter` running for OS metrics
- [ ] Firewall and SELinux configured

### PMM Dashboard Verification
- [ ] Node Summary dashboard shows all 3 nodes with live OS metrics
- [ ] PostgreSQL Overview shows postgresql-node-1, postgresql-node-2, postgresql-node-3
- [ ] Query Analytics (QAN) shows queries from all 3 nodes
- [ ] PMM Inventory shows 3 nodes and 3 PostgreSQL services

---

*Scope: OS Metrics + PostgreSQL/Patroni Metrics only*
*OS: Rocky Linux | PMM Version: 2.x*
*PMM Server: 192.168.109.128*
*Client Nodes: 192.168.109.133 | 192.168.109.134 | 192.168.109.135*
