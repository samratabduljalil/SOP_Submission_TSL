# pgBackRest Integration with Patroni HA Cluster
## Setup & Operations Guide

> **Version:** 2.0 | **Date:** March 2026
> **Stack:** PostgreSQL 16 · Patroni · etcd · HAProxy · Keepalived · pgBackRest
> **OS:** Rocky Linux 9
> **pgBackRest User:** `postgres` on all 4 servers

---

## Table of Contents

1. [What is pgBackRest & Why You Need It](#1-what-is-pgbackrest--why-you-need-it)
2. [How It Fits Into the Existing Setup](#2-how-it-fits-into-the-existing-setup)
3. [Server & IP Reference](#3-server--ip-reference)
4. [Pre-Installation Checklist](#4-pre-installation-checklist)
5. [Phase 1 — Prerequisites](#5-phase-1--prerequisites)
   - [5.1 Install pgBackRest](#51-install-pgbackrest)
   - [5.2 Create Repository & Log Directories](#52-create-repository--log-directories)
   - [5.3 Set Up postgres User SSH](#53-set-up-postgres-user-ssh)
   - [5.4 Configure Firewall](#54-configure-firewall)
6. [Phase 2 — Configuration](#6-phase-2--configuration)
   - [6.1 Configure pgbackrest.conf on All 4 Servers](#61-configure-pgbackrestconf-on-all-4-servers)
   - [6.2 Enable WAL Archiving via Patroni](#62-enable-wal-archiving-via-patroni)
7. [Phase 3 — Initialize & First Backup](#7-phase-3--initialize--first-backup)
   - [7.1 Create the Stanza](#71-create-the-stanza)
   - [7.2 Verify Everything is Wired Up](#72-verify-everything-is-wired-up)
   - [7.3 Take the First Full Backup](#73-take-the-first-full-backup)
8. [Phase 4 — Integrate with Patroni Replica Cloning](#8-phase-4--integrate-with-patroni-replica-cloning)
9. [Phase 5 — Schedule Automatic Backups](#9-phase-5--schedule-automatic-backups)
10. [Restore Operations](#10-restore-operations)
11. [Verify & Monitor](#11-verify--monitor)
12. [Failover Behavior](#12-failover-behavior)
13. [Troubleshooting](#13-troubleshooting)
14. [Quick Reference](#14-quick-reference)

---

## 1. What is pgBackRest & Why You Need It

pgBackRest is a dedicated PostgreSQL backup and restore tool. It replaces basic `pg_dump` for production use.

### pg_dump vs pgBackRest

| | pg_dump | pgBackRest |
|---|---|---|
| Type | Logical dump (SQL) | Physical backup (file-level) |
| Speed on large DBs | Slow | Fast |
| Point-in-time recovery | ❌ No | ✅ Yes |
| Incremental backups | ❌ No | ✅ Yes |
| WAL archiving | ❌ No | ✅ Yes |
| Works with Patroni | Manual | Native integration |

### Why streaming replication alone is not enough

Streaming replication is **HA, not backup**. It replicates everything — including accidental `DROP DATABASE`, data corruption, and bad migrations — to all replicas instantly. If data is destroyed on the primary, all replicas reflect that destruction within seconds.

pgBackRest gives you:
- **Full backups** — complete snapshot of the database cluster
- **Differential backups** — only changed blocks since last full backup
- **Incremental backups** — only changed blocks since last backup of any type
- **WAL archiving** — continuous shipping of WAL files to the repository
- **Point-in-time recovery (PITR)** — restore to any exact second in the past
- **Replica-based backups** — backup runs from a replica, zero load on the primary

---

## 2. How It Fits Into the Existing Setup



### Node Roles

| Hostname | IP | Role | Components |
|----------|----|------|------------|
| pgnode1 | `192.168.109.133` | Database | etcd, Patroni, PostgreSQL 16, pgBackRest client |
| pgnode2 | `192.168.109.134` | Database | etcd, Patroni, PostgreSQL 16, pgBackRest client |
| pgnode3 | `192.168.109.135` | Database | etcd, Patroni, PostgreSQL 16, pgBackRest client |
| repo | `192.168.109.128` | Backup Repository | pgBackRest repository host |

### Port Reference

| Port | Service | Purpose |
|------|---------|---------|
| 5000 | Repo | Routes to current primary |
| 5001 | Repo | Routes to replicas |
| 5432 | PostgreSQL | Direct database connections |
| 2379 | etcd | Client communication |
| 2380 | etcd | Peer communication |
| 8008 | Patroni | REST API / health checks |
| 22 | SSH | pgBackRest file transfer between nodes and repository |

---

## 3. Server & IP Reference

| Parameter | Value |
|-----------|-------|
| pgnode1 IP | `192.168.109.133` |
| pgnode2 IP | `192.168.109.134` |
| pgnode3 IP | `192.168.109.135` |
| pgbackrest(repo) | `192.168.109.128` |
| PostgreSQL data directory | `/data/pgsql/16/data/` |
| PostgreSQL port | `5432` |
| pgBackRest OS user | `postgres` (all 4 servers) |
| pgBackRest stanza name | `postgres` |
| Backup repository path | `/var/lib/pgbackrest` |
| pgBackRest config file | `/etc/pgbackrest/pgbackrest.conf` |
| Log directory | `/var/log/pgbackrest` |
| postgres SSH key directory | `/var/lib/pgsql/.ssh/` |

---

## 4. Pre-Installation Checklist

Run on **all 3 DB nodes** before starting:

```bash
# Confirm all services are running
sudo systemctl status postgresql-16
sudo systemctl status patroni
sudo systemctl status etcd

# Check Patroni cluster health — identify which node is primary
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres

# Confirm PostgreSQL data directory
sudo -u postgres psql -c "SHOW data_directory;"
# Expected: /data/pgsql/16/data/

# Confirm archive_mode is currently off (expected before this guide)
sudo -u postgres psql -c "SHOW archive_mode;"

# Check Rocky Linux version
cat /etc/rocky-release

# Check disk space on repo — needs 2-3× your total database size
# Run on repo
df -h /var/lib/
```

---

## 5. Phase 1 — Prerequisites

### 5.1 Install pgBackRest

> **Run on ALL 4 servers: pgnode1, pgnode2, pgnode3, and repo**

#### Enable the PostgreSQL PGDG repository

```bash
# Rocky Linux 9
sudo dnf install -y \
  https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Rocky Linux 8 — use this line instead
# sudo dnf install -y \
#   https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable built-in Rocky Linux PostgreSQL module to avoid conflicts
sudo dnf -qy module disable postgresql

# Refresh package metadata
sudo dnf makecache
```

#### Install pgBackRest

```bash
#if needed (where patroni not available) :
sudo dnf -qy module disable PostgreSQL

#install pg_backrest with necessary packages: 

sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb
sudo dnf clean all
sudo dnf makecache
sudo dnf install -y libssh2
sudo dnf install -y pgbackrest –nobest
pgbackrest version
```

**Expected output:**
```
pgBackRest 2.49
```

> ⚠️ **pgBackRest must be the exact same version on all 4 servers.**
> Run `pgbackrest version` on each server and compare before continuing.
> A version mismatch causes SSH protocol errors during archiving and backup.

---

### 5.2 Create Repository & Log Directories

#### On repo ONLY — create the backup repository

```bash
sudo mkdir -p /var/lib/pgbackrest
sudo chmod 750 /var/lib/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest

# Verify
ls -la /var/lib/pgbackrest
```

#### On ALL 4 servers (pgnode1, pgnode2, pgnode3, repo) — create log directory

```bash
sudo mkdir -p /var/log/pgbackrest
sudo chown postgres:postgres /var/log/pgbackrest
sudo chmod 750 /var/log/pgbackrest

# Verify
ls -la /var/log/pgbackrest
```

#### On ALL 4 servers — create config directory

```bash
sudo mkdir -p /etc/pgbackrest
sudo chmod 755 /etc/pgbackrest
```

---

### 5.3 Set Up postgres User SSH

pgBackRest transfers files over SSH as the `postgres` user. The `postgres` user exists on all DB nodes (created automatically by the PostgreSQL package) and needs SSH keys set up. On repo the user also exists since it is installed as part of system setup, but verify first.

#### Verify postgres user exists on all 4 servers

```bash
getent passwd postgres
# Expected: postgres:x:26:26:PostgreSQL Server:/var/lib/pgsql:/bin/bash
```

If the user does not exist on repo, create it:

```bash
# Only if getent returns nothing on repo
sudo useradd --system \
  --shell /bin/bash \
  --home-dir /var/lib/pgsql \
  --comment "PostgreSQL Server" \
  postgres

sudo mkdir -p /var/lib/pgsql
sudo chown postgres:postgres /var/lib/pgsql
sudo chmod 700 /var/lib/pgsql
```

---

#### Set a password for postgres on all 4 servers

This is required so that `ssh-copy-id` can authenticate for the initial key exchange.

```bash
# Run on pgnode1, pgnode2, pgnode3, and repo
sudo passwd postgres
```

> You can set the same password on all servers to make key exchange easier.
> The password is only needed once — SSH key auth is used after setup.

---

#### Create .ssh directory on all 4 servers

```bash
# Run on pgnode1, pgnode2, pgnode3, and repo
sudo mkdir -p /var/lib/pgsql/.ssh
sudo chown postgres:postgres /var/lib/pgsql/.ssh
sudo chmod 700 /var/lib/pgsql/.ssh
```

---

#### Generate SSH key as postgres on all 4 servers

Run on **pgnode1, pgnode2, pgnode3, and repo** — one at a time:

```bash
sudo -u postgres ssh-keygen -t rsa -b 4096 -N "" -f /var/lib/pgsql/.ssh/id_rsa
```

**Expected output:**
```
Generating public/private rsa key pair.
Your identification has been saved in /var/lib/pgsql/.ssh/id_rsa
Your public key has been saved in /var/lib/pgsql/.ssh/id_rsa.pub
```

---

#### Copy DB node keys to repo

Run on each DB node — this copies that node's public key into repo's `authorized_keys`:

```bash
# On pgnode1 (192.168.109.133):
sudo -u postgres ssh-copy-id postgres@192.168.109.128

# On pgnode2 (192.168.109.134):
sudo -u postgres ssh-copy-id postgres@192.168.109.128

# On pgnode3 (192.168.109.135):
sudo -u postgres ssh-copy-id postgres@192.168.109.128
```

You will be prompted for the postgres password on repo each time. Enter it.

---

#### Copy repo key to all DB nodes

Run on **repo** — this copies repo's public key to each DB node:

```bash
# On repo (192.168.109.128):
sudo -u postgres ssh-copy-id postgres@192.168.109.133
sudo -u postgres ssh-copy-id postgres@192.168.109.134
sudo -u postgres ssh-copy-id postgres@192.168.109.135
```

You will be prompted for the postgres password on each DB node. Enter it.

---

#### Test all SSH connections

Every connection must return the hostname **with no password prompt**:

```bash
# From repo — test connections to all 3 DB nodes:
sudo -u postgres ssh postgres@192.168.109.133 "hostname"
sudo -u postgres ssh postgres@192.168.109.134 "hostname"
sudo -u postgres ssh postgres@192.168.109.135 "hostname"

# From pgnode1 — test connection to repo:
sudo -u postgres ssh postgres@192.168.109.128 "hostname"

# From pgnode2 — test connection to repo:
sudo -u postgres ssh postgres@192.168.109.128 "hostname"

# From pgnode3 — test connection to repo:
sudo -u postgres ssh postgres@192.168.109.128 "hostname"
```

> ❌ If any connection asks for a password, SSH is not set up correctly.
> Do not proceed to Phase 2 until all connections are passwordless.

---

### 5.4 Configure Firewall

pgBackRest communicates over SSH (port 22) only. Ensure SSH is open between all nodes on the cluster subnet.

```bash
# Check firewalld is running — run on all 4 servers
sudo systemctl status firewalld

# Allow SSH
sudo firewall-cmd --permanent --add-service=ssh

# Allow SSH restricted to the cluster subnet only (more secure)
sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="192.168.109.0/24" service name="ssh" accept'

# Reload firewall
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-all
```

---

## 6. Phase 2 — Configuration

### 6.1 Configure pgbackrest.conf on All 4 Servers

The DB nodes (pgnode1/2/3) are **clients** — they point to repo as the repository host.
repo is the **repository host** — it stores all backups and defines all 3 DB nodes.

---

#### On ALL DB nodes (pgnode1, pgnode2, pgnode3) — same config on all three

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
# Repository is on repo
repo1-host=192.168.109.128
repo1-host-user=postgres
repo1-path=/var/lib/pgbackrest

# Retention policy
repo1-retention-full=2
repo1-retention-diff=7

# Logging
log-level-console=info
log-level-file=detail

# Performance options
start-fast=y
delta=y

[postgres]
pg1-path=/data/pgsql/16/data/
pg1-port=5432
```

Set correct ownership:

```bash
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
```

---

#### On repo ONLY — different config (repo IS the repo host)

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
# repo stores the repository locally
repo1-path=/var/lib/pgbackrest

# Retention policy
repo1-retention-full=2
repo1-retention-diff=7

# Logging
log-level-console=info
log-level-file=detail

# Performance options
start-fast=y
delta=y

# Backup from standby — reduces load on primary
#backup-standby=y

[postgres]
# Node 1
pg1-path=/data/pgsql/16/data/
pg1-host=192.168.109.133
pg1-host-user=postgres
pg1-port=5432

# Node 2
pg2-path=/data/pgsql/16/data/
pg2-host=192.168.109.134
pg2-host-user=postgres
pg2-port=5432

# Node 3
pg3-path=/data/pgsql/16/data/
pg3-host=192.168.109.135
pg3-host-user=postgres
pg3-port=5432
```

Set correct ownership:

```bash
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
```

---

### 6.2 Enable WAL Archiving via Patroni

> **Important:** WAL archiving must be enabled through `patronictl edit-config` — not by editing `patroni.yml` directly.
>
> `patronictl edit-config` writes to etcd and applies to the **entire live cluster**.
> Editing only the local `patroni.yml` file would apply to one node and be overwritten on reload.

#### Edit the live cluster config — run ONCE from any DB node

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
```

This opens the cluster config in your default editor. Add the `parameters:` block inside the `postgresql:` section. Your config should look like this after editing:

```yaml
loop_wait: 10
maximum_lag_on_failover: 1048576
postgresql:
  pg_hba:
  - host replication replicator 192.168.109.133/32 scram-sha-256
  - host replication replicator 192.168.109.134/32 scram-sha-256
  - host replication replicator 192.168.109.135/32 scram-sha-256
  - host all all 0.0.0.0/0 scram-sha-256
  - local all  postgres trust 
  parameters:
    archive_command: pgbackrest --stanza=postgres archive-push %p
    archive_mode: 'on'
    archive_timeout: 60
    max_wal_senders: 10
    wal_level: replica
    restore_command: pgbackrest --stanza=postgres archive-get %f "%p"
  use_pg_rewind: true
retry_timeout: 10
ttl: 30
```

> **Indentation is critical.** Use exactly 2 spaces for `parameters:` and 4 spaces for each parameter under it. YAML is whitespace-sensitive.

Save and exit the editor.

---

#### Verify the config saved correctly

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml show-config postgres
```

Confirm `archive_command`, `archive_mode`, `wal_level`, and `restore_command` appear in the output.

---

#### Reload all nodes to apply the config

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode1
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode3
```

---

#### Restart all nodes to activate archive_mode

`archive_mode` requires a full PostgreSQL restart — reload alone is not sufficient.
Patroni marks nodes with `pending_restart` until this is done.

Check for pending restart:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres
```

Look for `pending_restart: true` in the Tags column. Then restart — **replicas first, leader last** to avoid an unnecessary failover:

```bash
# Restart replicas first
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode3

# Restart primary last
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode1
```

---

#### Confirm archive_mode is active

```bash
sudo -u postgres psql -c "SELECT name, setting FROM pg_settings \
    WHERE name IN ('archive_mode','archive_command','wal_level','restore_command');"
```

Expected:

```
    name          |                          setting
------------------+----------------------------------------------------------
 archive_command  | pgbackrest --stanza=postgres archive-push %p
 archive_mode     | on
 restore_command  | pgbackrest --stanza=postgres archive-get %f "%p"
 wal_level        | replica
```

---

## 7. Phase 3 — Initialize & First Backup

### 7.1 Create the Stanza

A stanza is pgBackRest's configuration unit for one PostgreSQL cluster. It is created once and initialises the directory structure inside `/var/lib/pgbackrest/`.

**Ensure the Patroni cluster is fully healthy before running this:**

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres
# All 3 nodes must show 'running'
```

**Run on repo only:**

```bash
sudo -u postgres pgbackrest --stanza=postgres --log-level-console=info stanza-create
```

**Expected output:**

```
INFO: stanza-create command begin
INFO: stanza-create for stanza 'postgres' on repo1
INFO: stanza-create command end: completed successfully
```

> If this fails, it is almost always an SSH connectivity issue.
> Go back to [Step 5.3](#53-set-up-postgres-user-ssh) and re-test all SSH connections.

---

### 7.2 Verify Everything is Wired Up

**Run on repo:**

```bash
sudo -u postgres pgbackrest --stanza=postgres check
```

**Expected output:**

```
INFO: check command begin
INFO: WAL segment 000000010000000000000001 successfully archived
INFO: check command end: completed successfully
```

This confirms:
- pgBackRest can reach all DB nodes over SSH ✅
- WAL archiving is working — a test WAL segment was sent to the repo ✅
- The stanza configuration is correct ✅

---

### 7.3 Take the First Full Backup

**Run on repo:**

```bash
sudo -u postgres pgbackrest --stanza=postgres --log-level-console=info backup --type=full
```

Monitor progress in real time (open a second terminal on repo):

```bash
sudo tail -f /var/log/pgbackrest/postgres-backup.log
```

Check the result:

```bash
sudo -u postgres pgbackrest --stanza=postgres info
```

**Expected output:**

```
stanza: postgres
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 000000010000000000000001/000000010000000000000005

        full backup: 20260329-120000F
            timestamp start/stop: 2026-03-29 12:00:00 / 2026-03-29 12:04:30
            wal start/stop: 000000010000000000000001 / 000000010000000000000005
            database size: 2.4GB, database backup size: 2.4GB
            repo1: backup set size: 800MB, backup size: 800MB
```

---

## 8. Phase 4 — Integrate with Patroni Replica Cloning

This tells Patroni to use `pgbackrest restore` when rebuilding or adding a replica,
instead of the default `pg_basebackup`. For large databases this is much faster
and puts zero streaming load on the primary.

### Step 1 — Enable backup-standby in pgbackrest.conf on repo

Before integrating with Patroni, enable `backup-standby=y` in the repo's pgBackRest config.
This allows backups to run from a standby replica instead of the primary, reducing load.

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

Uncomment (or add) the `backup-standby` line in the `[global]` section:

```ini
[global]
# repo stores the repository locally
repo1-path=/var/lib/pgbackrest

# Retention policy
repo1-retention-full=2
repo1-retention-diff=7

# Logging
log-level-console=info
log-level-file=detail

# Performance options
start-fast=y
delta=y

# Backup from standby — reduces load on primary
backup-standby=y

[postgres]
# Node 1
pg1-path=/data/pgsql/16/data/
pg1-host=192.168.109.133
pg1-host-user=postgres
pg1-port=5432

# Node 2
pg2-path=/data/pgsql/16/data/
pg2-host=192.168.109.134
pg2-host-user=postgres
pg2-port=5432

# Node 3
pg3-path=/data/pgsql/16/data/
pg3-host=192.168.109.135
pg3-host-user=postgres
pg3-port=5432
```

> **Note:** With `backup-standby=y`, pgBackRest connects to the primary for metadata but reads
> the actual data files from a standby. All 3 DB node entries (pg1, pg2, pg3) must be defined
> so pgBackRest can identify which node is the current standby at backup time.

---

### Step 2 — Edit `patroni.yml` on ALL 3 DB nodes

```bash
sudo nano /etc/patroni/patroni.yml
```

Add the `create_replica_methods` block inside `bootstrap:`:

```yaml
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

  # Try pgbackrest first. Fall back to basebackup if repo
  # is unreachable or if restore fails.
  create_replica_methods:
    - pgbackrest
    - basebackup

  pgbackrest:
    command: >-
      pgbackrest
      --stanza=postgres
      --delta
      --type=standby
      --target-timeline=latest
      restore
    keep_data: True
    no_params: True

  basebackup:
    - checkpoint: fast
    - max-rate: 100M
```

**Flag explanations:**

| Flag | Purpose |
|------|---------|
| `--stanza=postgres` | Use the postgres stanza from pgbackrest.conf |
| `--delta` | Only restore changed files — much faster than a full restore |
| `--type=standby` | Write `standby.signal` so PostgreSQL starts as a streaming replica |
| `--target-timeline=latest` | Follow the most recent timeline — essential after a failover |
| `keep_data: True` | Do not delete the existing data directory before restoring |
| `no_params: True` | pgBackRest reads its own config — Patroni must not inject extra flags |

### Step 3 — Reload Patroni on all 3 DB nodes

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode1
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode3
```

---

## 9. Phase 5 — Schedule Automatic Backups

**On repo — edit the postgres user crontab:**

```bash
sudo -u postgres crontab -e
```

Add the following schedule:

```cron
# Full backup every Sunday at 1:00 AM
0 1 * * 0 pgbackrest --stanza=postgres backup --type=full

# Incremental backup Monday–Saturday at 1:00 AM
0 1 * * 1-6 pgbackrest --stanza=postgres backup --type=incr

# Differential backup at 7:00 AM and 7:00 PM daily
0 7,19 * * * pgbackrest --stanza=postgres backup --type=diff
```

Confirm the crontab saved:

```bash
sudo -u postgres crontab -l
```

### Backup type explanation

| Type | What it backs up | Depends on |
|------|-----------------|------------|
| `full` | Everything — complete standalone backup | Nothing |
| `diff` | Changes since the last full backup | Last full backup |
| `incr` | Changes since the last backup of any type | Last full, diff, or incr |

---

## 10. Restore Operations

> ⚠️ **Always stop Patroni on ALL nodes before restoring.**
> Restoring to a running cluster causes split-brain and data conflicts.

---

### 10.1 Restore to Latest Backup

This procedure restores the most recent backup to one node, then brings the cluster back up.
The restored node becomes the new primary and the other two nodes sync from it.

#### Step 1 — Stop Patroni on ALL 3 DB nodes

```bash
# Run on pgnode1
sudo systemctl stop patroni

# Run on pgnode2
sudo systemctl stop patroni

# Run on pgnode3
sudo systemctl stop patroni
```

#### Step 2 — Remove all data from ALL 3 DB nodes

```bash
# Run on pgnode1, pgnode2, and pgnode3
sudo rm -rf /data/pgsql/16/data/*
```

#### Step 3 — Restore the latest backup on ONE node (e.g. pgnode1)

```bash
# Run on pgnode1 only
sudo -u postgres pgbackrest --stanza=postgres restore
```

#### Step 4 — Fix ownership and permissions (if restore complains about permissions)

```bash
sudo chown -R postgres:postgres /data/pgsql/16/data
sudo chmod 700 /data/pgsql/16/data
```

#### Step 5 — Start PostgreSQL directly to complete recovery

pgBackRest writes a `recovery.conf` / `standby.signal`. PostgreSQL must start once to replay
WAL and reach a consistent state before Patroni takes over.

```bash
# Start PostgreSQL directly (NOT via Patroni)
sudo -u postgres /usr/pgsql-16/bin/pg_ctl \
    -D /data/pgsql/16/data \
    -l /var/lib/pgsql/pg_restore.log \
    start
```

Wait for PostgreSQL to finish replaying WAL. Monitor recovery:

```bash
# Watch the restore log
tail -f /var/lib/pgsql/pg_restore.log

# Check if recovery is complete
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# Returns 't' while replaying WAL
# Returns 'f' when promoted and ready
```

#### Step 6 — Stop PostgreSQL cleanly before handing over to Patroni

```bash
sudo -u postgres /usr/pgsql-16/bin/pg_ctl \
    -D /data/pgsql/16/data \
    stop
```

#### Step 7 — Clear stale etcd cluster state

This removes the old Patroni DCS state from etcd so the cluster initialises cleanly.
Without this step, Patroni may refuse to start because it sees a stale leader lock.

```bash
# Set your etcd endpoints first (adjust IPs to match your etcd cluster)
export ENDPOINTS="http://192.168.109.133:2379,http://192.168.109.134:2379,http://192.168.109.135:2379"

etcdctl del /db/ --prefix --endpoints=$ENDPOINTS
```

> **Note:** Replace `/db/` with your actual Patroni namespace.
> Find it in your `patroni.yml` under `etcd.namespace` or `scope`.
> Example: if `scope: postgres`, the key prefix is `/postgres/`.

#### Step 8 — Resume Patroni if it is paused (if needed)

If the cluster was previously paused (e.g. during maintenance), resume it:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml resume postgres
```

#### Step 9 — Start Patroni on ALL 3 nodes

Start the restored node first — it will become the new primary.
Then start the other two — they will sync as replicas.

```bash
# Start pgnode1 first (the restored node)
sudo systemctl start patroni

# Then start pgnode2
sudo systemctl start patroni

# Then start pgnode3
sudo systemctl start patroni
```

#### Step 10 — Confirm cluster is healthy

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres
```

Expected — all 3 nodes running, one Leader:

```
+ Cluster: postgres --------+----+---------+-----------+
| Member   | Host                 | Role    | State   | TL | Lag in MB |
+----------+----------------------+---------+---------+----+-----------+
| pgnode1  | 192.168.109.133:5432 | Leader  | running |  2 |           |
| pgnode2  | 192.168.109.134:5432 | Replica | running |  2 |         0 |
| pgnode3  | 192.168.109.135:5432 | Replica | running |  2 |         0 |
```

---

### 10.2 Restore to a Specific Point in Time (PITR)

Use this to recover to the exact moment before a `DROP TABLE`, bad batch job,
or data corruption event.

#### Step 1 — Stop Patroni on ALL 3 DB nodes

```bash
# Run on pgnode1
sudo systemctl stop patroni

# Run on pgnode2
sudo systemctl stop patroni

# Run on pgnode3
sudo systemctl stop patroni
```

#### Step 2 — Remove all data from ALL 3 DB nodes

```bash
# Run on pgnode1, pgnode2, and pgnode3
sudo rm -rf /data/pgsql/16/data/*
```

#### Step 3 — Restore to a specific timestamp on ONE node (e.g. pgnode1)

Replace the timestamp with the exact moment just before the incident.

```bash
# Run on pgnode1 only
sudo -u postgres pgbackrest --stanza=postgres restore \
    --type=time \
    --target="2026-04-01 17:51:53" \
    --target-action=promote
```

> **How to find the right timestamp:**
> Use `sudo -u postgres pgbackrest --stanza=postgres info` to see the WAL archive range,
> then pick a timestamp that falls within that range and is just before the incident.

#### Step 4 — Start PostgreSQL directly to replay WAL to the target time

```bash
sudo -u postgres /usr/pgsql-16/bin/pg_ctl \
    -D /data/pgsql/16/data \
    -l /var/lib/pgsql/pg_restore.log \
    start
```

Monitor recovery progress:

```bash
# Watch the restore log
tail -f /var/lib/pgsql/pg_restore.log

# Check when recovery finishes
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# Returns 't' while replaying WAL toward the target time
# Returns 'f' when it reaches the target and promotes itself
```

#### Step 5 — Verify your data is correct before continuing

```bash
sudo -u postgres psql -c "SELECT COUNT(*) FROM your_important_table;"
# Confirm the data looks correct at the recovery point
```

#### Step 6 — Stop PostgreSQL cleanly

```bash
sudo -u postgres /usr/pgsql-16/bin/pg_ctl \
    -D /data/pgsql/16/data \
    stop
```

#### Step 7 — Clear stale etcd cluster state

```bash
export ENDPOINTS="http://192.168.109.133:2379,http://192.168.109.134:2379,http://192.168.109.135:2379"

etcdctl del /db/ --prefix --endpoints=$ENDPOINTS
```

#### Step 8 — Resume Patroni if it is paused

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml resume postgres
```

#### Step 9 — Start Patroni on ALL 3 nodes

```bash
# Start pgnode1 first (the restored node — becomes primary)
sudo systemctl start patroni

# Then pgnode2 and pgnode3 (become replicas)
sudo systemctl start patroni
sudo systemctl start patroni
```

#### Step 10 — Confirm cluster is healthy

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres
```

---

### 10.3 Manually Rebuild a Failed Replica

Use when one replica is down or stale and needs to rejoin the cluster.
The primary keeps running — no cluster downtime required.

```bash
# Step 1 — Stop Patroni on the failed replica only
sudo systemctl stop patroni

# Step 2 — Clear the stale data directory
sudo rm -rf /data/pgsql/16/data/*

# Step 3 — Restore from repo using delta mode
sudo -u postgres pgbackrest --stanza=postgres \
  --delta \
  --type=standby \
  --target-timeline=latest \
  restore

# Step 4 — Start Patroni — it configures streaming replication automatically
sudo systemctl start patroni

# Step 5 — Verify the replica joined the cluster
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres
```

---

## 11. Verify & Monitor

### Check backup status and WAL archive range

```bash
# Run on repo
sudo -u postgres pgbackrest --stanza=postgres info

# JSON output for monitoring tools
sudo -u postgres pgbackrest --stanza=postgres info --output=json | python3 -m json.tool
```

### Verify archiving is working right now

```bash
# Force a WAL segment switch on the primary
sudo -u postgres psql -c "SELECT pg_switch_wal();"

# Confirm the segment arrived at repo
sudo -u postgres pgbackrest --stanza=postgres check
```

### Check PostgreSQL archiver statistics

```bash
sudo -u postgres psql \
  -x -c "SELECT * FROM pg_stat_archiver;"
```

| Field | Healthy value |
|-------|---------------|
| `archived_count` | Increasing over time |
| `failed_count` | 0 (not growing) |
| `last_archived_wal` | A recent WAL segment |
| `last_failed_wal` | NULL |
| `last_failed_time` | NULL |

### View logs

```bash
# On repo — backup log
sudo tail -f /var/log/pgbackrest/postgres-backup.log

# On current primary — WAL archive push log
sudo tail -f /var/log/pgbackrest/postgres-archive-push.log

# On any DB node during recovery — WAL archive get log
sudo tail -f /var/log/pgbackrest/postgres-archive-get.log

# PostgreSQL archive errors via journald (Rocky Linux)
sudo journalctl -u postgresql-16 --since "1 hour ago" | grep -i archive

# Patroni logs
sudo journalctl -u patroni -n 100 --no-pager
```

---

## 12. Failover Behavior

### What happens during a Patroni failover?

```
Scenario: pgnode1 (primary) becomes unavailable.

1.  etcd detects pgnode1's lease has expired.
2.  Patroni elects pgnode2 as the new primary.
3.  pgnode2 starts accepting writes.
4.  pgnode2 runs: pgbackrest --stanza=postgres archive-push %p
5.  pgBackRest SSHes as postgres to 192.168.109.128 (repo).
6.  WAL lands in /var/lib/pgbackrest/archive/postgres/ on repo.
7.  Archiving continues without interruption.
    The stanza name, repo IP, and OS user never change.

When pgnode1 recovers:
8.  Patroni detects pgnode1 is back but stale.
9.  Patroni calls:
    pgbackrest --stanza=postgres --delta --type=standby
    --target-timeline=latest restore
10. pgnode1 pulls data from postgres@192.168.109.128.
11. pgnode1 starts streaming replication from pgnode2 (new primary).
12. Cluster is fully healthy with 3 members again.
```

### Timeline tracking after failover

After a failover, PostgreSQL increments the WAL timeline. pgBackRest tracks this automatically:

```
wal archive min/max (16): 000000010000000000000001/000000020000000000000008
                                   ↑ Timeline 1          ↑ Timeline 2
```

PITR across timelines works automatically with `--target-timeline=latest`.

---

## 13. Troubleshooting

### ❌ `stanza-create` fails immediately

**Cause:** Almost always an SSH connectivity issue.

```bash
# Re-test all SSH connections from repo
sudo -u postgres ssh postgres@192.168.109.133 "hostname"
sudo -u postgres ssh postgres@192.168.109.134 "hostname"
sudo -u postgres ssh postgres@192.168.109.135 "hostname"

# Re-test from a DB node to repo
sudo -u postgres ssh postgres@192.168.109.128 "hostname"

# Check authorized_keys on repo
cat /var/lib/pgsql/.ssh/authorized_keys

# Check SSH key permissions on any failing node
ls -la /var/lib/pgsql/.ssh/
# drwx------  .ssh/              (700)
# -rw-------  authorized_keys    (600)
# -rw-------  id_rsa             (600)
# -rw-r--r--  id_rsa.pub         (644)

# Check auth log on the server refusing the connection
sudo tail -30 /var/log/secure
# or via journald
sudo journalctl -u sshd -n 50 --no-pager
```

---

### ❌ `WAL segment not successfully archived` after check

```bash
# Confirm archive settings are active
sudo -u postgres psql \
  -c "SELECT name, setting FROM pg_settings \
      WHERE name IN ('archive_mode','archive_command','wal_level');"

# Check for pending_restart
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres

# If pending_restart shown, restart nodes
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode3
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode1

# Check PostgreSQL archiver for the actual error
sudo -u postgres psql \
  -x -c "SELECT * FROM pg_stat_archiver;"

# Test archive manually with debug output (run on the primary DB node)
sudo -u postgres pgbackrest --stanza=postgres \
  --log-level-console=debug \
  archive-push /data/pgsql/16/data/pg_wal/000000010000000000000001
```

---

### ❌ SELinux blocking pgBackRest (Rocky Linux specific)

```bash
# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep pgbackrest
sudo tail -30 /var/log/audit/audit.log | grep denied

# Check current SELinux mode
getenforce

# Temporarily set permissive to confirm SELinux is the cause
sudo setenforce 0
# Retry the failing pgBackRest command

# Permanent fix: generate and install a custom policy
sudo audit2allow -M pgbackrest_policy < /var/log/audit/audit.log
sudo semodule -i pgbackrest_policy.pp

# Re-enable enforcing
sudo setenforce 1

# Re-apply correct file contexts
sudo restorecon -Rv /var/lib/pgbackrest
sudo restorecon -Rv /var/log/pgbackrest
sudo restorecon -Rv /var/lib/pgsql/.ssh
```

---

### ❌ `unable to find primary cluster` during backup

```bash
# Check Patroni — find the current primary
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres

# Can repo SSH into the primary?
sudo -u postgres ssh postgres@192.168.109.133 \
  "psql -c 'SELECT pg_is_in_recovery();'"
# Must return 'f' on the primary

# Check pg paths in repo config
sudo grep pg.-path /etc/pgbackrest/pgbackrest.conf
# All 3 must show: /data/pgsql/16/data/
```

---

### ❌ `patronictl edit-config` changes not applying

```bash
# Verify changes were saved to etcd
sudo -u postgres patronictl -c /etc/patroni/patroni.yml show-config postgres

# Force reload on all nodes
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode1
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode3

# If archive_mode is still off, a restart is required
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode3
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode1
```

---

### Enable maximum debug logging for any command

```bash
sudo -u postgres pgbackrest --stanza=postgres \
  --log-level-console=debug \
  --log-level-file=trace \
  backup --type=full
```

---

## 14. Quick Reference

### Which node runs what

| Task | Run on |
|------|--------|
| Install pgBackRest | All DB nodes + repo |
| Create repo directory | repo only |
| Create log directory | All DB nodes + repo |
| SSH key generation | All DB nodes + repo |
| pgbackrest.conf (client) | All DB nodes (same content) |
| pgbackrest.conf (repo) | repo only (different content) |
| `edit-config` for WAL archiving | Any one DB node (applies cluster-wide via etcd) |
| `stanza-create` | repo only |
| `check` | repo only |
| `backup` | repo only |
| Cron schedule | repo only |
| `restore` | repo only |

---

### Key commands

```bash
# Check backup status and WAL archive range
sudo -u postgres pgbackrest --stanza=postgres info

# Verify archiving is working
sudo -u postgres pgbackrest --stanza=postgres check

# Full backup
sudo -u postgres pgbackrest --stanza=postgres backup --type=full

# Incremental backup
sudo -u postgres pgbackrest --stanza=postgres backup --type=incr

# Differential backup
sudo -u postgres pgbackrest --stanza=postgres backup --type=diff

# Restore latest backup
sudo -u postgres pgbackrest --stanza=postgres restore

# Restore to a specific point in time
sudo -u postgres pgbackrest --stanza=postgres restore \
    --type=time \
    --target="YYYY-MM-DD HH:MM:SS" \
    --target-action=promote

# List backup details as JSON
sudo -u postgres pgbackrest --stanza=postgres info --output=json

# View cluster status
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list postgres

# Edit live cluster config (WAL archiving, parameters)
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres

# Show live cluster config
sudo -u postgres patronictl -c /etc/patroni/patroni.yml show-config postgres

# Reload a node (no restart)
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode1

# Restart a node safely via Patroni
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode2
```

---

### Config file locations

| File | Location | Server |
|------|----------|--------|
| pgBackRest config | `/etc/pgbackrest/pgbackrest.conf` | All 4 servers (different content) |
| Backup repository | `/var/lib/pgbackrest/` | repo only |
| pgBackRest logs | `/var/log/pgbackrest/` | All 4 servers |
| postgres SSH keys | `/var/lib/pgsql/.ssh/` | All 4 servers |
| Patroni config | `/etc/patroni/patroni.yml` | DB nodes only |
| System auth log | `/var/log/secure` | All (Rocky Linux) |
| Audit log | `/var/log/audit/audit.log` | All (Rocky Linux) |
| PostgreSQL service | `postgresql-16` | DB nodes only |

---


