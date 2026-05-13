# SQL Server Always On AG — Automatic Failover Setup
**OS:** Rocky Linux 9 | **Cluster:** Pacemaker 2.1.10 + Corosync

---

## ⚠️ Prerequisites — You Must Complete These First

This document is **Part 3** of a 3-part series. Before following any steps here, you must have completed both of the following guides in order:

| Step | Document | Description |
|------|----------|-------------|
| 1 | [Microsoft SQL Server Installation Guide on Rocky Linux](https://github.com/samratabduljalil/SOP_Submission_TSL/blob/main/mssql/Microsoft%20SQL%20Server%20Installation%20Guide%20on%20Rocky%20Linux.md) | Install and configure SQL Server on all 3 nodes |
| 2 | [MSSQL 3-Node Master-Slave Setup](https://github.com/samratabduljalil/SOP_Submission_TSL/blob/main/mssql/MSSQL-3-Node_Master-Slave_Setup.md) | Create the Always On Availability Group with certificates, endpoints, and replication |

> **Do not proceed with this guide unless AG1 is already created, AGTestDB is replicating across all 3 nodes, and all replicas show HEALTHY.**

---

## Architecture Overview

| Node  | Hostname | IP Address       | Role                            | Commit Mode  |
|-------|----------|------------------|---------------------------------|--------------|
| node2 | node2    | 192.168.109.168  | **Primary (Master)** ← current  | Synchronous  |
| node1 | node1    | 192.168.109.170  | Secondary (Slave 1)             | Synchronous  |
| node3 | node3    | 192.168.109.171  | Secondary (Slave 2)             | Synchronous  |

> **Current State:** node2 (`192.168.109.168`) is the active Primary. All 3 nodes are **SYNCHRONOUS_COMMIT** — any 1 node can die and automatic failover will work because 2 synchronous replicas remain.
> **VIP:** `192.168.109.100` — always points to current Primary. Use this for all application connections.

---

## Original AG Configuration (Reference Only — Do Not Re-run)

The AG was originally created with `CLUSTER_TYPE = NONE`. This must be recreated with `CLUSTER_TYPE = EXTERNAL` for Pacemaker to work. See Step 6A.

```sql
-- ORIGINAL (reference only)
CREATE AVAILABILITY GROUP [AG1]
WITH (CLUSTER_TYPE = NONE, DB_FAILOVER = ON, DTC_SUPPORT = NONE)
FOR DATABASE [AGTestDB] ...
```

> **Important:** `CLUSTER_TYPE` cannot be changed after creation — the AG must be dropped and recreated with `CLUSTER_TYPE = EXTERNAL`. See Step 6A.

---

## Prerequisites

- All 3 nodes running Rocky Linux 9
- SQL Server installed, HADR enabled, AG endpoint on port **5022**
- SA password: `Sam98765`
- Nodes reachable by hostname and IP
- Firewall allows: **2224** (pcsd), **5405** (Corosync), **1433** (SQL Server), **5022** (AG endpoint)

---

## Step 1 — Enable the HighAvailability Repository

> The HA repo is **disabled by default** on Rocky Linux. Without this, pacemaker/corosync packages will not be found.

Run on **all 3 nodes**:

```bash
# Check Rocky Linux version first
cat /etc/rocky-release

# Enable HA repo (Rocky Linux 9)
sudo dnf config-manager --set-enabled highavailability

# For Rocky Linux 8 use:
# sudo dnf config-manager --set-enabled ha

# If dnf config-manager is missing:
sudo dnf install -y 'dnf-command(config-manager)'

# Verify repo is active
sudo dnf repolist | grep -i ha
```

---

## Step 2 — Install Pacemaker & Corosync

Run on **all 3 nodes**:

```bash
sudo dnf install -y pacemaker corosync pcs fence-agents-all
sudo systemctl enable --now pcsd

# Set the SAME hacluster password on ALL nodes
sudo passwd hacluster
```

---

## Step 3 — Open Required Firewall Ports

Run on **all 3 nodes**:

```bash
sudo firewall-cmd --permanent --add-service=high-availability
sudo firewall-cmd --permanent --add-port=1433/tcp   # SQL Server
sudo firewall-cmd --permanent --add-port=5022/tcp   # AG mirroring endpoint
sudo firewall-cmd --reload
```

---

## Step 4 — Enable HADR in SQL Server

Run on **all 3 nodes**:

```bash
sudo /opt/mssql/bin/mssql-conf set hadr.hadrenabled 1
sudo systemctl restart mssql-server
```

Verify:
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
SELECT value_in_use FROM sys.configurations WHERE name = 'hadr enabled';
"
```

Expected: `1`

---

## Step 5 — Install the SQL Server HA Resource Agent

Run on **all 3 nodes**:

```bash
sudo dnf install -y mssql-server-ha
```

---

## Step 6 — Create Pacemaker SQL Login

> **Critical:** Use `CHECK_POLICY = OFF` to bypass OS password policy. The login must exist on ALL nodes. Recreate it after every SQL Server restart.

**On node1** (`192.168.109.170`):
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
USE master;
CREATE LOGIN pacemaker WITH PASSWORD = 'Pacemaker123!', CHECK_POLICY = OFF, CHECK_EXPIRATION = OFF;
GRANT ALTER, CONTROL, VIEW DEFINITION ON AVAILABILITY GROUP::[AG1] TO pacemaker;
GRANT VIEW SERVER STATE TO pacemaker;
GRANT VIEW ANY DEFINITION TO pacemaker;
"
```

**On node2** (`192.168.109.168`):
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
USE master;
CREATE LOGIN pacemaker WITH PASSWORD = 'Pacemaker123!', CHECK_POLICY = OFF, CHECK_EXPIRATION = OFF;
GRANT ALTER, CONTROL, VIEW DEFINITION ON AVAILABILITY GROUP::[AG1] TO pacemaker;
GRANT VIEW SERVER STATE TO pacemaker;
GRANT VIEW ANY DEFINITION TO pacemaker;
"
```

**On node3** (`192.168.109.171`):
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
USE master;
CREATE LOGIN pacemaker WITH PASSWORD = 'Pacemaker123!', CHECK_POLICY = OFF, CHECK_EXPIRATION = OFF;
GRANT ALTER, CONTROL, VIEW DEFINITION ON AVAILABILITY GROUP::[AG1] TO pacemaker;
GRANT VIEW SERVER STATE TO pacemaker;
GRANT VIEW ANY DEFINITION TO pacemaker;
"
```

> **Why `GRANT VIEW ANY DEFINITION`?** Without this, when the Pacemaker resource agent logs in as `pacemaker`, `sys.availability_groups` returns 0 rows even though the AG exists. This causes the cryptic error `Did not find AG row in sys.availability_groups` and the resource fails to start. Always include this grant.

Save credentials file on **all 3 nodes**:

```bash
sudo mkdir -p /var/opt/mssql/secrets
# IMPORTANT: file must contain ONLY the value — no "username=" prefix
sudo bash -c 'echo "pacemaker" > /var/opt/mssql/secrets/passwd'
sudo bash -c 'echo "Pacemaker123!" >> /var/opt/mssql/secrets/passwd'
sudo chmod 400 /var/opt/mssql/secrets/passwd
sudo chown root:root /var/opt/mssql/secrets/passwd

# Verify — must show exactly 2 lines: username then password
sudo cat /var/opt/mssql/secrets/passwd
```

Expected output:
```
pacemaker
Pacemaker123!
```

Verify pacemaker login can see the AG:
```bash
sqlcmd -S localhost -U pacemaker -P 'Pacemaker123!' -C -Q "
SELECT name FROM sys.availability_groups;
"
```

Must return `AG1`. If it returns 0 rows, re-run the `GRANT VIEW ANY DEFINITION` command.

---

## Step 6A — Recreate AG with CLUSTER_TYPE = EXTERNAL

> `CLUSTER_TYPE` cannot be altered after creation. The AG must be dropped and recreated. `CLUSTER_TYPE = EXTERNAL` with `FAILOVER_MODE = EXTERNAL` on ALL replicas (including async node3) is required for Pacemaker.

### Stop Pacemaker first

**On node2:**
```bash
sudo pcs cluster stop --all
```

### Drop AG on ALL nodes

**On node2:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "DROP AVAILABILITY GROUP [AG1];"
```

**On node1:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "DROP AVAILABILITY GROUP [AG1];"
```

**On node3:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "DROP AVAILABILITY GROUP [AG1];"
```

### Drop stale database on secondaries ONLY

**On node1:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "DROP DATABASE [AGTestDB];"
```

**On node3:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "DROP DATABASE [AGTestDB];"
```

### Verify AGTestDB is ONLINE on node2

**On node2:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
SELECT name, state_desc FROM sys.databases WHERE name = 'AGTestDB';
"
```

If `RESTORING`, recover it:
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
RESTORE DATABASE [AGTestDB] WITH RECOVERY;
"
```

### Take fresh backup on node2

```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
BACKUP DATABASE [AGTestDB] TO DISK = '/tmp/AGTestDB.bak' WITH INIT, FORMAT;
BACKUP LOG [AGTestDB] TO DISK = '/tmp/AGTestDB_log.bak' WITH INIT, FORMAT;
"
```

### Recreate AG with CLUSTER_TYPE = EXTERNAL

**On node2:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
CREATE AVAILABILITY GROUP [AG1]
WITH (
    CLUSTER_TYPE = EXTERNAL,
    DB_FAILOVER  = ON,
    DTC_SUPPORT  = NONE
)
FOR DATABASE [AGTestDB]
REPLICA ON
    N'node2' WITH (
        ENDPOINT_URL      = N'TCP://node2:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE     = EXTERNAL,
        SEEDING_MODE      = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
    ),
    N'node1' WITH (
        ENDPOINT_URL      = N'TCP://node1:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE     = EXTERNAL,
        SEEDING_MODE      = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
    ),
    N'node3' WITH (
        ENDPOINT_URL      = N'TCP://node3:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE     = EXTERNAL,
        SEEDING_MODE      = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
    );
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
"
```

> **All 3 replicas use `SYNCHRONOUS_COMMIT`** — this ensures automatic failover works when any single node dies, because 2 synchronous replicas always remain available.

### Join secondaries IMMEDIATELY after creating AG

**On node1:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
ALTER AVAILABILITY GROUP [AG1] JOIN WITH (CLUSTER_TYPE = EXTERNAL);
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
"
```

**On node3:**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
ALTER AVAILABILITY GROUP [AG1] JOIN WITH (CLUSTER_TYPE = EXTERNAL);
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
"
```

### Wait for seeding — verify ALL replicas HEALTHY before proceeding

**On node2:**
```bash
sleep 60
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
SELECT replica_server_name, role_desc, synchronization_health_desc
FROM sys.dm_hadr_availability_replica_states r
JOIN sys.availability_replicas ar ON r.replica_id = ar.replica_id;
"
```

Expected — all must show HEALTHY:
```
node2   PRIMARY    HEALTHY
node1   SECONDARY  HEALTHY
node3   SECONDARY  HEALTHY
```

### Set REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 0

> **Critical — run this while all nodes are healthy and online.** This setting must replicate to all nodes before any failover occurs. If set only on the Primary and Primary dies before replicating, secondaries won't have it and failover will fail with `need 2 but have 1`.

**On node2 (Primary):**
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
ALTER AVAILABILITY GROUP [AG1]
  SET (REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 0);
"
```

Verify it applied on ALL nodes:
```bash
# Run on node1, node2, node3
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
SELECT name, required_synchronized_secondaries_to_commit
FROM sys.availability_groups WHERE name = 'AG1';
"
```

All must return `0`.

> **Do NOT start Pacemaker until all replicas show HEALTHY.**

---

## Step 7 — Create and Configure the Pacemaker Cluster

**On node2:**
```bash
# Authenticate nodes
pcs host auth node1 node2 node3 -u hacluster

# Create cluster
pcs cluster setup sql_cluster node2 node1 node3 --force

# Start and enable
pcs cluster start --all
pcs cluster enable --all
```

### Disable STONITH and fix CIB if needed

```bash
pcs property set stonith-enabled=false
```

If CIB schema errors appear, reset the CIB on **all 3 nodes**:
```bash
sudo pcs cluster stop --all

# Run on ALL 3 nodes:
sudo rm -f /var/lib/pacemaker/cib/cib.xml*
sudo rm -f /var/lib/pacemaker/cib/cib-*

# Restart:
sudo pcs cluster start --all
sleep 15
pcs property set stonith-enabled=false
pcs status
```

All 3 nodes must show `Online` with no errors before continuing.

---

## Step 8 — Add the AG Resource to Pacemaker

> **Important:** Use `Promoted`/`Unpromoted` — NOT `Master`/`Slave`. Those are old Pacemaker 1.x terms. Pacemaker 2.x requires the new names. Also `notify=true` must be set on the resource itself via `meta`, not just on the clone.

**On node2:**
```bash
sudo pcs resource create ag_cluster ocf:mssql:ag \
  ag_name="AG1" \
  meta failure-timeout=60s notify=true \
  op start timeout=60s \
  op stop timeout=60s \
  op promote timeout=60s \
  op demote timeout=10s \
  op monitor timeout=60s interval=10s \
  op monitor timeout=60s interval=11s role="Promoted" \
  op monitor timeout=60s interval=12s role="Unpromoted"
```

Make it promotable:
```bash
sudo pcs resource promotable ag_cluster \
  meta promoted-max=1 promoted-node-max=1 \
  clone-max=3 clone-node-max=1 notify=true
```

Wait 60 seconds — do NOT rush:
```bash
sleep 60
pcs status
```

If resources show FAILED, clean up and wait again:
```bash
sudo pcs resource cleanup ag_cluster
sleep 60
pcs status
```

Expected:
```
Clone Set: ag_cluster-clone [ag_cluster] (promotable):
  * Promoted: [ node2 ]
  * Unpromoted: [ node1 node3 ]
```

---

## Step 9 — Create Virtual IP (AG Listener)

**On node2:**
```bash
sudo pcs resource create vip_ag ocf:heartbeat:IPaddr2 \
  ip=192.168.109.100 cidr_netmask=24 \
  op monitor interval=30s
```

---

## Step 10 — Add Colocation and Ordering Constraints

**On node2:**
```bash
# VIP must run on whichever node is Promoted (Primary)
sudo pcs constraint colocation add vip_ag with Promoted ag_cluster-clone INFINITY

# AG must be promoted before VIP starts
sudo pcs constraint order promote ag_cluster-clone then start vip_ag
```

---

## Step 11 — Final Verification

```bash
pcs status
pcs constraint show --full
```

Expected final output:
```
Clone Set: ag_cluster-clone [ag_cluster] (promotable):
  * Promoted: [ node2 ]
  * Unpromoted: [ node1 node3 ]
* vip_ag (ocf::heartbeat:IPaddr2): Started node2
```

---

## Testing Automatic Failover

### Test 1 — Simulate node2 failure

```bash
# On node2
sudo pcs cluster stop node2

# On node1, watch failover
watch -n2 'pcs status'
```

Pacemaker detects failure in ~20–30s, promotes node1, moves VIP to node1.

### Test 2 — Manual failover back to node2

```bash
pcs resource move ag_cluster-clone node2 --promoted
# Clear constraint after verifying
pcs resource clear ag_cluster-clone
```

### Test 3 — Bring node2 back

```bash
sudo pcs cluster start node2
pcs status
```

node2 rejoins as Secondary automatically.

---

## Failover Scenarios

### Scenario 1 — node2 (Primary) dies
```
node2 ✗  node1 ✓  node3 ✓  →  node1 auto-promoted ✅
```
Pacemaker detects in ~20s, promotes node1, VIP moves to node1.

### Scenario 2 — node1 (Secondary) dies
```
node2 ✓  node1 ✗  node3 ✓  →  node2 stays Primary ✅
```
No failover needed. node1 resyncs when it comes back.

### Scenario 3 — node3 (Secondary) dies
```
node2 ✓  node1 ✓  node3 ✗  →  node2 stays Primary ✅
```
No failover needed. node3 resyncs when it comes back.

### Scenario 4 — 2 nodes die simultaneously (worst case)
```
node2 ✗  node1 ✗  node3 ✓  →  cluster freezes ❌
```
Manual intervention required — stop Pacemaker then:
```bash
sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "
ALTER AVAILABILITY GROUP [AG1] FORCE_FAILOVER_ALLOW_DATA_LOSS;
"
```

> With 3 nodes: **1 node can always die and auto-failover works. 2 nodes die = manual recovery needed.**

---

## Manual Failover (Controlled)

To manually move Primary to a specific node:
```bash
# Move Primary to node1
pcs resource move ag_cluster-clone node1 --promoted
# Clear constraint after done
pcs resource clear ag_cluster-clone
```

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `No match for argument: pacemaker` | HA repo not enabled | `dnf config-manager --set-enabled highavailability` |
| `No stonith devices` | STONITH enabled, no fencing | `pcs property set stonith-enabled=false` |
| `CIB did not pass schema validation` | Corrupt CIB | Delete CIB files, restart cluster |
| `Did not find AG row in sys.availability_groups` | pacemaker login missing `VIEW ANY DEFINITION` | `GRANT VIEW ANY DEFINITION TO pacemaker` on all nodes |
| `Seeding cancelled: Database With Name Already Exists` | Stale DB on secondary in RESTORING state | `DROP DATABASE [AGTestDB]` on secondary nodes only |
| `CLUSTER_TYPE` alter fails | CLUSTER_TYPE is immutable after creation | Drop and recreate the entire AG |
| `Invalid usage of CLUSTER_TYPE in ALTER` | Running on a Secondary node | Only run AG-wide changes on the Primary node |
| `Cannot process ALTER: not the primary replica` | Wrong node | Connect to current Primary and retry |
| `Password validation failed` | OS password policy | Use `CHECK_POLICY = OFF, CHECK_EXPIRATION = OFF` |
| `Resource not running, cannot move` | Resource stopped | `pcs resource enable ag_cluster-clone` first |
| `notify=true` error on resource stop | notify not set on resource | Add `meta notify=true` on the resource definition |
| Login failed for `username=pacemaker` | Wrong format in secrets file | File must have only `pacemaker` then `Pacemaker123!` — no `username=` prefix |
| `Not enough replicas: need 2 but have 1` | `REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT` not 0, or only 1 sync replica visible | Set `REQUIRED_SYNCHRONIZED_SECONDARIES_TO_COMMIT = 0` on Primary while all nodes healthy, OR make all replicas synchronous |
| `EXTERNAL cluster type only supports EXTERNAL failover mode` | node3 had MANUAL failover mode | All replicas must use `FAILOVER_MODE = EXTERNAL` when `CLUSTER_TYPE = EXTERNAL` |

---

## Troubleshooting Commands

| Issue | Node | Command |
|-------|------|---------|
| Check cluster status | any | `pcs status` |
| Check cluster logs | any | `sudo journalctl -u pacemaker -f` |
| Clear failed resource | node2 | `sudo pcs resource cleanup ag_cluster` |
| Test pacemaker login | any | `sqlcmd -S localhost -U pacemaker -P 'Pacemaker123!' -C -Q "SELECT name FROM sys.availability_groups"` |
| Check AG replica states | any | `sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "SELECT replica_server_name, role_desc, synchronization_health_desc FROM sys.dm_hadr_availability_replica_states r JOIN sys.availability_replicas ar ON r.replica_id=ar.replica_id"` |
| Verify AG exists | any | `sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "SELECT name, cluster_type_desc FROM sys.availability_groups"` |
| Check seeding progress | node2 | `sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "SELECT * FROM sys.dm_hadr_automatic_seeding"` |
| Check DB state | any | `sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "SELECT name, state_desc FROM sys.databases WHERE name='AGTestDB'"` |
| Check endpoint state | any | `sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "SELECT name, state_desc FROM sys.database_mirroring_endpoints"` |
| Check HADR enabled | any | `sqlcmd -S localhost -U SA -P 'Sam98765' -C -Q "SELECT value_in_use FROM sys.configurations WHERE name = 'hadr enabled'"` |

---

## Connecting to the Cluster

Always connect via the **VIP** — never hardcode individual node IPs in your application.

| Method | Connection |
|---|---|
| sqlcmd | `sqlcmd -S 192.168.109.100 -U SA -P 'Sam98765' -C` |
| SSMS | Server: `192.168.109.100,1433` |
| ADO.NET | `Server=192.168.109.100,1433;Database=AGTestDB;User Id=SA;Password=Sam98765;` |
| Python pyodbc | `DRIVER={ODBC Driver 18 for SQL Server};SERVER=192.168.109.100,1433;DATABASE=AGTestDB;UID=SA;PWD=Sam98765;TrustServerCertificate=yes;` |

> The VIP always points to the current Primary. After failover your app reconnects automatically — no config change needed.

---

## Summary of Automatic Failover Behavior

```
All 3 nodes: SYNCHRONOUS_COMMIT
VIP: 192.168.109.100 → always follows Primary

node2 dies  →  Pacemaker detects (~20–30s)
            →  node1 OR node3 promoted to Primary
            →  VIP moves to new Primary
            →  Applications reconnect automatically

Recovered node  →  Rejoins as Secondary, resyncs automatically
```

> With all 3 nodes synchronous, **any single node can die** and automatic failover works. Two nodes dying simultaneously requires manual recovery.

---

*Document prepared for Rocky Linux 9 — SQL Server AG `[AG1]` protecting `[AGTestDB]`, all 3 replicas SYNCHRONOUS_COMMIT, managed by Pacemaker 2.1.10 + Corosync. VIP: `192.168.109.100`*
