# MySQL InnoDB Cluster Setup on Rocky Linux
## Master-Slave Replication | 3-Node Cluster | Failover | Backup & Restore

---

## Table of Contents

1. [Environment Overview](#1-environment-overview)
2. [Prerequisites & System Preparation](#2-prerequisites--system-preparation)
3. [MySQL Installation on All Nodes](#3-mysql-installation-on-all-nodes)
4. [Master-Slave Replication Setup](#4-master-slave-replication-setup)
5. [InnoDB Cluster Setup (3 Nodes)](#5-innodb-cluster-setup-3-nodes)
6. [MySQL Router Setup](#6-mysql-router-setup)
7. [Failover Configuration & Testing](#7-failover-configuration--testing)
8. [Backup & Restore](#8-backup--restore)
9. [Monitoring & Cluster Status](#9-monitoring--cluster-status)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Environment Overview

```
+------------------+     +------------------+     +------------------+
|   Node 1 (Pri)   |     |   Node 2 (Sec)   |     |   Node 3 (Sec)   |
|  192.168.109.164   |<--->|  192.168.109.165   |<--->|  192.168.109.166   |
|  node1     |     |  node2     |     |  node3     |
+------------------+     +------------------+     +------------------+
         |                       |                        |
         +----------- MySQL Router (192.168.109.167) -------+
                         (mysql_router)
```

| Role | Hostname | IP Address |
|------|----------|------------|
| Primary (Master) | node1 | 192.168.109.164 |
| Secondary (Slave 1) | node2 | 192.168.109.165 |
| Secondary (Slave 2) | node3 | 192.168.109.166 |
| MySQL Router | mysql_router | 192.168.109.167 |

> **Note:** All nodes and MySQL Router have dedicated IPs in this setup.

---

## 2. Prerequisites & System Preparation

### 2.1 Run on ALL nodes

```bash
# Update the system
sudo dnf update -y

# Set hostnames (run on each respective node)
# On Node 1:
sudo hostnamectl set-hostname node1
# On Node 2:
sudo hostnamectl set-hostname node2
# On Node 3:
sudo hostnamectl set-hostname node3

# Add /etc/hosts entries on ALL nodes
sudo tee -a /etc/hosts <<EOF
192.168.109.164   node1
192.168.109.165   node2
192.168.109.166   node3
192.168.109.167   mysql_router
EOF
```

### 2.2 Disable SELinux (or configure for MySQL)

```bash
# Disable SELinux temporarily
sudo setenforce 0

# Disable permanently
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config

# OR - Keep SELinux enforcing and set correct context
sudo setsebool -P mysql_connect_any 1
```

### 2.3 Configure Firewall on ALL nodes

```bash
# Open MySQL and Group Replication ports
sudo firewall-cmd --permanent --add-port=3306/tcp   # MySQL
sudo firewall-cmd --permanent --add-port=33060/tcp  # MySQL X Protocol
sudo firewall-cmd --permanent --add-port=33061/tcp  # Group Replication
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

### 2.4 Configure NTP (Time Sync) — Required for Cluster

```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd

# Verify time sync
chronyc tracking
```

---

## 3. MySQL Installation on All Nodes

### 3.1 Install MySQL 8.0 Repository

```bash
# Download MySQL repo RPM
sudo dnf install -y https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

# Import GPG key
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023

# Enable MySQL 8.0 module
sudo dnf module disable -y mysql
sudo dnf config-manager --enable mysql80-community
```

### 3.2 Install MySQL Server & Shell

```bash
# Install MySQL Server, Shell, and Router
sudo dnf install -y mysql-community-server mysql-shell mysql-router

# Start and enable MySQL
sudo systemctl enable --now mysqld

# Check status
sudo systemctl status mysqld
```

### 3.3 Get Temporary Root Password

```bash
sudo grep 'temporary password' /var/log/mysqld.log
# Example output: A temporary password is generated for root@localhost: xxxxxxxxxxx
```

### 3.4 Secure MySQL Installation

```bash
sudo mysql_secure_installation
# - Enter temporary password
# - Set new strong root password
# - Remove anonymous users: Y
# - Disallow root login remotely: Y
# - Remove test database: Y
# - Reload privilege tables: Y
```

---

## 4. Master-Slave Replication Setup

> This section sets up traditional async Master-Slave before converting to InnoDB Cluster.

### 4.1 Ensure `/etc/my.cnf` loads config from `my.cnf.d/` — Run on ALL Nodes

By default on Rocky Linux, `/etc/my.cnf` may be missing the `!includedir` directive.
Without it, files in `/etc/my.cnf.d/` (including `replication.cnf`) are **silently ignored**.

```bash
# Check if includedir line exists
grep 'includedir' /etc/my.cnf

# If no output, add it (use this exact command to avoid bash ! expansion error)
sudo tee -a /etc/my.cnf << 'EOF'
!includedir /etc/my.cnf.d/
EOF

# Verify it was added
tail -3 /etc/my.cnf
```

> ⚠️ **Do this on node1, node2, and node3 before proceeding.**

---

### 4.2 Configure Master Node (Node 1)

```bash
sudo tee /etc/my.cnf.d/replication.cnf <<EOF
[mysqld]
# Server Identity
server-id                   = 1
report_host                 = node1

# Binary Logging
log_bin                     = /var/lib/mysql/mysql-bin
binlog_format               = ROW
binlog_row_image            = FULL
expire_logs_days            = 7
max_binlog_size             = 100M

# GTID (Global Transaction Identifiers) - Required for InnoDB Cluster
gtid_mode                   = ON
enforce_gtid_consistency    = ON

# Replication settings
log_replica_updates         = ON
replica_preserve_commit_order = ON

# InnoDB settings
innodb_buffer_pool_size     = 1G
innodb_flush_log_at_trx_commit = 1
sync_binlog                 = 1

# Performance Schema
performance_schema          = ON
EOF

sudo systemctl restart mysqld

# Verify GTID and server-id are active
mysql -u root -p -e "SHOW VARIABLES LIKE 'gtid_mode';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'server_id';"
# gtid_mode must show ON before proceeding
```

### 4.3 Configure Slave Nodes (Node 2 & Node 3)

```bash
# On Node 2 - set server-id=2
sudo tee /etc/my.cnf.d/replication.cnf <<EOF
[mysqld]
server-id                   = 2
report_host                 = node2
log_bin                     = /var/lib/mysql/mysql-bin
binlog_format               = ROW
binlog_row_image            = FULL
expire_logs_days            = 7
gtid_mode                   = ON
enforce_gtid_consistency    = ON
log_replica_updates         = ON
replica_preserve_commit_order = ON
read_only                   = ON
innodb_buffer_pool_size     = 1G
innodb_flush_log_at_trx_commit = 1
sync_binlog                 = 1
performance_schema          = ON
EOF

sudo systemctl restart mysqld

# Verify GTID and server-id
mysql -u root -p -e "SHOW VARIABLES LIKE 'gtid_mode';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'server_id';"
```

```bash
# On Node 3 - set server-id=3
sudo tee /etc/my.cnf.d/replication.cnf <<EOF
[mysqld]
server-id                   = 3
report_host                 = node3
log_bin                     = /var/lib/mysql/mysql-bin
binlog_format               = ROW
binlog_row_image            = FULL
expire_logs_days            = 7
gtid_mode                   = ON
enforce_gtid_consistency    = ON
log_replica_updates         = ON
replica_preserve_commit_order = ON
read_only                   = ON
innodb_buffer_pool_size     = 1G
innodb_flush_log_at_trx_commit = 1
sync_binlog                 = 1
performance_schema          = ON
EOF

sudo systemctl restart mysqld

# Verify GTID and server-id
mysql -u root -p -e "SHOW VARIABLES LIKE 'gtid_mode';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'server_id';"
```

### 4.4 Create Replication User on Master (Node 1)

```bash
# Connect to MySQL on Node 1
mysql -u root -p
```

```sql
-- Create replication user
CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'Repl@Pass123!';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;

-- Reset master binary logs for a clean GTID start
-- ⚠️ Only safe on a fresh setup with no production data
RESET MASTER;

-- Verify clean GTID state
SHOW MASTER STATUS\G
```

```
Example output:
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 157
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
```

> ⚠️ `RESET MASTER` clears all binary logs and GTID history. Only run this on a **fresh setup**. If node1 already has production data, skip this step.

### 4.5 Connect Slaves to Master

```bash
# Run on Node 2 and Node 3
mysql -u root -p
```

```sql
-- Reset any previous replication config
STOP REPLICA;
RESET REPLICA ALL;
RESET MASTER;

-- Configure replication using GTID auto-positioning
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='node1',
  SOURCE_PORT=3306,
  SOURCE_USER='replicator',
  SOURCE_PASSWORD='Repl@Pass123!',
  SOURCE_AUTO_POSITION=1;

START REPLICA;

-- Verify replication is running
SHOW REPLICA STATUS\G
```

**Check these fields in output:**
```
Replica_IO_Running: Yes       ✅
Replica_SQL_Running: Yes      ✅
Seconds_Behind_Source: 0      ✅
Last_IO_Error: (empty)        ✅
Last_SQL_Error: (empty)       ✅
```

> ⚠️ **Common Issue — GTID_MODE mismatch error:**
> If you see `Last_IO_Error: The replication receiver thread cannot start because the source has GTID_MODE = OFF`, it means node1's GTID is still OFF.
> Go back to node1, confirm `!includedir` is in `/etc/my.cnf`, restart mysqld, then run `SHOW VARIABLES LIKE 'gtid_mode'` to confirm it is ON before retrying.

> ⚠️ **Common Issue — Worker error 1410:**
> If `Replica_SQL_Running: No` with `Last_SQL_Errno: 1410`, it means node1 had transactions before GTID was enabled causing a GTID set mismatch.
> Fix: Run `RESET MASTER` on node1 first, then `STOP REPLICA; RESET REPLICA ALL; RESET MASTER;` on the slave, then redo the `CHANGE REPLICATION SOURCE TO` command.

### 4.6 Verify Replication is Syncing

**On node1 (master):**
```sql
CREATE DATABASE repl_test;
USE repl_test;
CREATE TABLE test (id INT, name VARCHAR(50));
INSERT INTO test VALUES (1, 'replication works');
```

**On node2 and node3 (slaves) — data should appear automatically:**
```sql
USE repl_test;
SELECT * FROM test;
```

**Expected output:**
```
+----+-------------------+
| id | name              |
+----+-------------------+
|  1 | replication works |
+----+-------------------+
```

**Clean up test data:**
```sql
-- Run on node1
DROP DATABASE repl_test;
```

---

## 5. InnoDB Cluster Setup (3 Nodes)

### 5.1 Stop Existing Master-Slave Replication on ALL Nodes

> ⚠️ **Required step.** InnoDB Cluster uses Group Replication internally and will refuse to configure instances that are part of an existing async replication topology. You must stop replication before proceeding.
> If you skip this step you will get:
> `Dba.configureInstance: This function is not available through a session to an instance belonging to an unmanaged asynchronous replication topology (RuntimeError)`

```sql
-- Run on node2
mysql -u root -p
STOP REPLICA;
RESET REPLICA ALL;
exit
```

```sql
-- Run on node3
mysql -u root -p
STOP REPLICA;
RESET REPLICA ALL;
exit
```

```sql
-- Run on node1
mysql -u root -p
RESET MASTER;
exit
```

### 5.2 Configure Instances for InnoDB Cluster

> ⚠️ **Important — root remote login is disabled by default on Rocky Linux.**
> Always use `127.0.0.1` when connecting root in MySQL Shell on the same node.
> Using the hostname will give: `Access denied for user 'root'@'node1' (MySQL Error 1045)`

> ⚠️ **Important — `dba.configureInstance` must be run locally on each node.**
> You cannot configure node2 or node3 remotely from node1 using root.
> Open a separate terminal/SSH session for each node.

**On node1 — launch MySQL Shell:**

```bash
mysqlsh
```

```js
\connect root@127.0.0.1:3306

dba.configureInstance('root@127.0.0.1:3306', {
  clusterAdmin: 'clusteradmin',
  clusterAdminPassword: 'Cluster@Admin123!',
  restart: true
});
```

When prompted `Do you want to perform the required configuration changes? [y/n]:` → type **y**

Expected output:
```
Account clusteradmin@% was successfully created.
The instance 'node1:3306' was configured to be used in an InnoDB cluster.
```

**On node2 — open a new SSH terminal and run:**

```bash
mysqlsh
```

```js
\connect root@127.0.0.1:3306

dba.configureInstance('root@127.0.0.1:3306', {
  clusterAdmin: 'clusteradmin',
  clusterAdminPassword: 'Cluster@Admin123!',
  restart: true
});
```

**On node3 — open a new SSH terminal and run:**

```bash
mysqlsh
```

```js
\connect root@127.0.0.1:3306

dba.configureInstance('root@127.0.0.1:3306', {
  clusterAdmin: 'clusteradmin',
  clusterAdminPassword: 'Cluster@Admin123!',
  restart: true
});
```

### 5.3 Fix clusteradmin Authentication Plugin on ALL Nodes

> ⚠️ **Required step.** `dba.configureInstance` creates `clusteradmin` with `caching_sha2_password` by default. MySQL Shell remote connections will fail with `Access denied` until you change it to `mysql_native_password` on all nodes.
> Also ensure `clusteradmin` has full privileges — without `ALL PRIVILEGES`, even basic `CREATE DATABASE` will fail with `Access denied` when connecting through MySQL Router.

**On node1:**

```bash
mysql -u root -p
```

```sql
DROP USER 'clusteradmin'@'%';
CREATE USER 'clusteradmin'@'%' IDENTIFIED WITH mysql_native_password BY 'Cluster@Admin123!';
GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

-- Verify
SELECT user, host, plugin FROM mysql.user WHERE user='clusteradmin';
```

**On node2 and node3** — disable read_only first:

```sql
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;

DROP USER 'clusteradmin'@'%';
CREATE USER 'clusteradmin'@'%' IDENTIFIED WITH mysql_native_password BY 'Cluster@Admin123!';
GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;

-- Verify
SELECT user, host, plugin FROM mysql.user WHERE user='clusteradmin';
```

Expected on all nodes:
```
+--------------+------+-----------------------+
| user         | host | plugin                |
+--------------+------+-----------------------+
| clusteradmin | %    | mysql_native_password |
+--------------+------+-----------------------+
```

### 5.4 Clean Up Any Test Databases — ALL Nodes

> ⚠️ Group Replication requires all tables to have a **Primary Key**. Any table without a primary key will cause `checkInstanceConfiguration` to return `status: error`. Drop all test databases before proceeding.

**On node2 and node3:**

```sql
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;

-- Drop any test databases (adjust names as needed)
DROP DATABASE IF EXISTS repl_test;
DROP DATABASE IF EXISTS write_test256;

SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

**On node1:**

```sql
DROP DATABASE IF EXISTS repl_test;
DROP DATABASE IF EXISTS write_test256;
```

### 5.5 Verify Instance Configuration

Go back to **node1** MySQL Shell:

```bash
mysqlsh
```

```js
\connect clusteradmin@127.0.0.1:3306

dba.checkInstanceConfiguration('clusteradmin@node1:3306');
dba.checkInstanceConfiguration('clusteradmin@node2:3306');
dba.checkInstanceConfiguration('clusteradmin@node3:3306');
```

All three must return:
```json
{
    "status": "ok"
}
```

> ⚠️ **Common errors:**
> - `Access denied for user 'clusteradmin'@'node1'` → auth plugin not fixed yet, redo section 5.3
> - `Tables do not have a Primary Key` → drop the offending database, redo section 5.4
> - `Access denied for user 'clusteradmin'@'localhost'` → clusteradmin password mismatch, redo DROP/CREATE in 5.3

### 5.6 Create the InnoDB Cluster

Still in MySQL Shell on node1 (connected as clusteradmin):

```js
var cluster = dba.createCluster('MyCluster', {
  gtidSetIsComplete: true,
  multiPrimary: false
});
```

When prompted type **y** for any configuration changes.

### 5.7 Add Secondary Nodes to Cluster

```js
// Add Node 2
cluster.addInstance('clusteradmin@node2:3306', {
  recoveryMethod: 'clone'
});

// Add Node 3
cluster.addInstance('clusteradmin@node3:3306', {
  recoveryMethod: 'clone'
});

// Verify full cluster status
cluster.status();
```

**Expected Output:**
```json
{
    "clusterName": "MyCluster",
    "defaultReplicaSet": {
        "name": "default",
        "primary": "node1:3306",
        "ssl": "REQUIRED",
        "status": "OK",
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.",
        "topology": {
            "node1:3306": {
                "memberRole": "PRIMARY",
                "mode": "R/W",
                "status": "ONLINE",
                "version": "8.0.46"
            },
            "node2:3306": {
                "memberRole": "SECONDARY",
                "mode": "R/O",
                "status": "ONLINE",
                "version": "8.0.46"
            },
            "node3:3306": {
                "memberRole": "SECONDARY",
                "mode": "R/O",
                "status": "ONLINE",
                "version": "8.0.46"
            }
        },
        "topologyMode": "Single-Primary"
    },
    "groupInformationSourceMember": "node1:3306"
}
```

```
node1:3306  →  PRIMARY    R/W  ONLINE  ✅
node2:3306  →  SECONDARY  R/O  ONLINE  ✅
node3:3306  →  SECONDARY  R/O  ONLINE  ✅
Status: Cluster is ONLINE and can tolerate up to ONE failure. ✅
```

---

## 6. MySQL Router Setup

### 6.1 Install MySQL Router on Router Node

```bash
# On mysql_router server
sudo dnf install -y mysql-router

# Bootstrap Router with Cluster
mysqlrouter --bootstrap clusteradmin@node1:3306 \
  --directory /etc/mysqlrouter \
  --conf-use-sockets \
  --user mysqlrouter \
  --force
```

### 6.2 Start MySQL Router

```bash
sudo systemctl enable --now mysqlrouter
sudo systemctl status mysqlrouter
```

### 6.3 Router Port Summary

| Port | Purpose |
|------|---------|
| 6446 | Read/Write (Primary) |
| 6447 | Read-Only (Secondary) |
| 6448 | Read/Write (X Protocol) |
| 6449 | Read-Only (X Protocol) |

### 6.4 Grant clusteradmin Full Privileges on Primary

> ⚠️ Before testing, ensure `clusteradmin` has full privileges on the primary node. Without this, `CREATE DATABASE` will fail with `Access denied` even through the router.

```bash
# On node1
mysql -u root -p
```

```sql
GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

### 6.5 Test Router — INSERT, UPDATE, DELETE

**Connect via R/W port (6446) on mysql_router:**

```bash
mysql -u clusteradmin -p'Cluster@Admin123!' -h 127.0.0.1 -P 6446
```

```sql
-- Setup
CREATE DATABASE cluster_test;
USE cluster_test;
CREATE TABLE users (id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(50));

-- INSERT
INSERT INTO users (name) VALUES ('Alice'), ('Bob'), ('Charlie');
SELECT * FROM users;

-- UPDATE
UPDATE users SET name='Alice Updated' WHERE id=1;
SELECT * FROM users;

-- DELETE
DELETE FROM users WHERE id=3;
SELECT * FROM users;
```

**Expected output after all operations:**
```
+----+---------------+
| id | name          |
+----+---------------+
|  1 | Alice Updated |
|  2 | Bob           |
+----+---------------+
```

**Verify replication via R/O port (6447):**

```bash
mysql -u clusteradmin -p'Cluster@Admin123!' -h 127.0.0.1 -P 6447
```

```sql
USE cluster_test;
SELECT * FROM users;
-- Must show identical data as primary ✅
```

**Verify which node each port connects to:**

```bash
# Should return node1 (Primary)
mysql -u clusteradmin -p'Cluster@Admin123!' -h 127.0.0.1 -P 6446 -e "SELECT @@hostname;"

# Should return node2 or node3 (Secondary)
mysql -u clusteradmin -p'Cluster@Admin123!' -h 127.0.0.1 -P 6447 -e "SELECT @@hostname;"
```

---

## 7. Failover Configuration & Testing

### 7.1 View Cluster Topology

```bash
mysqlsh clusteradmin@node1:3306 -- cluster status
```

### 7.2 Simulate Primary Node Failure

```bash
# Stop MySQL on Node 1 to simulate failure
sudo systemctl stop mysqld    # Run on node1

# On another node, check cluster status
mysqlsh clusteradmin@node2:3306

var cluster = dba.getCluster('MyCluster');
cluster.status();
# A new primary should be elected automatically from Node 2 or Node 3
```

### 7.3 Manual Failover (Forced)

```bash
# If automatic failover doesn't occur, force it
mysqlsh clusteradmin@node2:3306

var cluster = dba.getCluster('MyCluster');
cluster.forceQuorumUsingPartitionOf('clusteradmin@node2:3306');
```

### 7.4 Rejoin Failed Node

```bash
# After restoring node1, rejoin it to cluster
mysqlsh clusteradmin@node2:3306

var cluster = dba.getCluster('MyCluster');
cluster.rejoinInstance('clusteradmin@node1:3306');

# Verify
cluster.status();
```

### 7.5 Switch Primary Manually (Planned Switchover)

```bash
mysqlsh clusteradmin@node1:3306

var cluster = dba.getCluster('MyCluster');

# Switch primary to Node 2
cluster.setPrimaryInstance('clusteradmin@node2:3306');

# Verify new primary
cluster.status();
```

### 7.6 Configure Auto-Rejoin (Persistent)

```bash
# Set auto-rejoin retries on the cluster
mysqlsh clusteradmin@node1:3306

var cluster = dba.getCluster('MyCluster');
cluster.setOption('autoRejoinTries', 3);
cluster.setOption('expelTimeout', 10);
```

---

## 8. Backup & Restore

### 8.1 Install MySQL Enterprise Backup (or use mysqldump / mysqlpump)

```bash
# Using mysqldump (available by default)
# Using mysqlbackup requires MySQL Enterprise Edition

# Install Percona XtraBackup as alternative (free)
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release enable-only tools
sudo dnf install -y percona-xtrabackup-80
```

### 8.2 Full Backup using mysqldump

```bash
# Full logical backup (all databases)
mysqldump \
  -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  --triggers \
  --routines \
  --events \
  --set-gtid-purged=ON \
  --compress \
  > /backup/full_backup_$(date +%Y%m%d_%H%M%S).sql

# Backup specific database
mysqldump \
  -u root -p \
  --single-transaction \
  --triggers \
  --routines \
  mydb \
  > /backup/mydb_$(date +%Y%m%d_%H%M%S).sql
```

### 8.3 Full Backup using Percona XtraBackup (Recommended for Large DBs)

```bash
# Create backup directory
sudo mkdir -p /backup/full /backup/incremental

# Full backup
sudo xtrabackup \
  --backup \
  --user=root \
  --password='YourPassword' \
  --target-dir=/backup/full/$(date +%Y%m%d) \
  --host=127.0.0.1 \
  --port=3306

# Prepare (apply logs to backup)
sudo xtrabackup \
  --prepare \
  --target-dir=/backup/full/$(date +%Y%m%d)
```

### 8.4 Incremental Backup

```bash
# First run full backup (above), then incremental:
sudo xtrabackup \
  --backup \
  --user=root \
  --password='YourPassword' \
  --target-dir=/backup/incremental/$(date +%Y%m%d_%H%M%S) \
  --incremental-basedir=/backup/full/20240101 \
  --host=127.0.0.1

# Apply incremental to full backup
sudo xtrabackup \
  --prepare \
  --apply-log-only \
  --target-dir=/backup/full/20240101 \
  --incremental-dir=/backup/incremental/20240102_120000
```

### 8.5 Automated Backup Script

```bash
sudo tee /usr/local/bin/mysql-backup.sh <<'EOF'
#!/bin/bash
# MySQL Automated Backup Script

BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7
MYSQL_USER="root"
MYSQL_PASS="YourPassword"
LOG_FILE="/var/log/mysql-backup.log"

mkdir -p "${BACKUP_DIR}"

echo "[${DATE}] Starting MySQL backup..." >> "${LOG_FILE}"

# Full backup with XtraBackup
xtrabackup \
  --backup \
  --user="${MYSQL_USER}" \
  --password="${MYSQL_PASS}" \
  --target-dir="${BACKUP_DIR}/full_${DATE}" \
  --host=127.0.0.1 >> "${LOG_FILE}" 2>&1

if [ $? -eq 0 ]; then
  echo "[${DATE}] Backup successful: ${BACKUP_DIR}/full_${DATE}" >> "${LOG_FILE}"
  # Compress backup
  tar -czf "${BACKUP_DIR}/full_${DATE}.tar.gz" -C "${BACKUP_DIR}" "full_${DATE}"
  rm -rf "${BACKUP_DIR}/full_${DATE}"
else
  echo "[${DATE}] Backup FAILED!" >> "${LOG_FILE}"
  exit 1
fi

# Remove old backups
find "${BACKUP_DIR}" -name "full_*.tar.gz" -mtime +${RETENTION_DAYS} -delete
echo "[${DATE}] Old backups cleaned (older than ${RETENTION_DAYS} days)" >> "${LOG_FILE}"

EOF

sudo chmod +x /usr/local/bin/mysql-backup.sh
```

### 8.6 Schedule Backup with Cron

```bash
sudo crontab -e

# Add these lines:
# Full backup daily at 2:00 AM
0 2 * * * /usr/local/bin/mysql-backup.sh

# Verify cron is running
sudo systemctl status crond
```

### 8.7 Restore from mysqldump Backup

```bash
# Stop cluster, restore on primary
mysql -u root -p < /backup/full_backup_20240101_020000.sql

# Or restore specific database
mysql -u root -p mydb < /backup/mydb_20240101_020000.sql

# After restore, reset GTID (if needed)
mysql -u root -p -e "RESET MASTER;"
```

### 8.8 Restore from XtraBackup

```bash
# Step 1: Stop MySQL
sudo systemctl stop mysqld

# Step 2: Clear data directory (CAUTION!)
sudo rm -rf /var/lib/mysql/*

# Step 3: Prepare the backup
sudo xtrabackup --prepare --target-dir=/backup/full/20240101

# Step 4: Restore
sudo xtrabackup \
  --copy-back \
  --target-dir=/backup/full/20240101

# Step 5: Fix permissions
sudo chown -R mysql:mysql /var/lib/mysql

# Step 6: Start MySQL
sudo systemctl start mysqld
```

### 8.9 Point-in-Time Recovery (PITR)

```bash
# Restore base backup first, then apply binary logs

# Find the binary logs after backup
mysqlbinlog \
  --start-datetime="2024-01-01 03:00:00" \
  --stop-datetime="2024-01-01 08:00:00" \
  /var/lib/mysql/mysql-bin.000001 \
  /var/lib/mysql/mysql-bin.000002 \
  | mysql -u root -p
```

---

## 9. Monitoring & Cluster Status

### 9.1 Check Cluster Status

```bash
# Using MySQL Shell
mysqlsh clusteradmin@node1:3306 -- cluster status

# Detailed status
mysqlsh clusteradmin@node1:3306 -- cluster status --extended=1
```

### 9.2 Check Replication Status via SQL

```bash
mysql -u root -p -e "SHOW REPLICA STATUS\G"
mysql -u root -p -e "SHOW MASTER STATUS\G"
```

### 9.3 Monitor Group Replication

```sql
-- Connect to any node and run:
SELECT * FROM performance_schema.replication_group_members;
SELECT * FROM performance_schema.replication_group_member_stats;
```

### 9.4 Useful Monitoring Queries

```sql
-- Check all replication channels
SELECT
  CHANNEL_NAME,
  SERVICE_STATE,
  LAST_ERROR_MESSAGE,
  LAST_QUEUED_TRANSACTION
FROM performance_schema.replication_connection_status;

-- Check slave lag
SELECT
  MEMBER_ID,
  MEMBER_HOST,
  MEMBER_STATE,
  MEMBER_ROLE
FROM performance_schema.replication_group_members;

-- Check InnoDB metrics
SELECT * FROM information_schema.INNODB_METRICS
WHERE name LIKE 'group_replication%';
```

### 9.5 Cluster Health Check Script

```bash
sudo tee /usr/local/bin/check-cluster.sh <<'EOF'
#!/bin/bash
echo "========================================="
echo "  MySQL InnoDB Cluster Health Check"
echo "  $(date)"
echo "========================================="

mysqlsh --no-wizard clusteradmin@node1:3306 \
  --execute "var c = dba.getCluster(); print(JSON.stringify(c.status(), null, 2));" 2>/dev/null

echo ""
echo "--- Node Status ---"
for NODE in node1 node2 node3; do
  STATUS=$(mysql -u clusteradmin -pCluster@Admin123! -h $NODE -e "SELECT @@hostname, @@read_only;" 2>/dev/null)
  echo "$NODE: $STATUS"
done
EOF

sudo chmod +x /usr/local/bin/check-cluster.sh
```

---

## 10. Troubleshooting

### 10.1 Common Issues

#### Node Not Joining Cluster

```bash
# Check error log
sudo tail -100 /var/log/mysqld.log | grep -i error

# Validate instance before joining
mysqlsh clusteradmin@node2:3306 -- dba checkInstanceConfiguration

# Re-configure instance
mysqlsh clusteradmin@node2:3306 -- dba configureInstance
```

#### Cluster Lost Quorum

```bash
# If majority of nodes are down, force quorum from surviving node
mysqlsh clusteradmin@node2:3306

var cluster = dba.getCluster();
cluster.forceQuorumUsingPartitionOf('clusteradmin@node2:3306');
```

#### Replication Lag Too High

```sql
-- Check lag on replicas
SHOW REPLICA STATUS\G

-- Check for long-running queries
SELECT * FROM information_schema.PROCESSLIST
WHERE TIME > 60 ORDER BY TIME DESC;

-- Tune parallel replication
SET GLOBAL replica_parallel_workers = 4;
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';
```

#### GTID Inconsistency

```bash
# Check GTID sets on all nodes
mysql -u root -p -e "SELECT @@global.gtid_executed;"

# Reset GTID on replica (use carefully!)
mysql -u root -p -e "STOP REPLICA; RESET REPLICA ALL; RESET MASTER;"
```

### 10.2 Key Log Files

| File | Purpose |
|------|---------|
| `/var/log/mysqld.log` | MySQL error log |
| `/var/lib/mysql/mysql-bin.*` | Binary logs |
| `/var/log/mysqlrouter/mysqlrouter.log` | Router log |
| `/var/log/mysql-backup.log` | Backup log (custom) |

### 10.3 Useful Commands Reference

```bash
# MySQL Shell shortcuts
mysqlsh clusteradmin@node1:3306 -- cluster status
mysqlsh clusteradmin@node1:3306 -- cluster describe
mysqlsh clusteradmin@node1:3306 -- cluster options

# Restart cluster (if all nodes restarted)
mysqlsh clusteradmin@node1:3306
dba.rebootClusterFromCompleteOutage();

# Remove node gracefully
var cluster = dba.getCluster();
cluster.removeInstance('clusteradmin@node3:3306');

# Dissolve cluster (DANGER!)
cluster.dissolve({force: true});
```

---

## Summary Checklist

- [ ] All nodes have unique `server-id` values
- [ ] GTID mode enabled on all nodes (`gtid_mode=ON`)
- [ ] Binary logging enabled on all nodes
- [ ] Firewall ports open (3306, 33060, 33061)
- [ ] NTP synchronized on all nodes
- [ ] `clusteradmin` user created on all nodes
- [ ] All instances pass `checkInstanceConfiguration`
- [ ] Cluster created and all 3 nodes show `ONLINE`
- [ ] MySQL Router bootstrapped and running
- [ ] Backup script scheduled via cron
- [ ] Failover tested successfully

---

*Generated for Rocky Linux 8/9 | MySQL 8.0 | InnoDB Cluster*
