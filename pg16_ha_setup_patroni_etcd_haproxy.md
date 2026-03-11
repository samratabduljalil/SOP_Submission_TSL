# PostgreSQL 16 High Availability with Patroni

> **Stack:** PostgreSQL 16 · Patroni · etcd · HAProxy · Keepalived  
> **Cluster:** pghelp01 (133) · pghelp02 (134) · pghelp03 (135) · VIP: 192.168.109.136

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [Pre-requisites](#2-pre-requisites)
3. [System Preparation (All Nodes)](#3-system-preparation-all-nodes)
4. [Install & Configure etcd](#4-install--configure-etcd)
5. [Install & Configure Keepalived](#5-install--configure-keepalived)
6. [Install & Configure HAProxy](#6-install--configure-haproxy)
7. [Install & Configure Patroni](#7-install--configure-patroni)
8. [Watchdog Setup](#8-watchdog-setup)
9. [Starting the Cluster](#9-starting-the-cluster)
10. [Verification & Testing](#10-verification--testing)
11. [Common Operations](#11-common-operations)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Infrastructure Overview

| Hostname | IP Address       | Role              |
|----------|-----------------|-------------------|
| pghelp01 | 192.168.109.133 | etcd · Patroni · HAProxy · Keepalived MASTER |
| pghelp02 | 192.168.109.134 | etcd · Patroni · HAProxy · Keepalived BACKUP |
| pghelp03 | 192.168.109.135 | etcd · Patroni · HAProxy · Keepalived BACKUP |
| ha-vip   | 192.168.109.136 | Virtual IP (floats to active Keepalived node) |

**Port Map:**

| Port | Service | Purpose |
|------|---------|---------|
| 2379 | etcd | Client connections |
| 2380 | etcd | Peer communication |
| 5432 | PostgreSQL | Database connections |
| 5000 | HAProxy | Primary (read/write) |
| 5001 | HAProxy | Replicas (read-only, round-robin) |
| 7000 | HAProxy | Stats dashboard |
| 8008 | Patroni REST API | Health checks |

---

## 2. Pre-requisites

- RHEL 9 / Rocky Linux 9 on all nodes
- Network connectivity between all nodes
- PostgreSQL 16 repository configured

---

## 3. System Preparation (All Nodes)

### 3.1 Add postgres user

```bash
useradd postgres
passwd postgres
```

### 3.2 Grant sudo access

Edit `/etc/sudoers`:

```
root     ALL=(ALL) ALL
postgres ALL=(ALL) NOPASSWD:ALL
```

### 3.3 Set hostnames

```bash
# Run on each machine with its respective hostname
sudo hostnamectl set-hostname pghelp01   # change per node
reboot
```

### 3.4 Configure /etc/hosts (all nodes)

```
192.168.109.133   pghelp01
192.168.109.134   pghelp02
192.168.109.135   pghelp03
192.168.109.136   ha-vip
```

### 3.5 Disable SELinux

```bash
nano /etc/selinux/config
```

Change:
```
SELINUX=enforcing
```
To:
```
SELINUX=disabled
```

```bash
reboot
```

### 3.6 Disable firewalld

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

### 3.7 Install PostgreSQL 16

```bash
# Add the official PostgreSQL YUM repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable the built-in PostgreSQL module to avoid conflicts
sudo dnf -qy module disable postgresql

# Install PostgreSQL 16
sudo dnf install -y postgresql16-server
sudo dnf install -y postgresql16-contrib
```

> **Important:** Do NOT run `postgresql-16-setup initdb`. Patroni will initialize the data directory.  
> Do NOT enable or start the `postgresql-16` service — Patroni manages PostgreSQL.

---

## 4. Install & Configure etcd

### 4.1 Install (all nodes)

```bash
dnf install 'dnf-command(config-manager)'
dnf config-manager --enable pgdg-rhel9-extras
dnf -y install etcd
```

### 4.2 Configure etcd

Backup the original config and create a new one:

```bash
mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf.orig
nano /etc/etcd/etcd.conf
```

#### pghelp01 — `/etc/etcd/etcd.conf`

```ini
ETCD_NAME=pghelp01
ETCD_DATA_DIR="/var/lib/etcd/pghelp01"

ETCD_LISTEN_PEER_URLS="http://192.168.109.133:2380,http://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.109.133:2379,http://127.0.0.1:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.109.133:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.109.133:2379"

ETCD_INITIAL_CLUSTER="pghelp01=http://192.168.109.133:2380,pghelp02=http://192.168.109.134:2380,pghelp03=http://192.168.109.135:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ENABLE_V2="true"
```

#### pghelp02 — `/etc/etcd/etcd.conf`

```ini
ETCD_NAME=pghelp02
ETCD_DATA_DIR="/var/lib/etcd/pghelp02"

ETCD_LISTEN_PEER_URLS="http://192.168.109.134:2380,http://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.109.134:2379,http://127.0.0.1:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.109.134:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.109.134:2379"

ETCD_INITIAL_CLUSTER="pghelp01=http://192.168.109.133:2380,pghelp02=http://192.168.109.134:2380,pghelp03=http://192.168.109.135:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ENABLE_V2="true"
```

#### pghelp03 — `/etc/etcd/etcd.conf`

```ini
ETCD_NAME=pghelp03
ETCD_DATA_DIR="/var/lib/etcd/pghelp03"

ETCD_LISTEN_PEER_URLS="http://192.168.109.135:2380,http://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.109.135:2379,http://127.0.0.1:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.109.135:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.109.135:2379"

ETCD_INITIAL_CLUSTER="pghelp01=http://192.168.109.133:2380,pghelp02=http://192.168.109.134:2380,pghelp03=http://192.168.109.135:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ENABLE_V2="true"
```

### 4.3 Start etcd (all nodes)

```bash
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

### 4.4 Configure bash_profile (all nodes)

Add to `~/.bash_profile`:

```bash
export pghelp01=192.168.109.133
export pghelp02=192.168.109.134
export pghelp03=192.168.109.135
export ENDPOINTS="http://192.168.109.133:2379,http://192.168.109.134:2379,http://192.168.109.135:2379"
```

```bash
source ~/.bash_profile
```

### 4.5 Verify etcd cluster

```bash
etcdctl endpoint status --write-out=table --endpoints=$ENDPOINTS
etcdctl endpoint health --write-out=table --endpoints=$ENDPOINTS
```

Expected output — all three nodes healthy:
```
+------------------------------+------------------+--------+
| ENDPOINT                     | HEALTH           | TOOK   |
+------------------------------+------------------+--------+
| http://192.168.109.133:2379  | true             | ...    |
| http://192.168.109.134:2379  | true             | ...    |
| http://192.168.109.135:2379  | true             | ...    |
+------------------------------+------------------+--------+
```

---

## 5. Install & Configure Keepalived

### 5.1 Install (all nodes)

```bash
dnf -y install keepalived
```

### 5.2 Enable IP non-local bind (all nodes)

```bash
nano /etc/sysctl.conf
```

Add:
```
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
```

```bash
sudo sysctl --system
sudo sysctl -p
```

### 5.3 Configure Keepalived

```bash
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.orig
nano /etc/keepalived/keepalived.conf
```

#### pghelp01 — MASTER

```
vrrp_script check_haproxy {
  script "pkill -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  state MASTER
  interface ens160
  virtual_router_id 51
  priority 101
  advert_int 1
  virtual_ipaddress {
    192.168.109.136
  }
  track_script {
    check_haproxy
  }
}
```

#### pghelp02 and pghelp03 — BACKUP

```
vrrp_script check_haproxy {
  script "pkill -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  state BACKUP
  interface ens160
  virtual_router_id 51
  priority 100
  advert_int 1
  virtual_ipaddress {
    192.168.109.136
  }
  track_script {
    check_haproxy
  }
}
```

> **Note:** Change `priority` to a lower value (e.g. 99) on pghelp03 so pghelp02 is preferred backup.

### 5.4 Start Keepalived (all nodes)

```bash
systemctl enable keepalived
systemctl start keepalived
```

### 5.5 Verify VIP

On pghelp01 (MASTER), the VIP should appear on the interface:

```bash
ip addr show ens160
```

You should see `192.168.109.136` listed.

---

## 6. Install & Configure HAProxy

### 6.1 Install (all nodes)

```bash
dnf -y install haproxy
```

### 6.2 Configure HAProxy

```bash
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
nano /etc/haproxy/haproxy.cfg
```

#### `/etc/haproxy/haproxy.cfg` (same on all nodes)

```
global
    maxconn     1000

defaults
    mode                    tcp
    log                     global
    option                  tcplog
    retries                 3
    timeout queue           1m
    timeout connect         4s
    timeout client          60m
    timeout server          60m
    timeout check           5s
    maxconn                 900

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen primary
    bind 192.168.109.136:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pghelp01 192.168.109.133:5432 maxconn 100 check port 8008
    server pghelp02 192.168.109.134:5432 maxconn 100 check port 8008
    server pghelp03 192.168.109.135:5432 maxconn 100 check port 8008

listen standby
    bind 192.168.109.136:5001
    balance roundrobin
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pghelp01 192.168.109.133:5432 maxconn 100 check port 8008
    server pghelp02 192.168.109.134:5432 maxconn 100 check port 8008
    server pghelp03 192.168.109.135:5432 maxconn 100 check port 8008
```

### 6.3 Copy config to other nodes

```bash
scp /etc/haproxy/haproxy.cfg pghelp02:/etc/haproxy/haproxy.cfg
scp /etc/haproxy/haproxy.cfg pghelp03:/etc/haproxy/haproxy.cfg
```

### 6.4 Start HAProxy (all nodes)

```bash
systemctl start haproxy
systemctl enable haproxy
systemctl status haproxy
```

---

## 7. Install & Configure Patroni

### 7.1 Install (all nodes)

```bash
dnf -y install patroni
dnf -y install patroni-etcd
dnf -y install watchdog
```

### 7.2 Create data directory (all nodes)

```bash
mkdir -p /data/pgsql/16/data
chown -R postgres:postgres /data/pgsql/16
chmod 700 /data/pgsql/16/data
```

### 7.3 Create socket directory (all nodes)

```bash
mkdir -p /run/postgresql
chown postgres:postgres /run/postgresql
chmod 775 /run/postgresql

# Persist across reboots
echo 'd /run/postgresql 0775 postgres postgres -' > /etc/tmpfiles.d/postgresql.conf
```

### 7.4 Create Patroni configuration

```bash
mkdir -p /etc/patroni
nano /etc/patroni/patroni.yml
```

#### pghelp01 — `/etc/patroni/patroni.yml`

```yaml
scope: postgres
namespace: /db/
name: pghelp01

restapi:
    listen: 192.168.109.133:8008
    connect_address: 192.168.109.133:8008

etcd3:
    hosts: 192.168.109.133:2379,192.168.109.134:2379,192.168.109.135:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true
    initdb:
    - encoding: UTF8
    - data-checksums
    pg_hba:
    - host replication replicator 192.168.109.133/32 scram-sha-256
    - host replication replicator 192.168.109.134/32 scram-sha-256
    - host replication replicator 192.168.109.135/32 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

postgresql:
    listen: 192.168.109.133:5432
    connect_address: 192.168.109.133:5432
    data_dir: /data/pgsql/16/data
    bin_dir: /usr/pgsql-16/bin
    authentication:
        replication:
            username: replicator
            password: replicator
        superuser:
            username: postgres
            password: postgres
    parameters:
        unix_socket_directories: '/run/postgresql'

watchdog:
    mode: off
    device: /dev/watchdog
    safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

#### pghelp02 — `/etc/patroni/patroni.yml`

```yaml
scope: postgres
namespace: /db/
name: pghelp02

restapi:
    listen: 192.168.109.134:8008
    connect_address: 192.168.109.134:8008

etcd3:
    hosts: 192.168.109.133:2379,192.168.109.134:2379,192.168.109.135:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true
    initdb:
    - encoding: UTF8
    - data-checksums
    pg_hba:
    - host replication replicator 192.168.109.133/32 scram-sha-256
    - host replication replicator 192.168.109.134/32 scram-sha-256
    - host replication replicator 192.168.109.135/32 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

postgresql:
    listen: 192.168.109.134:5432
    connect_address: 192.168.109.134:5432
    data_dir: /data/pgsql/16/data
    bin_dir: /usr/pgsql-16/bin
    authentication:
        replication:
            username: replicator
            password: replicator
        superuser:
            username: postgres
            password: postgres
    parameters:
        unix_socket_directories: '/run/postgresql'

watchdog:
    mode: off
    device: /dev/watchdog
    safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

#### pghelp03 — `/etc/patroni/patroni.yml`

```yaml
scope: postgres
namespace: /db/
name: pghelp03

restapi:
    listen: 192.168.109.135:8008
    connect_address: 192.168.109.135:8008

etcd3:
    hosts: 192.168.109.133:2379,192.168.109.134:2379,192.168.109.135:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true
    initdb:
    - encoding: UTF8
    - data-checksums
    pg_hba:
    - host replication replicator 192.168.109.133/32 scram-sha-256
    - host replication replicator 192.168.109.134/32 scram-sha-256
    - host replication replicator 192.168.109.135/32 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

postgresql:
    listen: 192.168.109.135:5432
    connect_address: 192.168.109.135:5432
    data_dir: /data/pgsql/16/data
    bin_dir: /usr/pgsql-16/bin
    authentication:
        replication:
            username: replicator
            password: replicator
        superuser:
            username: postgres
            password: postgres
    parameters:
        unix_socket_directories: '/run/postgresql'

watchdog:
    mode: off
    device: /dev/watchdog
    safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

> **Critical YAML rules:**
> - Use spaces only — no tabs
> - `use_pg_rewind: true` must be indented under `postgresql:` inside `dcs:`
> - `initdb:` section is required — without it Patroni will never bootstrap
> - Use `/32` not `/0` in pg_hba replication entries

---

## 8. Watchdog Setup (Optional)

> **Watchdog is not mandatory.** The cluster will still perform automatic failover without it. Watchdog provides an extra safety layer by forcefully rebooting a node that loses the DCS lock, preventing split-brain in extreme network partition scenarios. For most environments `mode: off` is perfectly fine.

**Watchdog modes:**

| Mode | Behaviour |
|------|-----------|
| `off` | Watchdog disabled. Automatic failover still works via etcd. ✅ Recommended for most setups |
| `automatic` | Uses watchdog if available, falls back gracefully if not |
| `required` | Patroni refuses to start if watchdog is not accessible |

The configs in this guide use **`mode: off`**. If you want to enable watchdog later, run the following on all nodes:

```bash
# Load softdog kernel module
modprobe softdog

# Create device node if it doesn't exist
ls /dev/watchdog || mknod /dev/watchdog c 10 130

# Set correct ownership and permissions
chown postgres:postgres /dev/watchdog
chmod 600 /dev/watchdog

# Verify
ls -la /dev/watchdog
# Expected: crw------- 1 postgres root 10, 130 ...

# Persist across reboots
echo 'softdog' > /etc/modules-load.d/softdog.conf

cat > /etc/udev/rules.d/99-watchdog.rules << 'EOF'
KERNEL=="watchdog", OWNER="postgres", GROUP="root", MODE="0600"
EOF

udevadm control --reload-rules
```

Verify postgres user can write to watchdog:

```bash
sudo -u postgres bash -c 'test -w /dev/watchdog && echo "WRITABLE OK" || echo "CANNOT WRITE"'
```

Then change `mode: off` to `mode: required` in all patroni.yml files and restart Patroni.

---

## 9. Starting the Cluster

> **Important:** Always start pghelp01 first and wait for it to become Leader before starting other nodes.

### Step 1 — Disable standalone PostgreSQL (all nodes)

```bash
systemctl stop postgresql-16
systemctl disable postgresql-16
```

### Step 2 — Start Patroni on pghelp01 only

```bash
systemctl start patroni
journalctl -u patroni -f
```

Wait for these log lines:
```
INFO: trying to bootstrap a new cluster
INFO: initialized a new cluster
INFO: no action. I am (pghelp01), the leader with the lock
```

### Step 3 — Verify pghelp01 is Leader

```bash
patronictl -c /etc/patroni/patroni.yml list
```

### Step 4 — Start Patroni on pghelp02 and pghelp03

```bash
systemctl start patroni
journalctl -u patroni -f
```

You should see:
```
INFO: trying to bootstrap from leader 'pghelp01'
INFO: clone from 'pghelp01' in progress
```

### Step 5 — Enable Patroni on boot (all nodes)

```bash
systemctl enable patroni
```

---

## 10. Verification & Testing

### Check cluster status

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Expected:
```
+----------+-----------------+---------+-----------+----+-----------+
| Member   | Host            | Role    | State     | TL | Lag in MB |
+----------+-----------------+---------+-----------+----+-----------+
| pghelp01 | 192.168.109.133 | Leader  | running   |  1 |           |
| pghelp02 | 192.168.109.134 | Replica | streaming |  1 |         0 |
| pghelp03 | 192.168.109.135 | Replica | streaming |  1 |         0 |
+----------+-----------------+---------+-----------+----+-----------+
```

### Check where writes go (primary — port 5000)

```bash
psql -h 192.168.109.136 -p 5000 -U postgres \
  -c "SELECT inet_server_addr() as node, pg_is_in_recovery() as is_replica;"
```

Expected: `is_replica = f` (false = primary, accepts writes)

### Check where reads go (replicas — port 5001)

```bash
psql -h 192.168.109.136 -p 5001 -U postgres \
  -c "SELECT inet_server_addr() as node, pg_is_in_recovery() as is_replica;"
```

Expected: `is_replica = t` (true = replica, read-only)

### Test round-robin load balancing

```bash
for i in {1..6}; do
  psql -h 192.168.109.136 -p 5001 -U postgres \
    -c "SELECT inet_server_addr();" -t
done
```

### Check Patroni REST API

```bash
curl -s http://192.168.109.133:8008/patroni | python3 -m json.tool
```

### Check HAProxy stats

Open in browser: `http://192.168.109.136:7000/`

### Test automatic failover

```bash
# Stop the current leader
systemctl stop patroni   # run on the leader node

# Watch from another node — new leader elected within ~30 seconds
patronictl -c /etc/patroni/patroni.yml list
```

### Manual switchover

```bash
patronictl -c /etc/patroni/patroni.yml switchover postgres
```

---

## 11. Common Operations

### Connect to the database

```bash
# Write connection (always goes to primary)
psql -h 192.168.109.136 -p 5000 -U postgres

# Read connection (round-robin across replicas)
psql -h 192.168.109.136 -p 5001 -U postgres
```

### Create a database

```sql
CREATE DATABASE mydb;
\l
\c mydb

CREATE TABLE test (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO test (name) VALUES ('hello'), ('world');
SELECT * FROM test;
```

### Verify replication

```bash
# Check the database exists on a replica
psql -h 192.168.109.133 -p 5432 -U postgres -c "\l"
```

### Reinitialize a replica

```bash
patronictl -c /etc/patroni/patroni.yml reinit postgres pghelp02
```

### Pause/resume Patroni auto-failover

```bash
patronictl -c /etc/patroni/patroni.yml pause
patronictl -c /etc/patroni/patroni.yml resume
```

---

## 12. Troubleshooting

### Empty cluster list

```bash
# 1. Check etcd is healthy
etcdctl endpoint health --endpoints=$ENDPOINTS

# 2. Clean stale etcd data
etcdctl del /db/ --prefix --endpoints=$ENDPOINTS

# 3. Stop patroni on all nodes, start pghelp01 first
systemctl stop patroni   # all nodes
systemctl start patroni  # pghelp01 only first
```

### "waiting for leader to bootstrap"

Causes and fixes:

| Cause | Fix |
|-------|-----|
| Missing `initdb:` in patroni.yml | Add `initdb:` section under `bootstrap:` |
| Wrong YAML indentation | `use_pg_rewind` must be under `postgresql:` inside `dcs:` |
| Watchdog blocking start | Set `mode: off` in patroni.yml — watchdog is optional
| Stale etcd key | `etcdctl del /db/ --prefix --endpoints=$ENDPOINTS` |
| Data dir not empty | `rm -rf /data/pgsql/16/data/*` |

### Watchdog issue (if mode is required)

> If you set `mode: required` and Patroni won't start, run:

```bash
rmmod softdog
modprobe softdog
chown postgres:postgres /dev/watchdog
chmod 600 /dev/watchdog
sudo -u postgres bash -c 'test -w /dev/watchdog && echo "OK"'
```

> Or simply set `mode: off` in patroni.yml — failover still works without watchdog.

### PostgreSQL connection refused

```bash
# Check if standalone postgresql-16 service is running (it must be disabled)
systemctl status postgresql-16
systemctl stop postgresql-16
systemctl disable postgresql-16

# Check data dir ownership
ls -la /data/pgsql/16/data/
chown -R postgres:postgres /data/pgsql/16
chmod 700 /data/pgsql/16/data
```

### $ENDPOINTS variable empty

```bash
# Add http:// prefix — required for etcdctl
export ENDPOINTS="http://192.168.109.133:2379,http://192.168.109.134:2379,http://192.168.109.135:2379"
```

### Useful diagnostic commands

```bash
# Live Patroni logs
journalctl -u patroni -f

# etcd cluster keys
etcdctl get /db/ --prefix --endpoints=$ENDPOINTS

# Patroni REST API status
curl -s http://192.168.109.133:8008/patroni | python3 -m json.tool

# Check all ports are listening
ss -tlnp | grep -E '5432|8008|2379|2380'

# Check watchdog is held open by patroni
lsof /dev/watchdog

# Check replication lag
psql -h 192.168.109.136 -p 5000 -U postgres \
  -c "SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;"
```

---

*PostgreSQL 16 HA Cluster — pghelp01 · pghelp02 · pghelp03*
