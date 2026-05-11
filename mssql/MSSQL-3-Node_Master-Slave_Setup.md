# SQL Server Always On Availability Groups — Rocky Linux Setup Guide

> **3-Node Master-Slave Setup | SQL Server on Rocky Linux**

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Set Hostnames](#step-1--set-hostnames)
4. [Step 2 — Configure /etc/hosts](#step-2--configure-etchosts)
5. [Step 3 — Open Firewall Ports](#step-3--open-firewall-ports)
6. [Step 4 — Enable Always On AG](#step-4--enable-always-on-ag)
7. [Step 5 — Create Certificate on Primary](#step-5--create-certificate-on-primary)
8. [Step 6 — Copy Certificate to Secondaries](#step-6--copy-certificate-to-secondaries)
9. [Step 7 — Create Mirroring Endpoint](#step-7--create-mirroring-endpoint)
10. [Step 8 — Prepare the Database](#step-8--prepare-the-database)
11. [Step 9 — Create the Availability Group](#step-9--create-the-availability-group)
12. [Step 10 — Join Secondaries to the AG](#step-10--join-secondaries-to-the-ag)
13. [Step 11 — Verify the Setup](#step-11--verify-the-setup)
14. [Step 12 — Manual Failover](#step-12--manual-failover)
15. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

| Node   | Hostname | IP Address       | Role                          | Commit Mode          |
|--------|----------|------------------|-------------------------------|----------------------|
| node1  | node1    | 192.168.109.170  | **Primary (Master)**          | Synchronous          |
| node2  | node2    | 192.168.109.168  | **Secondary (Slave 1)**       | Synchronous          |
| node3  | node3    | 192.168.109.171  | **Secondary (Slave 2 / DR)**  | Asynchronous         |

```
<img width="1536" height="1024" alt="c1c9f794-66d4-420c-a9d0-3179e4eae7dc" src="https://github.com/user-attachments/assets/6fb62be8-fe52-4e56-915b-d96da968ac1b" />

```

> **Your VM IPs are already configured** in this guide: node1=`192.168.109.170`, node2=`192.168.109.168`, node3=`192.168.109.171`.

---

## Prerequisites

- SQL Server installed and running on all 3 VMs
- Rocky Linux (8 or 9) on all nodes
- Root or sudo access on all nodes
- All VMs can ping each other by IP
- Same SQL Server version on all nodes

---

## Step 1 — Set Hostnames

Run each command **on the respective node only**.

### 🖥️ Run on node1
```bash
sudo hostnamectl set-hostname node1
```

### 🖥️ Run on node2
```bash
sudo hostnamectl set-hostname node2
```

### 🖥️ Run on node3
```bash
sudo hostnamectl set-hostname node3
```

**Verify hostname (on each node):**
```bash
hostname
```

---

## Step 2 — Configure /etc/hosts

### 🖥️ Run on ALL 3 NODES (node1, node2, node3)

```bash
sudo nano /etc/hosts
```

Add the following lines at the bottom:

```
192.168.109.170  node1
192.168.109.168  node2
192.168.109.171  node3
```

**Verify connectivity from each node:**
```bash
ping -c 3 node1
ping -c 3 node2
ping -c 3 node3
```

All pings should succeed before proceeding.

---

## Step 3 — Open Firewall Ports

### 🖥️ Run on ALL 3 NODES (node1, node2, node3)

```bash
# SQL Server port
sudo firewall-cmd --permanent --add-port=1433/tcp

# AG mirroring endpoint port
sudo firewall-cmd --permanent --add-port=5022/tcp

# Reload firewall
sudo firewall-cmd --reload

# Verify ports are open
sudo firewall-cmd --list-ports
```

Expected output should include: `1433/tcp 5022/tcp`

---

## Step 4 — Enable Always On AG

### 🖥️ Run on ALL 3 NODES (node1, node2, node3)

```bash
# Enable the HADR feature
sudo /opt/mssql/bin/mssql-conf set hadr.hadrenabled 1

# Restart SQL Server to apply
sudo systemctl restart mssql-server

# Confirm SQL Server is running
sudo systemctl status mssql-server
```

---

## Step 5 — Create Certificate on Primary

### 🖥️ Run on node1 ONLY (Primary)

Connect to SQL Server on **node1**:

```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

Then run the following SQL commands:

```sql
-- Step 5a: Create a master key
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPass123!';
GO

-- Step 5b: Create the AG certificate
CREATE CERTIFICATE AG_Cert
WITH SUBJECT = 'AG Certificate',
EXPIRY_DATE = '2099-01-01';
GO

-- Step 5c: Backup certificate and private key to disk
BACKUP CERTIFICATE AG_Cert
TO FILE = '/var/opt/mssql/data/AG_Cert.cer'
WITH PRIVATE KEY (
    FILE = '/var/opt/mssql/data/AG_Cert.key',
    ENCRYPTION BY PASSWORD = 'CertPass123!'
);
GO
```

**Verify the certificate files were created:**
```bash
ls -la /var/opt/mssql/data/AG_Cert.*
```

---

## Step 6 — Copy Certificate to Secondaries

### 🖥️ Run on node1 ONLY (copy files FROM node1 TO node2 and node3)

```bash
# Copy to node2 (192.168.109.168)
scp /var/opt/mssql/data/AG_Cert.cer your_user@192.168.109.168:/var/opt/mssql/data/
scp /var/opt/mssql/data/AG_Cert.key your_user@192.168.109.168:/var/opt/mssql/data/

# Copy to node3 (192.168.109.171)
scp /var/opt/mssql/data/AG_Cert.cer your_user@192.168.109.171:/var/opt/mssql/data/
scp /var/opt/mssql/data/AG_Cert.key your_user@192.168.109.171:/var/opt/mssql/data/
```

### 🖥️ Run on node2 AND node3 (fix file ownership)

```bash
sudo chown mssql:mssql /var/opt/mssql/data/AG_Cert.cer
sudo chown mssql:mssql /var/opt/mssql/data/AG_Cert.key

# Verify ownership
ls -la /var/opt/mssql/data/AG_Cert.*
```

---

## Step 7 — Create Mirroring Endpoint

### 🖥️ Run on node1 ONLY

Connect to SQL Server:
```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

```sql
-- Create the mirroring endpoint on node1
CREATE ENDPOINT AG_Endpoint
STATE = STARTED
AS TCP (LISTENER_PORT = 5022)
FOR DATABASE_MIRRORING (
    ROLE = ALL,
    AUTHENTICATION = CERTIFICATE AG_Cert,
    ENCRYPTION = REQUIRED ALGORITHM AES
);
GO
```

---

### 🖥️ Run on node2 ONLY

Connect to SQL Server:
```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

```sql
-- Step 7a: Create master key on node2
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPass123!';
GO

-- Step 7b: Restore the certificate from copied files
CREATE CERTIFICATE AG_Cert
FROM FILE = '/var/opt/mssql/data/AG_Cert.cer'
WITH PRIVATE KEY (
    FILE = '/var/opt/mssql/data/AG_Cert.key',
    DECRYPTION BY PASSWORD = 'CertPass123!'
);
GO

-- Step 7c: Create the mirroring endpoint on node2
CREATE ENDPOINT AG_Endpoint
STATE = STARTED
AS TCP (LISTENER_PORT = 5022)
FOR DATABASE_MIRRORING (
    ROLE = ALL,
    AUTHENTICATION = CERTIFICATE AG_Cert,
    ENCRYPTION = REQUIRED ALGORITHM AES
);
GO
```

---

### 🖥️ Run on node3 ONLY

Connect to SQL Server:
```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

```sql
-- Step 7a: Create master key on node3
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPass123!';
GO

-- Step 7b: Restore the certificate from copied files
CREATE CERTIFICATE AG_Cert
FROM FILE = '/var/opt/mssql/data/AG_Cert.cer'
WITH PRIVATE KEY (
    FILE = '/var/opt/mssql/data/AG_Cert.key',
    DECRYPTION BY PASSWORD = 'CertPass123!'
);
GO

-- Step 7c: Create the mirroring endpoint on node3
CREATE ENDPOINT AG_Endpoint
STATE = STARTED
AS TCP (LISTENER_PORT = 5022)
FOR DATABASE_MIRRORING (
    ROLE = ALL,
    AUTHENTICATION = CERTIFICATE AG_Cert,
    ENCRYPTION = REQUIRED ALGORITHM AES
);
GO
```

**Verify endpoint is running (on each node):**
```sql
SELECT name, state_desc, role_desc
FROM sys.database_mirroring_endpoints;
GO
```

Expected state: `STARTED`

---

## Step 8 — Prepare the Database

### 🖥️ Run on node1 ONLY (Primary)

```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

```sql
-- Step 8a: Create a test database
CREATE DATABASE AGTestDB;
GO

-- Step 8b: Set recovery model to FULL (required for AG)
ALTER DATABASE AGTestDB SET RECOVERY FULL;
GO

-- Step 8c: Take a full database backup
BACKUP DATABASE AGTestDB
TO DISK = '/var/opt/mssql/data/AGTestDB.bak'
WITH FORMAT, INIT;
GO

-- Step 8d: Take a transaction log backup
BACKUP LOG AGTestDB
TO DISK = '/var/opt/mssql/data/AGTestDB_log.bak'
WITH FORMAT, INIT;
GO
```

> **Note:** Both backups are required before adding the database to an Availability Group.

---

## Step 9 — Create the Availability Group

### 🖥️ Run on node1 ONLY (Primary)

```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

```sql
CREATE AVAILABILITY GROUP [AG1]
WITH (
    CLUSTER_TYPE = NONE,        -- Use NONE for Linux without Pacemaker
    DB_FAILOVER  = ON,
    DTC_SUPPORT  = NONE
)
FOR DATABASE [AGTestDB]
REPLICA ON
    N'node1' WITH (
        ENDPOINT_URL          = N'TCP://node1:5022',
        AVAILABILITY_MODE     = SYNCHRONOUS_COMMIT,   -- Strong consistency
        FAILOVER_MODE         = MANUAL,
        SEEDING_MODE          = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
    ),
    N'node2' WITH (
        ENDPOINT_URL          = N'TCP://node2:5022',
        AVAILABILITY_MODE     = SYNCHRONOUS_COMMIT,   -- Strong consistency
        FAILOVER_MODE         = MANUAL,
        SEEDING_MODE          = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
    ),
    N'node3' WITH (
        ENDPOINT_URL          = N'TCP://node3:5022',
        AVAILABILITY_MODE     = ASYNCHRONOUS_COMMIT,  -- DR node (lower latency impact)
        FAILOVER_MODE         = MANUAL,
        SEEDING_MODE          = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
    );
GO

-- Grant AG permission to seed databases automatically
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
```

---

## Step 10 — Join Secondaries to the AG

### 🖥️ Run on node2 ONLY

```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

```sql
-- Join node2 to the Availability Group
ALTER AVAILABILITY GROUP [AG1] JOIN WITH (CLUSTER_TYPE = NONE);
GO

-- Allow automatic seeding on node2
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
```

---

### 🖥️ Run on node3 ONLY

```bash
sqlcmd -S localhost -U SA -P 'YourSAPassword'
```

```sql
-- Join node3 to the Availability Group
ALTER AVAILABILITY GROUP [AG1] JOIN WITH (CLUSTER_TYPE = NONE);
GO

-- Allow automatic seeding on node3
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
```

Wait a few minutes for the initial database seeding to complete before verifying.

---

## Step 11 — Verify the Setup

### 🖥️ Run on ANY node

```sql
-- Check replica roles and health
SELECT
    ag.name                         AS AG_Name,
    ar.replica_server_name          AS Replica,
    ars.role_desc                   AS Role,
    ars.synchronization_health_desc AS Sync_Health,
    ars.connected_state_desc        AS Connected
FROM sys.availability_groups              ag
JOIN sys.availability_replicas            ar  ON ag.group_id  = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
GO
```

**Expected output:**

| AG_Name | Replica | Role      | Sync_Health | Connected  |
|---------|---------|-----------|-------------|------------|
| AG1     | node1   | PRIMARY   | HEALTHY     | CONNECTED  |
| AG1     | node2   | SECONDARY | HEALTHY     | CONNECTED  |
| AG1     | node3   | SECONDARY | HEALTHY     | CONNECTED  |

```sql
-- Check database synchronization state
SELECT
    db.name                              AS Database_Name,
    drs.database_state_desc             AS DB_State,
    drs.synchronization_state_desc      AS Sync_State,
    drs.synchronization_health_desc     AS Sync_Health,
    drs.is_primary_replica              AS Is_Primary
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.databases db ON drs.database_id = db.database_id
WHERE db.name = 'AGTestDB';
GO
```

---

## Step 12 — Manual Failover

> ⚠️ Since `CLUSTER_TYPE = NONE`, all failovers are **manual**. There is no automatic failover without Pacemaker.

---

### Scenario A — Planned Failover (node1 is healthy, no data loss)

Use this when you want to gracefully switch primary (maintenance, upgrades etc).

#### 🖥️ Run on node1 (current Primary)

```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
ALTER AVAILABILITY GROUP [AG1] FAILOVER;
GO

SELECT
    ar.replica_server_name          AS Replica,
    ars.role_desc                   AS Role,
    ars.synchronization_health_desc AS Sync_Health,
    ars.connected_state_desc        AS Connected
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
GO
EOF
```

---

### Scenario B — Emergency Failover (node1 crashed/unreachable)

Use this when the primary is **down and not reachable**. Follow all steps in order.

---

#### 🖥️ STEP 1 — Run on node2 (promote to PRIMARY)

```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
ALTER AVAILABILITY GROUP [AG1] FORCE_FAILOVER_ALLOW_DATA_LOSS;
GO

SELECT
    ar.replica_server_name          AS Replica,
    ars.role_desc                   AS Role,
    ars.synchronization_health_desc AS Sync_Health,
    ars.connected_state_desc        AS Connected
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
GO
EOF
```

Expected output:
```
Replica   Role        Sync_Health   Connected
--------- ----------- ------------- ------------
node1     SECONDARY   NOT_HEALTHY   DISCONNECTED
node2     PRIMARY     HEALTHY       CONNECTED
node3     SECONDARY   NOT_HEALTHY   CONNECTED
```

---

#### 🖥️ STEP 2 — Fix node3 (re-sync with new primary node2)

After failover, node3 gets suspended automatically. Fix in order:

**First — run on node3**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
ALTER DATABASE AGTestDB SET HADR RESUME;
GO
EOF
```

**Second — run on node2**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
ALTER AVAILABILITY GROUP [AG1] REMOVE REPLICA ON N'node3';
GO

ALTER AVAILABILITY GROUP [AG1] ADD REPLICA ON N'node3' WITH (
    ENDPOINT_URL      = N'TCP://node3:5022',
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE     = MANUAL,
    SEEDING_MODE      = AUTOMATIC,
    SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
);
GO

ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
EOF
```

**Third — run on node3**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
DROP AVAILABILITY GROUP [AG1];
GO

ALTER AVAILABILITY GROUP [AG1] JOIN WITH (CLUSTER_TYPE = NONE);
GO

ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
EOF
```

> ⚠️ `DROP AVAILABILITY GROUP` on a secondary only removes local metadata — it does NOT affect node2 or the AG itself. Completely safe!

---

#### 🖥️ STEP 3 — Verify all nodes from node2

```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
SELECT
    ar.replica_server_name          AS Replica,
    ars.role_desc                   AS Role,
    ars.synchronization_health_desc AS Sync_Health,
    ars.connected_state_desc        AS Connected
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
GO
EOF
```

Expected output:
```
Replica   Role        Sync_Health   Connected
--------- ----------- ------------- -----------
node1     SECONDARY   NOT_HEALTHY   DISCONNECTED
node2     PRIMARY     HEALTHY       CONNECTED
node3     SECONDARY   HEALTHY       CONNECTED
```

---

#### 🖥️ STEP 4 — Bring node1 back online (when recovered)

```bash
sudo systemctl start mssql-server

sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
ALTER AVAILABILITY GROUP [AG1] JOIN WITH (CLUSTER_TYPE = NONE);
GO
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
EOF
```

---

#### ⚠️ Important Notes After Emergency Failover

| Action | Required? |
|---|---|
| Update app connection string to node2 IP `192.168.109.168` | ✅ Yes |
| `DROP AVAILABILITY GROUP` on node3 is safe | ✅ Yes — only removes local metadata |
| node1 data is safe when it comes back | ✅ Yes — it re-syncs from node2 |
| Run failover command twice | ❌ No — causes error (already primary) |

---


## Step 13 — Recover a Failed Secondary Back into AG

Use this when a secondary node (e.g. node1) comes back online after a crash but **cannot rejoin the AG** due to stuck metadata or old database files.

> ✅ Battle-tested procedure — works for both node1 and node3 recovery.

---

### 🖥️ PART A — Fix from the Primary node first

Remove and re-add the failed replica cleanly from the primary side.

**To recover node1 — run on current Primary:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
ALTER AVAILABILITY GROUP [AG1] REMOVE REPLICA ON N'node1';
GO

ALTER AVAILABILITY GROUP [AG1] ADD REPLICA ON N'node1' WITH (
    ENDPOINT_URL      = N'TCP://node1:5022',
    AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
    FAILOVER_MODE     = MANUAL,
    SEEDING_MODE      = AUTOMATIC,
    SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
);
GO

ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
EOF
```

**To recover node3 — run on current Primary:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
ALTER AVAILABILITY GROUP [AG1] REMOVE REPLICA ON N'node3';
GO

ALTER AVAILABILITY GROUP [AG1] ADD REPLICA ON N'node3' WITH (
    ENDPOINT_URL      = N'TCP://node3:5022',
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE     = MANUAL,
    SEEDING_MODE      = AUTOMATIC,
    SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
);
GO

ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
EOF
```

---

### 🖥️ PART B — Fix on the recovered node (node1 or node3)

Run this **immediately after** PART A on the node being recovered:

```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
-- Step 1: Drop old stuck AG metadata
DROP AVAILABILITY GROUP [AG1];
GO

-- Step 2: Drop old database files if database still exists
ALTER DATABASE AGTestDB SET OFFLINE WITH ROLLBACK IMMEDIATE;
GO
DROP DATABASE AGTestDB;
GO

-- Step 3: Rejoin the AG cleanly
ALTER AVAILABILITY GROUP [AG1] JOIN WITH (CLUSTER_TYPE = NONE);
GO

ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
GO
EOF
```

> ✅ `DROP AVAILABILITY GROUP` on a secondary only removes local metadata — does NOT affect the primary or AG.
> ✅ `DROP DATABASE` on secondary is safe — primary auto-seeds a fresh copy via `SEEDING_MODE = AUTOMATIC`.

---

### 🖥️ PART C — Verify from primary node after 1-2 minutes

```bash
sqlcmd -S localhost -U SA -P 'Sam98765.' -C <<EOF
SELECT
    ar.replica_server_name          AS Replica,
    ars.role_desc                   AS Role,
    ars.synchronization_health_desc AS Sync_Health,
    ars.connected_state_desc        AS Connected
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
GO
EOF
```

Expected output after full recovery:
```
Replica   Role        Sync_Health   Connected
--------- ----------- ------------- -----------
node1     SECONDARY   HEALTHY       CONNECTED   ✅
node2     PRIMARY     HEALTHY       CONNECTED   ✅
node3     SECONDARY   HEALTHY       CONNECTED   ✅
```

---

### Common Errors During Recovery and Fixes

| Error | Cause | Fix |
|---|---|---|
| `replica already exists` | Old AG metadata stuck on node | Run `DROP AVAILABILITY GROUP [AG1]` on that node first |
| `database is not joined to AG` | DB exists but AG was dropped | Run `JOIN` again — skip the RESUME command |
| `Seeding task failed 0x00000002` | Old database files conflict | Drop the database on secondary then rejoin |
| `SNI error 10022` | Endpoint in broken state | Run `ALTER ENDPOINT AG_Endpoint STATE = STOPPED` then `STARTED` |
| `SUSPEND_FROM_PARTNER` | Primary suspended data movement | Run `ALTER DATABASE AGTestDB SET HADR RESUME` on secondary |

---

## Troubleshooting

### Check SQL Server error log

```bash
# On any node
sudo cat /var/opt/mssql/log/errorlog | grep -i "error\|hadr\|ag1"
```

### Check endpoint status

```sql
SELECT name, state_desc, role_desc, connection_auth_desc
FROM sys.database_mirroring_endpoints;
GO
```

### Common Issues and Fixes

| Problem | Likely Cause | Fix |
|--------|--------------|-----|
| Endpoint won't start | Port 5022 blocked | Run `sudo firewall-cmd --list-ports` and re-open port 5022 |
| Nodes can't reach each other | `/etc/hosts` wrong | Verify entries with `ping node1` from each node |
| `NOT SYNCHRONIZING` on secondary | Certificate error or network | Check errorlog; verify `AG_Cert.*` ownership is `mssql:mssql` |
| Certificate error on restore | Wrong password or permissions | Re-check `CertPass123!` and run `chown mssql:mssql` again |
| AG stuck in `JOINING` | Endpoint not started | Run `ALTER ENDPOINT AG_Endpoint STATE = STARTED` on that node |
| Secondary DB in RESTORING state | Seeding still in progress | Wait 2–3 minutes; re-run the verify query |

### SELinux Issues (Rocky Linux specific)

If SELinux is enforcing and blocking SQL Server:
```bash
# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep mssql

# Allow SQL Server to bind to port 5022
sudo semanage port -a -t mssql_port_t -p tcp 5022

# Or temporarily set SELinux to permissive for testing
sudo setenforce 0
```

---

## Quick Reference — Which Command Runs Where

| Step | node1 (Primary) | node2 (Secondary) | node3 (Secondary) |
|------|:-----------:|:-----------:|:-----------:|
| Set hostname | ✅ | ✅ | ✅ |
| Edit /etc/hosts | ✅ | ✅ | ✅ |
| Open firewall ports | ✅ | ✅ | ✅ |
| Enable HADR | ✅ | ✅ | ✅ |
| Create master key | ✅ | ✅ | ✅ |
| Create certificate | ✅ only | ❌ | ❌ |
| Backup certificate | ✅ only | ❌ | ❌ |
| SCP copy cert files | ✅ (sends) | ✅ (receives) | ✅ (receives) |
| chown cert files | ❌ | ✅ | ✅ |
| Restore certificate | ❌ | ✅ only | ✅ only |
| Create endpoint | ✅ | ✅ | ✅ |
| Create database + backup | ✅ only | ❌ | ❌ |
| CREATE AVAILABILITY GROUP | ✅ only | ❌ | ❌ |
| JOIN AVAILABILITY GROUP | ❌ | ✅ only | ✅ only |
| Verify AG health | ✅ | ✅ | ✅ |

---

> **Next Step (Optional):** For **automatic failover**, install **Pacemaker + Corosync** as a cluster manager on top of this setup and change `CLUSTER_TYPE = EXTERNAL` and `FAILOVER_MODE = EXTERNAL`.
