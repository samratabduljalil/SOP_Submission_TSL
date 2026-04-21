# MongoDB Replica Set Cluster on Rocky Linux
### A Beginner-Friendly, Production-Oriented Guide

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites](#2-prerequisites)
3. [Environment Setup (All Nodes)](#3-environment-setup-all-nodes)
4. [Install MongoDB (All Nodes)](#4-install-mongodb-all-nodes)
5. [Configure MongoDB (All Nodes)](#5-configure-mongodb-all-nodes)
6. [Firewall & SELinux Configuration](#6-firewall--selinux-configuration)
7. [Start & Enable MongoDB Service](#7-start--enable-mongodb-service)
8. [Initialize the Replica Set](#8-initialize-the-replica-set)
9. [Create Admin Users](#9-create-admin-users)
10. [Enable Authentication & Keyfile Security](#10-enable-authentication--keyfile-security)
11. [Verify Replication](#11-verify-replication)
12. [Failover Testing](#12-failover-testing)
13. [Cluster Health Checks](#13-cluster-health-checks)
14. [Backup & Restore](#14-backup--restore)
15. [Troubleshooting Tips](#15-troubleshooting-tips)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Overview & Architecture

### What is a Replica Set?

A MongoDB **replica set** is a group of MongoDB instances that maintain the same dataset. It provides:

- **High Availability** — automatic failover if the primary goes down
- **Data Redundancy** — multiple copies of your data
- **Read Scalability** — secondary nodes can serve read requests

### Our 3-Node Architecture

```
┌─────────────────────────────────────────────────────┐
│               MongoDB Replica Set                   │
│                   (rs0)                             │
│                                                     │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────┐ │
│  │   Node 1     │ │   Node 2     │ │   Node 3    │ │
│  │  (Primary)   │◄│ (Secondary)  │◄│ (Secondary) │ │
│  │192.168.109.  │ │192.168.109.  │ │192.168.109. │ │
│  │    150       │ │    151       │ │    152      │ │
│  │   :27017     │ │   :27017     │ │   :27017    │ │
│  └──────────────┘ └──────────────┘ └─────────────┘ │
│         │                │                │         │
│         └────────────────┴────────────────┘         │
│                   Oplog Replication                 │
└─────────────────────────────────────────────────────┘
```

### Node Reference Table

| Hostname        | IP Address       | Role      |
|-----------------|------------------|-----------|
| mongo-node1     | 192.168.109.150  | Primary   |
| mongo-node2     | 192.168.109.151  | Secondary |
| mongo-node3     | 192.168.109.152  | Secondary |

> **Note:** These are the actual IP addresses used throughout this guide. If your environment differs, replace them consistently everywhere.

---

## 2. Prerequisites

Before starting, ensure you have:

- 3 servers running **Rocky Linux 8 or 9**
- **Root or sudo access** on all 3 nodes
- All nodes can **reach each other on port 27017**
- Minimum **2 GB RAM** and **10 GB disk space** per node (more for production)
- **NTP/Chrony** configured for time synchronization (critical for replica sets)

### Verify Rocky Linux Version

Run this on each node:

```bash
cat /etc/rocky-release
```

Expected output:
```
Rocky Linux release 9.x (Blue Onyx)
```

---

## 3. Environment Setup (All Nodes)

> ⚠️ **Run every command in this section on ALL 3 nodes unless stated otherwise.**

### 3.1 Set Hostnames

Set unique hostnames on each respective node:

```bash
# On Node 1
sudo hostnamectl set-hostname mongo-node1

# On Node 2
sudo hostnamectl set-hostname mongo-node2

# On Node 3
sudo hostnamectl set-hostname mongo-node3
```

### 3.2 Configure /etc/hosts

Add all node entries to `/etc/hosts` on **every** node so they can resolve each other by hostname:

```bash
sudo tee -a /etc/hosts <<EOF

# MongoDB Replica Set Nodes
192.168.109.150    mongo-node1
192.168.109.151    mongo-node2
192.168.109.152    mongo-node3
EOF
```

**Why this matters:** MongoDB replica set members communicate using hostnames. Without proper DNS or `/etc/hosts` entries, nodes will fail to connect to each other.

Verify connectivity from each node:

```bash
ping -c 3 mongo-node1
ping -c 3 mongo-node2
ping -c 3 mongo-node3
```

### 3.3 Synchronize Time with Chrony

Time skew between nodes can cause replication issues. Ensure Chrony is running:

```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
chronyc tracking
```

The output should show your reference server and low system time offset (ideally < 1 second).

### 3.4 Disable Transparent Huge Pages (THP)

MongoDB recommends disabling THP for better performance:

```bash
# Create a systemd unit to disable THP at boot
sudo tee /etc/systemd/system/disable-transparent-huge-pages.service <<EOF
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=mongod.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now disable-transparent-huge-pages.service
```

Verify it's applied:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# Expected: always madvise [never]
```

### 3.5 Update the System

```bash
sudo dnf update -y
```

---

## 4. Install MongoDB (All Nodes)

### 4.1 Add the MongoDB Repository

Create the Yum repository file for MongoDB 7.0 (latest stable):

```bash
sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo <<EOF
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/\$releasever/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF
```

**What this does:** Tells DNF where to download MongoDB packages and which GPG key to use for verification.

### 4.2 Install MongoDB

```bash
sudo dnf install -y mongodb-org
```

This installs the following packages:
- `mongodb-org-server` — The `mongod` daemon
- `mongodb-org-mongos` — The `mongos` router (for sharding)
- `mongodb-org-database-tools-extra` — Extra tools
- `mongodb-org-tools` — Import/export utilities
- `mongosh` — The MongoDB Shell

### 4.3 Verify Installation

```bash
mongod --version
mongosh --version
```

Expected output (versions may vary):
```
db version v7.0.x
Build Info: { ... }
```

---

## 5. Configure MongoDB (All Nodes)

MongoDB's main configuration file is `/etc/mongod.conf`. We'll configure it for replica set use.

### 5.1 Edit mongod.conf

```bash
sudo cp /etc/mongod.conf /etc/mongod.conf.bak  # Always backup first!
sudo tee /etc/mongod.conf <<'EOF'
# mongod.conf - MongoDB Configuration File
# For documentation: https://docs.mongodb.com/manual/reference/configuration-options/

# === Storage Settings ===
storage:
  dbPath: /data/mongo             # Custom data directory
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1               # RAM allocated to WiredTiger cache
                                   # Production tip: set to ~50% of available RAM

# === System Log Settings ===
systemLog:
  destination: file
  logAppend: true                  # Append to existing log, don't overwrite
  path: /log/mongo/mongod.log      # Custom log directory

# === Network Settings ===
net:
  port: 27017                      # Default MongoDB port
  bindIp: 0.0.0.0                  # Listen on all interfaces
                                   # Production tip: restrict to specific IPs
                                   # e.g., 127.0.0.1,192.168.109.150

# === Process Management ===
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

# === Security Settings (enabled after replica set init) ===
# security:
#   authorization: enabled
#   keyFile: /etc/mongodb-keyfile

# === Replica Set Settings ===
replication:
  replSetName: "rs0"               # All nodes MUST use the same name
                                   # This identifies your replica set

EOF
```

> ⚠️ **MongoDB 7.0 Note:** The `storage.journal.enabled` option was **removed in MongoDB 7.0** — journaling is always enabled and can no longer be configured. Including it will cause `mongod` to fail with `Unrecognized option: storage.journal.enabled`. If you copied a config from an older guide or have it in an existing file, remove it with:
> ```bash
> sudo sed -i '/journal:/d' /etc/mongod.conf
> sudo sed -i '/enabled: true.*durability/d' /etc/mongod.conf
> ```
> Then restart: `sudo systemctl restart mongod`

> **Important:** The `replSetName` value (`rs0`) must be **identical** on all three nodes. Any mismatch will prevent nodes from joining the set.

### 5.2 Create Custom Data & Log Directories

Since we're using non-default paths (`/data/mongo` and `/log/mongo`), we must create them and assign correct ownership **on all nodes**:

```bash
# Create the directories
sudo mkdir -p /data/mongo
sudo mkdir -p /log/mongo

# Assign ownership to the mongod user
sudo chown -R mongod:mongod /data/mongo
sudo chown -R mongod:mongod /log/mongo

# Set strict permissions
sudo chmod 750 /data/mongo
sudo chmod 750 /log/mongo
```

Apply SELinux file contexts to the custom paths so mongod is allowed to access them:

```bash
sudo semanage fcontext -a -t mongod_var_lib_t "/data/mongo(/.*)?"
sudo restorecon -Rv /data/mongo

sudo semanage fcontext -a -t mongod_log_t "/log/mongo(/.*)?"
sudo restorecon -Rv /log/mongo
```

Verify the directories are ready:

```bash
ls -laZ /data/mongo
ls -laZ /log/mongo
# The SELinux context column should show: mongod_var_lib_t and mongod_log_t respectively
```

---

## 6. Firewall & SELinux Configuration

### 6.1 Configure Firewalld

Open port 27017 between MongoDB nodes:

```bash
# Allow MongoDB port through the firewall
sudo firewall-cmd --permanent --add-port=27017/tcp
sudo firewall-cmd --reload

# Verify the rule was added
sudo firewall-cmd --list-ports
```

Expected output:
```
27017/tcp
```

> **Production tip:** Instead of opening port 27017 to everyone, restrict access to only your MongoDB node IPs:
> ```bash
> sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.109.0/24" port protocol="tcp" port="27017" accept'
> sudo firewall-cmd --reload
> ```

### 6.2 Configure SELinux

Rocky Linux uses SELinux in **enforcing** mode by default. You must tell SELinux to allow MongoDB to operate correctly.

#### Option A: Set the correct SELinux context on custom directories (Recommended)

```bash
# Install the SELinux policy tools
sudo dnf install -y policycoreutils-python-utils

# Allow mongod to use the custom data path
sudo semanage fcontext -a -t mongod_var_lib_t "/data/mongo(/.*)?"
sudo restorecon -Rv /data/mongo

# Allow mongod to use the custom log path
sudo semanage fcontext -a -t mongod_log_t "/log/mongo(/.*)?"
sudo restorecon -Rv /log/mongo
```

#### Option B: Check SELinux is not blocking MongoDB (for troubleshooting)

```bash
# Check for SELinux denials related to mongod
sudo ausearch -c 'mongod' --raw | audit2allow -M mongod_custom
sudo semodule -i mongod_custom.pp
```

#### Verify SELinux Status

```bash
sestatus
```

> **Note:** Setting SELinux to permissive (`sudo setenforce 0`) will make things work but is **not recommended for production**. Always use proper context labels instead.

---

## 7. Start & Enable MongoDB Service

Start MongoDB on all 3 nodes:

```bash
# Start the mongod service
sudo systemctl start mongod

# Enable it to start automatically on boot
sudo systemctl enable mongod

# Verify the service is running
sudo systemctl status mongod
```

Expected output (look for `active (running)`):
```
● mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; enabled)
     Active: active (running) since ...
```

### Check the MongoDB Log

If the service fails to start, check the log for errors:

```bash
sudo tail -50 /log/mongo/mongod.log
```

Common startup errors:
- `Data directory /data/mongo not found` → Check permissions with `ls -la /data/mongo`
- `Address already in use` → Another process is using port 27017: `sudo ss -tlnp | grep 27017`
- `Permission denied` → SELinux blocking: check `sudo ausearch -c mongod`

---

## 8. Initialize the Replica Set

> ⚠️ **Run the replica set initiation from Node 1 (Primary) only.**

### 8.1 Connect to MongoDB Shell on Node 1

```bash
mongosh
```

### 8.2 Initiate the Replica Set

Inside the MongoDB shell, run:

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-node1:27017", priority: 2 },
    { _id: 1, host: "mongo-node2:27017", priority: 1 },
    { _id: 2, host: "mongo-node3:27017", priority: 1 }
  ]
})
```

**Understanding the parameters:**
- `_id: "rs0"` — Must match the `replSetName` in `mongod.conf`
- `host` — Must be reachable from all other nodes (use hostnames, not IPs)
- `priority` — Higher priority = more likely to be elected Primary. Node 1 has `2`, others have `1`
- `_id` per member — Unique integer identifier for each member

Expected output:
```json
{ "ok": 1 }
```

### 8.3 Watch the Replica Set Come Online

After initiating, the shell prompt will change. Wait ~30 seconds and then check the status:

```javascript
rs.status()
```

Look for the `stateStr` field for each member:
- `"PRIMARY"` — This node accepts writes
- `"SECONDARY"` — This node replicates data from Primary
- `"STARTUP2"` — Node is syncing (wait a moment)

---

## 9. Create Admin Users

Before enabling authentication, create your admin user **while auth is still disabled**.

### 9.1 Connect to the Primary (Node 1)

```bash
mongosh
```

### 9.2 Create the Admin User

```javascript
use admin

db.createUser({
  user: "mongoAdmin",
  pwd: "YourStrongPasswordHere",   // Replace with a secure password!
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase",   db: "admin" },
    { role: "clusterAdmin",         db: "admin" }
  ]
})
```

**Role explanations:**
| Role | Purpose |
|------|---------|
| `userAdminAnyDatabase` | Create/manage users across all databases |
| `readWriteAnyDatabase` | Read and write to all databases |
| `dbAdminAnyDatabase` | Admin operations (indexes, stats) on all databases |
| `clusterAdmin` | Manage replica set and cluster operations |

Expected output:
```json
{ "ok": 1 }
```

---

## 10. Enable Authentication & Keyfile Security

Keyfile authentication allows replica set members to authenticate with each other.

### 10.1 Generate the Keyfile (On Node 1 Only)

```bash
# Generate a random 756-byte keyfile
openssl rand -base64 756 > /tmp/mongodb-keyfile

# Copy it to the final location
sudo cp /tmp/mongodb-keyfile /etc/mongodb-keyfile

# Set strict permissions — mongod requires this exact ownership/mode
sudo chown mongod:mongod /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile
```

### 10.2 Copy Keyfile to Nodes 2 and 3

The **exact same keyfile** must exist on all nodes.

```bash
# From Node 1, copy to Node 2 and Node 3
scp /tmp/mongodb-keyfile user@mongo-node2:/tmp/mongodb-keyfile
scp /tmp/mongodb-keyfile user@mongo-node3:/tmp/mongodb-keyfile
```

Then on **Node 2** and **Node 3**, run:

```bash
sudo cp /tmp/mongodb-keyfile /etc/mongodb-keyfile
sudo chown mongod:mongod /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile

# Apply the correct SELinux context
sudo semanage fcontext -a -t mongod_var_lib_t "/etc/mongodb-keyfile"
sudo restorecon -v /etc/mongodb-keyfile
```

### 10.3 Enable Authentication in mongod.conf (All Nodes)

Uncomment the security section in `/etc/mongod.conf` on **all 3 nodes**:

```bash
sudo sed -i 's/^# security:/security:/' /etc/mongod.conf
sudo sed -i 's/^#   authorization: enabled/  authorization: enabled/' /etc/mongod.conf
sudo sed -i 's/^#   keyFile: \/etc\/mongodb-keyfile/  keyFile: \/etc\/mongodb-keyfile/' /etc/mongod.conf
```

Or manually edit `/etc/mongod.conf` so the security section looks like this:

```yaml
security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile
```

### 10.4 Restart MongoDB on All Nodes

```bash
sudo systemctl restart mongod
sudo systemctl status mongod
```

### 10.5 Connect with Authentication

Now connect to the shell using credentials:

```bash
mongosh -u mongoAdmin -p YourStrongPasswordHere --authenticationDatabase admin
```

Or using a connection string:

```bash
mongosh "mongodb://mongoAdmin:YourStrongPasswordHere@mongo-node1:27017/admin?replicaSet=rs0"
```

---

## 11. Verify Replication

### 11.1 Check Replica Set Status

```javascript
rs.status()
```

A healthy replica set will show:
```json
{
  "set": "rs0",
  "members": [
    { "name": "mongo-node1:27017", "stateStr": "PRIMARY",   "health": 1 },
    { "name": "mongo-node2:27017", "stateStr": "SECONDARY", "health": 1 },
    { "name": "mongo-node3:27017", "stateStr": "SECONDARY", "health": 1 }
  ],
  "ok": 1
}
```

### 11.2 Test Write Replication

**On the Primary (Node 1)**, write a test document:

```javascript
use testdb

db.replication_test.insertOne({
  message: "Hello from Primary!",
  timestamp: new Date()
})
```

**On a Secondary (Node 2 or 3)**, verify the data was replicated:

```bash
# Connect to a secondary node
mongosh -u mongoAdmin -p YourStrongPasswordHere --authenticationDatabase admin --host mongo-node2
```

```javascript
// Allow reads on secondary
rs.secondaryOk()

use testdb
db.replication_test.find().pretty()
```

You should see the document inserted on the Primary. If you do — **replication is working!** ✅

### 11.3 Check Replication Lag

```javascript
rs.printSecondaryReplicationInfo()
```

This shows how far behind each secondary is from the primary. Ideally, lag should be 0 seconds or a few milliseconds.

---

## 12. Failover Testing

This section walks you through simulating a real-world failure scenario to verify your replica set will automatically elect a new primary.

> 💡 **Why this matters:** In production, nodes can fail due to hardware issues, network partitions, or OS crashes. Your cluster must survive these events without manual intervention.

### 12.1 Identify the Current Primary

```javascript
rs.isMaster()
// or
rs.status()
```

Note the `"primary"` field value (e.g., `"mongo-node1:27017"`).

### 12.2 Simulate Primary Node Failure

On the current **Primary node** (Node 1):

```bash
# Graceful stop (simulates controlled shutdown)
sudo systemctl stop mongod

# OR to simulate a hard crash:
sudo kill -9 $(pgrep mongod)
```

### 12.3 Observe the Election Process

On **Node 2** or **Node 3**, watch the replica set elect a new primary:

```bash
# Connect to a running secondary
mongosh -u mongoAdmin -p YourStrongPasswordHere --authenticationDatabase admin --host mongo-node2
```

```javascript
// Watch the election happen in real time
// Run this several times over ~30 seconds
rs.status()
```

You'll see the sequence:
1. `"stateStr": "SECONDARY"` → `"RECOVERING"` → `"PRIMARY"` (on one of the nodes)
2. Node 1 appears as `"(not reachable/healthy)"`
3. A new primary is elected within **10–30 seconds**

**What MongoDB does automatically:**
- Detects the primary is unreachable (heartbeat timeout)
- Eligible secondaries hold an election
- The node with the most up-to-date oplog and highest priority wins
- The winner steps up to Primary and resumes accepting writes

### 12.4 Verify Writes Still Work

After the election, connect to the **new Primary** and test writes:

```bash
mongosh "mongodb://mongoAdmin:YourStrongPasswordHere@mongo-node2:27017,mongo-node3:27017/admin?replicaSet=rs0"
```

```javascript
use testdb
db.replication_test.insertOne({ message: "Written to new Primary!", timestamp: new Date() })
```

If this succeeds — your **automatic failover is working correctly!** ✅

### 12.5 Recover the Failed Node

Bring Node 1 back online:

```bash
# On Node 1
sudo systemctl start mongod
sudo systemctl status mongod
```

Node 1 will automatically:
1. Detect the replica set's current state
2. Recognize a new Primary exists
3. Sync missed oplog entries (catch up)
4. Transition back to `SECONDARY`

Verify recovery from a shell connected to the cluster:

```javascript
rs.status()
```

Node 1 should now show `"stateStr": "SECONDARY"` with `"health": 1`.

> **Note:** If Node 1 was down for a long time and missed too many operations for oplog catch-up, it will perform an **initial sync** — a full data copy from the primary. This is automatic but can take minutes to hours depending on data size.

### 12.6 Manually Step Down Primary (Graceful Failover)

For planned maintenance, trigger a graceful step-down instead of killing the process:

```javascript
// On the current Primary
rs.stepDown()
// Primary steps down for 60 seconds, then may re-elect itself
```

---

## 13. Cluster Health Checks

Use these commands regularly to monitor your replica set.

### 13.1 Full Status Check

```javascript
rs.status()
```

**Key fields to watch:**
| Field | What to Look For |
|-------|-----------------|
| `health` | Must be `1` for all members |
| `stateStr` | One `PRIMARY`, rest `SECONDARY` |
| `optimeDate` | Should be close to current time |
| `lastHeartbeat` | Should be recent (within seconds) |

### 13.2 Replica Set Configuration

```javascript
rs.conf()
```

Shows your current replica set configuration including member priorities, votes, and hidden/delayed settings.

### 13.3 Check Oplog Window

```javascript
rs.printReplicationInfo()
```

Output example:
```
configured oplog size: 2048 MB
log length start to end: 72400 secs (20.11 hrs)
oplog first event time: ...
oplog last event time:  ...
```

The **oplog window** is how far back in time a secondary can fall behind before it needs a full resync. In production, this should be at least 24 hours.

### 13.4 Check Individual Member Health

```javascript
db.adminCommand({ replSetGetStatus: 1 }).members.forEach(m => {
  print(m.name + " | State: " + m.stateStr + " | Health: " + m.health)
})
```

### 13.5 Monitor Real-Time Replication Stats

```bash
# From the OS, monitor replication stats every 5 seconds
mongostat --host 192.168.109.150:27017 -u mongoAdmin -p YourStrongPasswordHere --authenticationDatabase admin --interval 5
```

### 13.6 Check for Replication Errors in Logs

```bash
sudo grep -i "repl\|election\|error\|FATAL" /log/mongo/mongod.log | tail -50
```

---

## 14. Backup & Restore

> 💡 **Always run backups against a Secondary node** to avoid impacting Primary performance. With authentication enabled, always pass credentials.

### 14.1 Install MongoDB Database Tools

`mongodump` and `mongorestore` are part of the `mongodb-database-tools` package:

```bash
sudo dnf install -y mongodb-database-tools
mongodump --version
```

---

### 14.2 Backup — Full Cluster Dump (via Replica Set URI)

This connects to the replica set and automatically picks a secondary to dump from:

```bash
# Create a timestamped backup directory
BACKUP_DIR="/backup/mongodb/$(date +%Y%m%d_%H%M%S)"
sudo mkdir -p /backup/mongodb

mongodump \
  --uri="mongodb://mongoAdmin:YourStrongPasswordHere@192.168.109.150:27017,192.168.109.151:27017,192.168.109.152:27017/admin?replicaSet=rs0&readPreference=secondary" \
  --out="$BACKUP_DIR" \
  --oplog \
  --gzip

echo "Backup saved to: $BACKUP_DIR"
```

**Options explained:**
| Option | Purpose |
|--------|---------|
| `--uri` | Full replica set connection string |
| `--out` | Directory to store the backup files |
| `--oplog` | Captures oplog entries for a point-in-time consistent backup |
| `--gzip` | Compresses each BSON file to save disk space |

---

### 14.3 Backup — Single Database

```bash
mongodump \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --out="/backup/mongodb/testdb_$(date +%Y%m%d)" \
  --gzip
```

---

### 14.4 Backup — Single Collection

```bash
mongodump \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --collection="replication_test" \
  --out="/backup/mongodb/single_collection_$(date +%Y%m%d)" \
  --gzip
```

---

### 14.5 Backup — Export to JSON/CSV with mongoexport

`mongoexport` is useful for human-readable exports (not for full restores):

```bash
# Export a collection to JSON
mongoexport \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --collection="replication_test" \
  --out="/backup/mongodb/replication_test.json" \
  --jsonArray

# Export to CSV with specific fields
mongoexport \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --collection="replication_test" \
  --type=csv \
  --fields="message,timestamp" \
  --out="/backup/mongodb/replication_test.csv"
```

---

### 14.6 Restore — Full Cluster Restore

> ⚠️ **`mongorestore` to a replica set always writes through the Primary.** Point to the Primary node (or the replica set URI).

```bash
mongorestore \
  --uri="mongodb://mongoAdmin:YourStrongPasswordHere@192.168.109.150:27017,192.168.109.151:27017,192.168.109.152:27017/admin?replicaSet=rs0" \
  --oplogReplay \
  --gzip \
  /backup/mongodb/20240101_120000/
```

**Options explained:**
| Option | Purpose |
|--------|---------|
| `--oplogReplay` | Replays the oplog captured during dump for point-in-time consistency |
| `--gzip` | Required if backup was created with `--gzip` |
| Last argument | Path to the backup directory created by `mongodump` |

---

### 14.7 Restore — Single Database

```bash
mongorestore \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --gzip \
  /backup/mongodb/20240101_120000/testdb/
```

---

### 14.8 Restore — Single Collection

```bash
mongorestore \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --collection="replication_test" \
  --gzip \
  /backup/mongodb/20240101_120000/testdb/replication_test.bson.gz
```

---

### 14.9 Restore — Import from JSON/CSV with mongoimport

```bash
# Import from JSON
mongoimport \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --collection="replication_test" \
  --file="/backup/mongodb/replication_test.json" \
  --jsonArray

# Import from CSV
mongoimport \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb" \
  --collection="replication_test" \
  --type=csv \
  --headerline \
  --file="/backup/mongodb/replication_test.csv"
```

---

### 14.10 Automate Backups with a Cron Job

Create a backup script and schedule it daily:

```bash
sudo tee /usr/local/bin/mongodb-backup.sh <<'EOF'
#!/bin/bash
# MongoDB Daily Backup Script

BACKUP_BASE="/backup/mongodb"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$BACKUP_BASE/$TIMESTAMP"
RETENTION_DAYS=7
LOGFILE="/log/mongo/backup.log"

echo "[$TIMESTAMP] Starting MongoDB backup..." | tee -a "$LOGFILE"

mkdir -p "$BACKUP_DIR"

mongodump \
  --uri="mongodb://mongoAdmin:YourStrongPasswordHere@192.168.109.150:27017,192.168.109.151:27017,192.168.109.152:27017/admin?replicaSet=rs0&readPreference=secondary" \
  --out="$BACKUP_DIR" \
  --oplog \
  --gzip >> "$LOGFILE" 2>&1

if [ $? -eq 0 ]; then
  echo "[$TIMESTAMP] Backup completed successfully: $BACKUP_DIR" | tee -a "$LOGFILE"
else
  echo "[$TIMESTAMP] ERROR: Backup FAILED!" | tee -a "$LOGFILE"
  exit 1
fi

# Remove backups older than RETENTION_DAYS
find "$BACKUP_BASE" -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
echo "[$TIMESTAMP] Old backups cleaned up (>${RETENTION_DAYS} days)" | tee -a "$LOGFILE"
EOF

sudo chmod +x /usr/local/bin/mongodb-backup.sh
```

Schedule with cron (runs at 2:00 AM daily):

```bash
echo "0 2 * * * root /usr/local/bin/mongodb-backup.sh" | sudo tee /etc/cron.d/mongodb-backup
```

Verify the cron job:

```bash
sudo cat /etc/cron.d/mongodb-backup
# Test manually:
sudo /usr/local/bin/mongodb-backup.sh
```

---

### 14.11 Verify a Backup

Always verify your backups are restorable:

```bash
# Check the backup directory contents
ls -lh /backup/mongodb/20240101_120000/

# Restore to a test database to verify integrity
mongorestore \
  --host="192.168.109.150:27017" \
  --username="mongoAdmin" \
  --password="YourStrongPasswordHere" \
  --authenticationDatabase="admin" \
  --db="testdb_verify" \
  --gzip \
  /backup/mongodb/20240101_120000/testdb/ \
  --dryRun

# --dryRun validates the restore without writing any data
# Remove --dryRun to perform the actual restore to testdb_verify
```

---

### Backup Strategy Summary

| Method | Best For | Consistency |
|--------|----------|-------------|
| `mongodump --oplog` | Full cluster backup | Point-in-time |
| `mongodump --db` | Single database | Snapshot |
| `mongodump --collection` | Single collection | Snapshot |
| `mongoexport` | Human-readable / CSV export | Snapshot |
| Filesystem snapshot (LVM/cloud) | Large datasets, zero downtime | Point-in-time |

> **Production tip:** For datasets larger than 100GB, consider **filesystem-level snapshots** (LVM snapshots, AWS EBS snapshots, etc.) for faster and more consistent backups than `mongodump`.

---

## 15. Troubleshooting Tips

### ❌ "Unrecognized option: storage.journal.enabled"

**Problem:** The `storage.journal.enabled` option was removed in MongoDB 7.0. Including it causes `mongod` to exit immediately with `status=2/INVALIDARGUMENT`.

**Solution:** Remove the journal lines from `/etc/mongod.conf`:

```bash
sudo sed -i '/journal:/d' /etc/mongod.conf
sudo sed -i '/enabled: true.*durability/d' /etc/mongod.conf

# Verify the lines are gone
grep -n "journal" /etc/mongod.conf   # Should return nothing

# Restart the service
sudo systemctl restart mongod
sudo systemctl status mongod
```

> **Note:** In MongoDB 7.0+, journaling is always enabled automatically. You do not need to configure it.

---

### ❌ "Not master and slaveOk=false"

**Problem:** You're trying to read from a secondary without enabling it.

**Solution:**
```javascript
rs.secondaryOk()
// or in older MongoDB versions:
db.getMongo().setReadPref('secondaryPreferred')
```

---

### ❌ Members stuck in STARTUP2 / never become SECONDARY

**Problem:** Nodes can't communicate with each other.

**Checklist:**
```bash
# 1. Check firewall allows port 27017
sudo firewall-cmd --list-ports

# 2. Test TCP connectivity between nodes (run from each node)
nc -zv 192.168.109.151 27017
nc -zv 192.168.109.152 27017

# 3. Check /etc/hosts has correct entries
cat /etc/hosts | grep mongo

# 4. Check mongod is actually listening
ss -tlnp | grep 27017

# 5. Look for connection errors in logs
sudo grep "connection refused\|connect failed" /log/mongo/mongod.log
```

---

### ❌ "Authentication failed" errors

**Problem:** Keyfiles don't match or auth isn't configured correctly.

**Solution:**
```bash
# Verify keyfile checksums match on all nodes
md5sum /etc/mongodb-keyfile

# Check keyfile permissions
ls -la /etc/mongodb-keyfile
# Should be: -r-------- 1 mongod mongod

# Check mongod.conf security section is correct
grep -A3 "security:" /etc/mongod.conf
```

---

### ❌ Replica set election not happening / cluster stuck

**Problem:** Cluster can't reach majority consensus (needs 2 out of 3 nodes).

**Why:** MongoDB requires a **majority** (>50%) of voting members to elect a Primary. With 3 nodes, at least 2 must be reachable.

**Solution:** If two nodes are down, you may need to manually reconfigure:
```javascript
// Force reconfiguration with remaining node (EMERGENCY USE ONLY)
cfg = rs.conf()
cfg.members = [{ _id: 0, host: "192.168.109.150:27017" }]
rs.reconfig(cfg, { force: true })
```

---

### ❌ SELinux blocking mongod

```bash
# Check for SELinux denials
sudo ausearch -c 'mongod' -ts recent

# Generate and apply a policy
sudo ausearch -c 'mongod' --raw | audit2allow -M mongod_fix
sudo semodule -i mongod_fix.pp
```

---

### ❌ mongod fails to start — "Address already in use"

```bash
# Find what's using port 27017
sudo ss -tlnp | grep 27017
sudo lsof -i :27017

# Kill the conflicting process if safe to do so
sudo kill -9 <PID>
```

---

### ❌ Disk space causing replication issues

```bash
# Check disk usage on data and log volumes
df -h /data/mongo
df -h /log/mongo

# Check MongoDB data directory size breakdown
du -sh /data/mongo/*

# Check log file sizes
du -sh /log/mongo/*
```

---

### ❌ Node stuck in RECOVERING after keyfile is added

**Problem:** After enabling `keyFile` in `mongod.conf`, one node stays in `RECOVERING` state and other nodes show `authenticated: false` and `health: 0` for it in `rs.status()`.

**Root cause:** The `authorization: enabled` line was added alongside `keyFile`. These conflict — `keyFile` alone is enough and automatically enables inter-node authentication.

**Step 1 — Check the security section on the problem node:**
```bash
grep -A3 "security:" /etc/mongod.conf
```

It must show **only** this — no `authorization: enabled`:
```yaml
security:
  keyFile: /etc/mongodb-keyfile
```

**Step 2 — If `authorization: enabled` is present, remove it:**
```bash
sudo sed -i '/authorization: enabled/d' /etc/mongod.conf

# Verify
grep -A3 "security:" /etc/mongod.conf

sudo systemctl restart mongod
sudo systemctl status mongod
```

**Step 3 — Verify all keyfiles are identical across all nodes:**
```bash
# Run on each node — all 3 hashes must match exactly
md5sum /etc/mongodb-keyfile
```

**Step 4 — If keyfile MD5 differs on any node, reset it:**
```bash
# On the working node (e.g. Node 2 or Node 3), print the keyfile
cat /etc/mongodb-keyfile
```

Then on the problem node, paste the exact content:
```bash
sudo tee /etc/mongodb-keyfile << 'EOF'
<paste exact keyfile content here>
EOF

sudo chown mongod:mongod /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile
sudo restorecon -v /etc/mongodb-keyfile   # Restore SELinux context
sudo systemctl restart mongod
```

**Step 5 — Check the log for the exact auth error:**
```bash
sudo tail -30 /log/mongo/mongod.log
```

Look for keywords: `KeyFile mismatch`, `bad auth`, `Unauthorized`, `RECOVERY`.

---

### ❌ `scp` fails with "Permission denied" for keyfile distribution

**Problem:** Copying the keyfile from Node 1 to other nodes fails:
```
root@mongo-node2's password: Permission denied
samrat@mongo-node2's password: Permission denied
```

**Cause:** SSH root login or password authentication is disabled on the target node.

**Option 1 — Use the node's sudo username:**
```bash
scp /tmp/mongodb-keyfile samrat@192.168.109.151:/tmp/mongodb-keyfile
scp /tmp/mongodb-keyfile samrat@192.168.109.152:/tmp/mongodb-keyfile
```

**Option 2 — Enable password authentication on target nodes:**
```bash
# On Node 2 and Node 3
grep PasswordAuthentication /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

Then retry `scp`.

**Option 3 — Copy manually (no SSH needed):**

On Node 1, print the keyfile:
```bash
cat /tmp/mongodb-keyfile
```

Then log in directly to Node 2 and Node 3 and paste the content:
```bash
sudo tee /tmp/mongodb-keyfile << 'EOF'
<paste content here>
EOF
```

Then on each node:
```bash
sudo cp /tmp/mongodb-keyfile /etc/mongodb-keyfile
sudo chown mongod:mongod /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile
sudo restorecon -v /etc/mongodb-keyfile
sudo systemctl restart mongod
```

---

### ❌ Replica set re-elects a new Primary unexpectedly

**Problem:** After fixing Node 2 and Node 3, they form a 2-node majority and elect a new Primary (e.g. Node 3 becomes Primary, Node 2 becomes Secondary) while Node 1 is still recovering.

**This is expected and correct behavior.** MongoDB automatically elects a new Primary when the original Primary is unreachable. The cluster self-heals.

**To check which node is currently Primary:**
```bash
# Connect to any node
mongosh -u mongoAdmin -p YourStrongPasswordHere \
  --authenticationDatabase admin \
  --host 192.168.109.151

rs.isMaster().primary
```

**To verify the full cluster status from the current Primary:**
```javascript
rs.status()
```

Expected healthy output:
```
mongo-node1  →  SECONDARY    health: 1  ✅
mongo-node2  →  SECONDARY    health: 1  ✅
mongo-node3  →  PRIMARY      health: 1  ✅
```

> **Note:** The Primary role may have shifted from Node 1 to Node 2 or Node 3 during recovery. This is normal. If you want Node 1 to be Primary again, adjust priorities after all nodes are healthy:
> ```javascript
> cfg = rs.conf()
> cfg.members[0].priority = 2   // Node 1 highest priority
> cfg.members[1].priority = 1
> cfg.members[2].priority = 1
> rs.reconfig(cfg)
> ```

---

## 16. Quick Reference Cheat Sheet

### Service Management

```bash
sudo systemctl start mongod       # Start MongoDB
sudo systemctl stop mongod        # Stop MongoDB
sudo systemctl restart mongod     # Restart MongoDB
sudo systemctl status mongod      # Check status
sudo systemctl enable mongod      # Enable at boot
sudo journalctl -u mongod -f      # Follow service logs
```

### Connect to Cluster

```bash
# Connect via replica set URI (auto-routes to Primary)
mongosh "mongodb://mongoAdmin:PASSWORD@192.168.109.150:27017,192.168.109.151:27017,192.168.109.152:27017/admin?replicaSet=rs0"

# Connect to a specific node directly
mongosh -u mongoAdmin -p PASSWORD --authenticationDatabase admin --host 192.168.109.150
```

### Essential Shell Commands

```javascript
rs.status()                    // Full replica set status
rs.conf()                      // Current configuration
rs.isMaster()                  // Quick primary check
rs.stepDown()                  // Step down current primary
rs.secondaryOk()               // Allow reads on secondary
rs.printReplicationInfo()      // Oplog info
rs.printSecondaryReplicationInfo() // Lag info
db.adminCommand({ping: 1})     // Test connection
```

### Add a New Member to Existing Replica Set

```javascript
// From the Primary — add a 4th node
rs.add("192.168.109.153:27017")
```

### Remove a Member

```javascript
rs.remove("192.168.109.152:27017")
```

### Change Member Priority

```javascript
cfg = rs.conf()
cfg.members[2].priority = 0    // Set Node 3 to never become Primary
rs.reconfig(cfg)
```

### Backup & Restore Quick Reference

```bash
# Full cluster backup (point-in-time, compressed)
mongodump \
  --uri="mongodb://mongoAdmin:PASSWORD@192.168.109.150:27017,192.168.109.151:27017,192.168.109.152:27017/admin?replicaSet=rs0&readPreference=secondary" \
  --out="/backup/mongodb/$(date +%Y%m%d_%H%M%S)" \
  --oplog --gzip

# Single database backup
mongodump --host=192.168.109.150:27017 -u mongoAdmin -p PASSWORD \
  --authenticationDatabase admin --db mydb \
  --out=/backup/mongodb/mydb_$(date +%Y%m%d) --gzip

# Full cluster restore
mongorestore \
  --uri="mongodb://mongoAdmin:PASSWORD@192.168.109.150:27017,192.168.109.151:27017,192.168.109.152:27017/admin?replicaSet=rs0" \
  --oplogReplay --gzip /backup/mongodb/20240101_120000/

# Single database restore
mongorestore --host=192.168.109.150:27017 -u mongoAdmin -p PASSWORD \
  --authenticationDatabase admin --db mydb \
  --gzip /backup/mongodb/20240101_120000/mydb/

# Export collection to JSON
mongoexport --host=192.168.109.150:27017 -u mongoAdmin -p PASSWORD \
  --authenticationDatabase admin --db mydb \
  --collection mycollection --out=/backup/mycollection.json --jsonArray

# Import collection from JSON
mongoimport --host=192.168.109.150:27017 -u mongoAdmin -p PASSWORD \
  --authenticationDatabase admin --db mydb \
  --collection mycollection --file=/backup/mycollection.json --jsonArray
```

---

## Summary

You have successfully:

- ✅ Installed MongoDB 7.0 on 3 Rocky Linux nodes (`192.168.109.150/151/152`)
- ✅ Configured custom data (`/data/mongo`) and log (`/log/mongo`) directories
- ✅ Adjusted firewall and SELinux for secure operation
- ✅ Initialized a 3-node replica set (`rs0`)
- ✅ Created an admin user and enabled authentication with keyfile security
- ✅ Verified data replication across all nodes
- ✅ Performed and understood failover testing
- ✅ Learned backup and restore with `mongodump`/`mongorestore` and automated cron scheduling
- ✅ Learned how to monitor and maintain cluster health

### Next Steps for Production

- 📊 **Monitoring:** Set up MongoDB Cloud Manager, Prometheus + MongoDB Exporter, or Grafana
- 💾 **Backups:** Configure `mongodump` cron jobs or use MongoDB Atlas Backup
- 🔐 **TLS/SSL:** Enable encrypted connections with TLS certificates
- 📈 **Sizing:** Tune `cacheSizeGB` and oplog size based on workload
- 🚨 **Alerting:** Set up alerts for replication lag, disk usage, and node health

---


