# Apache Cassandra 3-Node Cluster: Complete Production Guide

> **OS:** Rocky Linux 9 | **Cassandra:** 4.1.x (Latest Stable) | **Java:** OpenJDK 11

---

## Table of Contents

1. [Cluster Architecture Overview](#1-cluster-architecture-overview)
2. [Pre-Installation: System Preparation](#2-pre-installation-system-preparation)
3. [Java Dependency Setup](#3-java-dependency-setup)
4. [Apache Cassandra Installation](#4-apache-cassandra-installation)
5. [Cassandra Configuration (`cassandra.yaml`)](#5-cassandra-configuration-cassandrayaml)
   - [Node 1 (Seed Node) — 192.168.109.156](#node-1-seed-node--19216810912)
   - [Node 2 — 192.168.109.158](#node-2--192168109153)
   - [Node 3 — 192.168.109.157](#node-3--192168109154)
6. [JVM & Performance Tuning](#6-jvm--performance-tuning)
7. [Firewall Port Configuration](#7-firewall-port-configuration)
8. [Cluster Formation: Starting Nodes in Order](#8-cluster-formation-starting-nodes-in-order)
9. [Cluster Verification](#9-cluster-verification)
10. [Failover Test: Simulate Node 2 Failure](#10-failover-test-simulate-node-2-failure)
11. [Backup with `nodetool snapshot`](#11-backup-with-nodetool-snapshot)
12. [Restore Procedure](#12-restore-procedure)
13. [Production Gotchas & Checklist](#13-production-gotchas--checklist)

---

## 1. Cluster Architecture Overview

| Node   | Hostname | IP Address       | Role        |
|--------|----------|------------------|-------------|
| Node 1 | node1    | 192.168.109.156  | Seed Node   |
| Node 2 | node2    | 192.168.109.158  | Non-Seed    |
| Node 3 | node3    | 192.168.109.157  | Non-Seed    |

- **Cluster Name:** `ProductionCluster`
- **Replication Factor:** 3 (data replicated across all nodes)
- **Snitch:** `GossipingPropertyFileSnitch`
- **Num Tokens:** 16 (virtual nodes)

> ⚠️ **Seed nodes are NOT special in production operation.** They are only used for initial bootstrapping. Never list ALL nodes as seeds — list only 1–2 stable nodes.

---

## 2. Pre-Installation: System Preparation

**Run on ALL 3 nodes unless stated otherwise.**

### 2.1 Set Hostnames

```bash
# [Node1 - 192.168.109.156]
sudo hostnamectl set-hostname node1

# [Node2 - 192.168.109.158]
sudo hostnamectl set-hostname node2

# [Node3 - 192.168.109.157]
sudo hostnamectl set-hostname node3
```

### 2.2 Update `/etc/hosts` on ALL Nodes

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]
sudo tee -a /etc/hosts <<EOF

# Cassandra Cluster Nodes
192.168.109.156  node1
192.168.109.158  node2
192.168.109.157  node3
EOF
```

### 2.3 Disable SELinux

Rocky Linux ships with SELinux enforcing by default. Cassandra requires either disabling SELinux or writing a full policy. For production environments with a dedicated security team, a custom SELinux policy is preferred. For most deployments, set to permissive or disabled.

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# Check current SELinux status
sestatus

# Option A: Set to permissive (logs violations but does not block — recommended for initial setup)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Option B: Disable completely (requires reboot)
# sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# sudo reboot

# Verify
getenforce
```

> ⚠️ **SELinux in enforcing mode will silently block Cassandra** from binding to ports, writing to data directories, and reading config files. Always confirm `getenforce` returns `Permissive` or `Disabled` before starting the cluster.

### 2.4 OS-Level Tuning

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# Disable swap (critical for Cassandra performance)
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Set vm.max_map_count for memory-mapped files
echo "vm.max_map_count=1048575" | sudo tee -a /etc/sysctl.conf

# Increase open file limits
# Rocky Linux uses /etc/security/limits.d/ drop-ins (preferred over limits.conf)
sudo tee /etc/security/limits.d/cassandra.conf <<EOF
cassandra  -  nofile   100000
cassandra  -  nproc    32768
cassandra  -  memlock  unlimited
EOF

# Set network buffer sizes
sudo tee -a /etc/sysctl.d/99-cassandra.conf <<EOF
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 87380 16777216
net.ipv4.tcp_wmem=4096 65536 16777216
vm.max_map_count=1048575
EOF

# Apply sysctl changes
sudo sysctl --system
```

> ⚠️ **Swap must be disabled.** Cassandra is very sensitive to swapping. If the JVM swaps heap pages, latency spikes dramatically and the node may be marked DOWN by the cluster.

---

## 3. Java Dependency Setup

Cassandra 4.1.x requires **Java 11**. Run on ALL 3 nodes.

### 3.1 Install OpenJDK 11

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

sudo dnf install -y java-11-openjdk-headless

# Verify Java installation
java -version
```

**Expected output (exact patch version may differ):**
```
openjdk version "11.0.25" 2024-10-15 LTS
OpenJDK Runtime Environment (Red_Hat-11.0.25.0.9-1) (build 11.0.25+9-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-11.0.25.0.9-1) (build 11.0.25+9-LTS, mixed mode, sharing)
```

> ⚠️ **Do not hardcode the Java patch version.** The Rocky Linux repos will install the latest patch release (e.g. `11.0.25` not `11.0.22`). Always use `readlink -f $(which java)` to get the real installed path rather than guessing it manually.

### 3.2 Auto-Detect and Set JAVA_HOME

Rocky Linux installs Java under a fully versioned path like `/usr/lib/jvm/java-11-openjdk-11.0.25.0.9-7.el9.x86_64`. This path changes with every patch update. Use auto-detection to always get it right:

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# Auto-detect the real Java binary path (resolves all symlinks)
REAL_JAVA=$(readlink -f $(which java))
REAL_JAVA_HOME=$(dirname $(dirname $REAL_JAVA))

echo "Detected Java binary : $REAL_JAVA"
echo "Detected JAVA_HOME   : $REAL_JAVA_HOME"

# Verify the binary actually exists at this path
ls -la $REAL_JAVA_HOME/bin/java
```

**Expected output:**
```
Detected Java binary : /usr/lib/jvm/java-11-openjdk-11.0.25.0.9-7.el9.x86_64/bin/java
Detected JAVA_HOME   : /usr/lib/jvm/java-11-openjdk-11.0.25.0.9-7.el9.x86_64
-rwxr-xr-x. 1 root root 12520 Oct 15 2024 /usr/lib/jvm/java-11-openjdk-11.0.25.0.9-7.el9.x86_64/bin/java
```

### 3.3 Set JAVA_HOME System-Wide

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# Write to /etc/profile.d/ — the correct place for system-wide env vars on Rocky Linux
# NOTE: Do NOT use /etc/environment for this — it does not support variable expansion
#       like $PATH:$JAVA_HOME and will silently break your PATH.
sudo tee /etc/profile.d/java.sh <<EOF
export JAVA_HOME=$REAL_JAVA_HOME
export PATH=\$JAVA_HOME/bin:\$PATH
EOF

sudo chmod +x /etc/profile.d/java.sh

# Apply to current session
source /etc/profile.d/java.sh

# Confirm
echo $JAVA_HOME
java -version
```

### 3.4 Set JAVA_HOME for Cassandra (Critical Step)

> ⚠️ **This step is mandatory on Rocky Linux.** The `cassandra` startup script (`/usr/sbin/cassandra`) does NOT read `/etc/profile.d/`. It sources its own include file at `/usr/share/cassandra/cassandra.in.sh`. If `JAVA_HOME` is not set there, Cassandra will print `Unable to find java executable` even when `java -version` works fine in your shell.
>
> The `cassandra-env.sh.d/` drop-in directory used on Debian/Ubuntu packages does **not exist** on RPM installs. The only correct location is `/usr/share/cassandra/cassandra.in.sh`.

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# Confirm the include file exists
ls -la /usr/share/cassandra/cassandra.in.sh

# Inject JAVA_HOME at the top of cassandra.in.sh using the auto-detected path
sudo sed -i "1s|^|export JAVA_HOME=$REAL_JAVA_HOME\n|" \
  /usr/share/cassandra/cassandra.in.sh

# Verify it was inserted correctly
head -3 /usr/share/cassandra/cassandra.in.sh
```

**Expected head output:**
```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.25.0.9-7.el9.x86_64
#
# Licensed to the Apache Software Foundation (ASF) under one
```

### 3.5 Verify Cassandra Can Find Java

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

cassandra -version
```

**Expected output:**
```
Cassandra 4.1.5
```

If this still fails, run the debug trace to pinpoint exactly where the lookup breaks:

```bash
bash -x /usr/sbin/cassandra -v 2>&1 | head -60
```

---

## 4. Apache Cassandra Installation

Run on ALL 3 nodes.

### 4.1 Install Prerequisites

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# If you get a GPG error about pgdg repos (see gotcha below), disable them first
sudo dnf config-manager --disable "pgdg*"

sudo dnf install -y curl wget gnupg2
```

> ⚠️ **pgdg (PostgreSQL) repo GPG conflict on Rocky Linux 9.** If your machine has any PostgreSQL repos configured, you will see this error during any `dnf` operation:
> ```
> Error: Failed to download metadata for repo 'pgdg-common':
> repomd.xml GPG signature verification error: gpgme_engine_check_version() error: Invalid crypto engine
> ```
> Rocky Linux 9 ships with **gpgme 1.18+** which enforces stricter crypto engine validation. The pgdg repo's GPG signature fails this check. Since this is a dedicated Cassandra cluster, disable all pgdg repos permanently:
> ```bash
> sudo dnf config-manager --disable "pgdg*"
> # Or remove them entirely if PostgreSQL is not needed on these nodes:
> sudo rm -f /etc/yum.repos.d/pgdg*.repo
> sudo dnf clean all
> ```

### 4.2 Add the Cassandra RPM Repository

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

sudo tee /etc/yum.repos.d/cassandra.repo <<EOF
[cassandra]
name=Apache Cassandra
baseurl=https://redhat.cassandra.apache.org/41x/
gpgcheck=0
repo_gpgcheck=0
enabled=1
EOF
```

> ⚠️ **Why `gpgcheck=0` here?** Rocky Linux 9's gpgme engine also rejects the Cassandra repo's `repomd.xml` GPG signature with the same `gpgme_engine_check_version()` error when `repo_gpgcheck=1` is set. Setting both `gpgcheck=0` and `repo_gpgcheck=0` bypasses this. After installation, you can harden it by importing the key and re-enabling checks:
> ```bash
> # Optional: re-enable GPG checking after importing the key
> curl -fsSL https://downloads.apache.org/cassandra/KEYS -o /tmp/cassandra.keys
> sudo rpm --import /tmp/cassandra.keys
> sudo sed -i 's/gpgcheck=0/gpgcheck=1/' /etc/yum.repos.d/cassandra.repo
> sudo sed -i 's/repo_gpgcheck=0/repo_gpgcheck=1/' /etc/yum.repos.d/cassandra.repo
> ```

### 4.3 Install Cassandra

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

sudo dnf clean all
sudo dnf install -y cassandra

# Verify installation
cassandra -version
```

**Expected output:**
```
getopt: invalid option -- 'e'
getopt: invalid option -- 'r'
getopt: invalid option -- 's'
getopt: invalid option -- 'i'
getopt: invalid option -- 'o'
getopt: invalid option -- 'n'
4.1.11
```

> ℹ️ **The `getopt` warnings are harmless.** This is a known cosmetic bug with the Cassandra RPM on Rocky Linux 9. As long as the version number prints at the end (e.g. `4.1.11`), the installation is successful. These warnings do not affect cluster operation.

> ⚠️ **`cassandra -version` will fail with "Unable to find java executable"** until you complete Section 3.4 (setting `JAVA_HOME` inside `/usr/share/cassandra/cassandra.in.sh`). The installation itself is successful — the Java lookup error is a separate configuration step. Do not re-install Cassandra if you see this message; proceed to Section 3 first.

```bash
# If systemctl reports "Unit cassandra.service not found", run daemon-reload first.
# This is a known behaviour on Rocky Linux 9 where the RPM registers the unit
# but systemd has not yet picked it up. A single daemon-reload fixes it.
sudo systemctl daemon-reload

# Stop Cassandra before configuring (important!)
sudo systemctl stop cassandra

# Verify it is stopped
sudo systemctl status cassandra
```

> ⚠️ **Do NOT start Cassandra yet.** You must configure `cassandra.yaml` on all nodes first, before starting any node. Starting with default config will create a single-node cluster that is difficult to re-join into a multi-node cluster.

---

## 5. Cassandra Configuration (`cassandra.yaml`)

On Rocky Linux, the RPM installs **two** config locations:

| Path | Purpose |
|------|---------|
| `/etc/cassandra/default.conf/cassandra.yaml` | RPM default — do not edit directly |
| `/etc/cassandra/conf/cassandra.yaml` | Active config read at runtime |

Always edit `/etc/cassandra/conf/cassandra.yaml`. If `conf/cassandra.yaml` is missing or empty, copy it from `default.conf` first.

---

### Node 1 (Seed Node) — 192.168.109.156

```bash
# [Node1 - 192.168.109.156]

# Back up the active config
sudo cp /etc/cassandra/conf/cassandra.yaml /etc/cassandra/conf/cassandra.yaml.bak

# Edit the  config
sudo nano /cassandra.yaml

# Replace yml file
sudo cp /cassandra.yaml /etc/cassandra/conf/cassandra.yaml
```

Find and set the following values in `cassandra.yaml`:

```yaml
# ============================================================
# NODE 1 CONFIGURATION (Seed Node — 192.168.109.156)
# ============================================================

# --- Cluster Identity ---
cluster_name: 'ProductionCluster'

# --- Partitioner ---
# Required on Rocky Linux RPM installs — the default conf may omit this.
# If missing, add it manually or Cassandra will fail with a partitioner error.
# Check with: grep partitioner /etc/cassandra/conf/cassandra.yaml
# If absent, run: sudo sed -i '/^cluster_name/a partitioner: org.apache.cassandra.dht.Murmur3Partitioner' /etc/cassandra/conf/cassandra.yaml
partitioner: org.apache.cassandra.dht.Murmur3Partitioner

# --- Virtual Nodes ---
# 16 vnodes offers a good balance for a 3-node cluster
num_tokens: 16

# --- Network Settings ---
# The IP address that this node binds to for internal cluster communication
listen_address: 192.168.109.156

# The IP address of the interface that Cassandra binds to for client connections
# (CQL clients, drivers)
rpc_address: 192.168.109.156

# Broadcast addresses (important if using NAT or multiple NICs)
broadcast_address: 192.168.109.156
broadcast_rpc_address: 192.168.109.156

# --- Seed Provider ---
# List seed nodes for initial cluster bootstrapping
# Only list 1–2 stable nodes; do NOT list all nodes
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "192.168.109.156"

# --- Snitch ---
# GossipingPropertyFileSnitch is recommended for production.
# It reads DC/rack info from cassandra-rackdc.properties
endpoint_snitch: GossipingPropertyFileSnitch

# --- Data Directories ---
data_file_directories:
  - /var/lib/cassandra/data

commitlog_directory: /var/lib/cassandra/commitlog
saved_caches_directory: /var/lib/cassandra/saved_caches
hints_directory: /var/lib/cassandra/hints

# ⚠️ CRITICAL: saved_caches_directory and hints_directory MUST be different paths.
# Cassandra will refuse to start with: "saved_caches_directory must not be the same
# as the hints_directory". Double-check these two lines on every node.

# --- Performance Tuning ---

# Compaction throughput (MB/s). Set to 0 to disable throttling.
# 64 MB/s is a safe production default.
compaction_throughput_mb_per_sec: 64

# Number of concurrent compactors. For SSDs, use number of disks.
# For HDDs, use 1 per spindle.
concurrent_compactors: 2

# Concurrent reads/writes. SSDs: (# of cores * 2), HDDs: (# of spindles * 16)
concurrent_reads: 32
concurrent_writes: 32
concurrent_counter_writes: 16

# Memtable flush writers (one per data directory is good default)
memtable_flush_writers: 2

# Commit log sync: periodic is generally faster; batch is safer.
commitlog_sync: periodic
commitlog_sync_period_in_ms: 10000
commitlog_segment_size_in_mb: 32

# Row cache: Only enable if you have specific hot rows. Default OFF.
row_cache_size_in_mb: 0

# Key cache (default is good; adjust based on heap size)
key_cache_size_in_mb: 100

# Phi threshold for failure detection (higher = less sensitive to pauses)
phi_convict_threshold: 8

# Read/Write timeouts
read_request_timeout_in_ms: 5000
write_request_timeout_in_ms: 2000
request_timeout_in_ms: 10000

# Tombstone warning/failure thresholds
tombstone_warn_threshold: 1000
tombstone_failure_threshold: 100000

# Hinted handoff: enables data repair when a node was temporarily down
hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000  # 3 hours

# --- Authentication (enable for production) ---
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
role_manager: CassandraRoleManager

# --- SSL (configure certs for production) ---
# client_encryption_options and server_encryption_options
# Should be configured with TLS in production — omitted here for brevity
```

#### Configure Rack & DC for Node 1

```bash
# [Node1 - 192.168.109.156]
sudo nano /etc/cassandra/conf/cassandra-rackdc.properties
```

```properties
# NODE 1 — cassandra-rackdc.properties
dc=DC1
rack=RACK1
```

---

### Node 2 — 192.168.109.158

```bash
# [Node2 - 192.168.109.158]
sudo cp /etc/cassandra/conf/cassandra.yaml /etc/cassandra/conf/cassandra.yaml.bak
sudo cp /etc/cassandra/default.conf/cassandra.yaml /etc/cassandra/conf/cassandra.yaml
sudo nano /etc/cassandra/conf/cassandra.yaml
```

```yaml
# ============================================================
# NODE 2 CONFIGURATION (192.168.109.158)
# ============================================================

# --- Cluster Identity ---
# MUST match exactly across all nodes (case-sensitive)
cluster_name: 'ProductionCluster'

# --- Virtual Nodes ---
num_tokens: 16

# --- Network Settings ---
listen_address: 192.168.109.158
rpc_address: 192.168.109.158
broadcast_address: 192.168.109.158
broadcast_rpc_address: 192.168.109.158

# --- Seed Provider ---
# Node 2 points to Node 1 as seed — do NOT add Node 2 itself as a seed
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "192.168.109.156"

# --- Snitch ---
endpoint_snitch: GossipingPropertyFileSnitch

# --- Data Directories ---
data_file_directories:
  - /var/lib/cassandra/data

commitlog_directory: /var/lib/cassandra/commitlog
saved_caches_directory: /var/lib/cassandra/saved_caches
hints_directory: /var/lib/cassandra/hints

# --- Performance Tuning (same as Node 1) ---
compaction_throughput_mb_per_sec: 64
concurrent_compactors: 2
concurrent_reads: 32
concurrent_writes: 32
concurrent_counter_writes: 16
memtable_flush_writers: 2

commitlog_sync: periodic
commitlog_sync_period_in_ms: 10000
commitlog_segment_size_in_mb: 32

row_cache_size_in_mb: 0
key_cache_size_in_mb: 100

phi_convict_threshold: 8
read_request_timeout_in_ms: 5000
write_request_timeout_in_ms: 2000
request_timeout_in_ms: 10000

tombstone_warn_threshold: 1000
tombstone_failure_threshold: 100000

hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000

# --- Authentication ---
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
role_manager: CassandraRoleManager
```

#### Configure Rack & DC for Node 2

```bash
# [Node2 - 192.168.109.158]
sudo nano /etc/cassandra/conf/cassandra-rackdc.properties
```

```properties
# NODE 2 — cassandra-rackdc.properties
dc=DC1
rack=RACK1
```

---

### Node 3 — 192.168.109.157

```bash
# [Node3 - 192.168.109.157]
sudo cp /etc/cassandra/conf/cassandra.yaml /etc/cassandra/conf/cassandra.yaml.bak
sudo cp /etc/cassandra/default.conf/cassandra.yaml /etc/cassandra/conf/cassandra.yaml
sudo nano /etc/cassandra/conf/cassandra.yaml
```

```yaml
# ============================================================
# NODE 3 CONFIGURATION (192.168.109.157)
# ============================================================

# --- Cluster Identity ---
cluster_name: 'ProductionCluster'

# --- Virtual Nodes ---
num_tokens: 16

# --- Network Settings ---
listen_address: 192.168.109.157
rpc_address: 192.168.109.157
broadcast_address: 192.168.109.157
broadcast_rpc_address: 192.168.109.157

# --- Seed Provider ---
# Node 3 also points to Node 1 as the seed
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "192.168.109.156"

# --- Snitch ---
endpoint_snitch: GossipingPropertyFileSnitch

# --- Data Directories ---
data_file_directories:
  - /var/lib/cassandra/data

commitlog_directory: /var/lib/cassandra/commitlog
saved_caches_directory: /var/lib/cassandra/saved_caches
hints_directory: /var/lib/cassandra/hints

# ⚠️ CRITICAL: saved_caches_directory and hints_directory MUST be different paths.
# Cassandra will refuse to start with: "saved_caches_directory must not be the same
# as the hints_directory". Double-check these two lines on every node.

# --- Performance Tuning (same as Node 1 & 2) ---
compaction_throughput_mb_per_sec: 64
concurrent_compactors: 2
concurrent_reads: 32
concurrent_writes: 32
concurrent_counter_writes: 16
memtable_flush_writers: 2

commitlog_sync: periodic
commitlog_sync_period_in_ms: 10000
commitlog_segment_size_in_mb: 32

row_cache_size_in_mb: 0
key_cache_size_in_mb: 100

phi_convict_threshold: 8
read_request_timeout_in_ms: 5000
write_request_timeout_in_ms: 2000
request_timeout_in_ms: 10000

tombstone_warn_threshold: 1000
tombstone_failure_threshold: 100000

hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000

# --- Authentication ---
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
role_manager: CassandraRoleManager
```

#### Configure Rack & DC for Node 3

```bash
# [Node3 - 192.168.109.157]
sudo nano /etc/cassandra/conf/cassandra-rackdc.properties
```

```properties
# NODE 3 — cassandra-rackdc.properties
dc=DC1
rack=RACK1
```

### 5.4 Create Required Data Directories (All Nodes)

The RPM does not always pre-create all directories referenced in `cassandra.yaml`. Create them explicitly on all nodes:

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

sudo mkdir -p /var/lib/cassandra/data
sudo mkdir -p /var/lib/cassandra/commitlog
sudo mkdir -p /var/lib/cassandra/saved_caches
sudo mkdir -p /var/lib/cassandra/hints
sudo chown -R cassandra:cassandra /var/lib/cassandra/
```

> ⚠️ **`saved_caches_directory` conflict fix.** If Cassandra fails on startup with `saved_caches_directory must not be the same as the hints_directory`, it means both are pointing to the same path in `cassandra.yaml`. Run this to detect and fix it:
> ```bash
> # Check current values
> grep -E "saved_caches_directory|hints_directory" /etc/cassandra/conf/cassandra.yaml
>
> # Fix if saved_caches_directory wrongly points to /var/lib/cassandra/hints
> sudo sed -i 's|saved_caches_directory: /var/lib/cassandra/hints|saved_caches_directory: /var/lib/cassandra/saved_caches|' \
>   /etc/cassandra/conf/cassandra.yaml
>
> # Re-create the directory and set ownership
> sudo mkdir -p /var/lib/cassandra/saved_caches
> sudo chown cassandra:cassandra /var/lib/cassandra/saved_caches
>
> # Verify both paths are now different
> grep -E "saved_caches_directory|hints_directory" /etc/cassandra/conf/cassandra.yaml
> ```

### 5.5 Verify partitioner Is Set (All Nodes)

The Rocky Linux RPM's default `cassandra.yaml` may omit the `partitioner` key. Cassandra will fail silently or use an unexpected default. Always verify:

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

grep partitioner /etc/cassandra/conf/cassandra.yaml
```

If the output is empty, add it:

```bash
sudo sed -i '/^cluster_name/a partitioner: org.apache.cassandra.dht.Murmur3Partitioner' \
  /etc/cassandra/conf/cassandra.yaml

# Verify
grep partitioner /etc/cassandra/conf/cassandra.yaml
```

**Expected output:**
```
partitioner: org.apache.cassandra.dht.Murmur3Partitioner
```

---

## 6. JVM & Performance Tuning

On Rocky Linux, the JVM options file is at **`/etc/cassandra/conf/jvm-server.options`** (not `jvm11-server.options` as on Debian/Ubuntu installs).

Edit on **ALL 3 nodes**:

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

sudo nano /etc/cassandra/conf/jvm-server.options
```

### 6.1 Set Heap Size with sed (Recommended)

The heap lines in `jvm-server.options` are commented out by default (prefixed with `#`). Use `sed` to uncomment and set them — this avoids manual editing errors:

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# --- For production nodes with 16GB+ RAM: use 4G heap ---
sudo sed -i 's/^#-Xms4G/-Xms4G/' /etc/cassandra/conf/jvm-server.options
sudo sed -i 's/^#-Xmx4G/-Xmx4G/' /etc/cassandra/conf/jvm-server.options

# --- For lab/low-RAM nodes (under 4GB RAM): use 256M or 512M ---
# First uncomment the 4G lines if they were commented, then override:
sudo sed -i 's/^#-Xms4G/-Xms512M/' /etc/cassandra/conf/jvm-server.options
sudo sed -i 's/^#-Xmx4G/-Xmx512M/' /etc/cassandra/conf/jvm-server.options
# Then reduce further if needed:
sudo sed -i 's/^-Xms512M/-Xms256M/' /etc/cassandra/conf/jvm-server.options
sudo sed -i 's/^-Xmx512M/-Xmx256M/' /etc/cassandra/conf/jvm-server.options

# Verify the result — both Xms and Xmx must be set and equal
grep -E "^-Xms|^-Xmx" /etc/cassandra/conf/jvm-server.options
```

**Expected output (production):**
```
-Xms4G
-Xmx4G
```

**Expected output (lab/low-RAM):**
```
-Xms256M
-Xmx256M
```

> ⚠️ **`-Xms` must equal `-Xmx`.** If they differ, the JVM resizes the heap under load causing latency spikes.

> ⚠️ **`MAX_HEAP_SIZE` should never exceed 8 GB.** Above 8 GB, G1GC pause times increase unpredictably. For nodes with 64 GB RAM, leave the rest for the OS page cache — Cassandra benefits heavily from OS-level SSTable file caching.

> ⚠️ **Low heap warning.** The log line `Global memtable on-heap threshold is enabled at 59MiB` means the heap is set to only 256M. This is fine for lab environments but will cause `OutOfMemoryError` under any real write load. Always size heap to at least 2G for any production workload.

### 6.2 Additional JVM Tuning Options

Add or verify these lines in `/etc/cassandra/conf/jvm-server.options`:

```bash
# Use G1 Garbage Collector (default for Java 11+)
-XX:+UseG1GC

# G1GC pause target in ms (lower = more frequent but shorter GC pauses)
-XX:MaxGCPauseMillis=300

# Reserve 10% of heap for G1 to avoid full GC
-XX:G1ReservePercent=10

# InitiatingHeapOccupancyPercent: start concurrent marking early
-XX:InitiatingHeapOccupancyPercent=45

# G1 heap region size
-XX:G1HeapRegionSize=16m

# Per-thread stack size
-Xss256k

# Disable explicit GC calls from Cassandra internals
-XX:+DisableExplicitGC

# Off-heap/direct memory — set equal to MAX_HEAP for file cache headroom
-XX:MaxDirectMemorySize=4G

# GC logging (recommended for debugging)
-Xlog:gc=info,gc+heap=debug,gc+age=trace:file=/var/log/cassandra/gc.log:time,uptime,pid,tid,level:filecount=10,filesize=10m
```

---

## 7. Firewall Port Configuration

Rocky Linux uses **`firewalld`** (not `ufw`). Run on **ALL 3 nodes**.

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# Ensure firewalld is running
sudo systemctl enable --now firewalld
sudo firewall-cmd --state

# Allow inter-node cluster communication (gossip + streaming)
sudo firewall-cmd --permanent --add-port=7000/tcp
sudo firewall-cmd --permanent --add-port=7001/tcp

# Allow JMX — restrict to management subnet only
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.109.0/24" port protocol="tcp" port="7199" accept'

# Allow native CQL transport (application clients)
sudo firewall-cmd --permanent --add-port=9042/tcp

# Allow CQL over SSL (if enabled)
sudo firewall-cmd --permanent --add-port=9142/tcp

# Reload firewall rules to apply
sudo firewall-cmd --reload

# Verify active rules
sudo firewall-cmd --list-all
```

**Expected output (excerpt):**
```
public (active)
  target: default
  ports: 7000/tcp 7001/tcp 9042/tcp 9142/tcp
  rich rules:
    rule family="ipv4" source address="192.168.109.0/24" port port="7199" protocol="tcp" accept
```

| Port | Protocol | Purpose                          | Exposure              |
|------|----------|----------------------------------|-----------------------|
| 7000 | TCP      | Inter-node cluster communication | Internal cluster only |
| 7001 | TCP      | Inter-node SSL                   | Internal cluster only |
| 7199 | TCP      | JMX monitoring (`nodetool`)      | Management subnet     |
| 9042 | TCP      | CQL native client transport      | Application tier      |
| 9142 | TCP      | CQL over SSL                     | Application tier      |

> ⚠️ **Never expose port 7199 (JMX) to the internet.** JMX has no authentication by default and allows remote code execution if exposed publicly.

---

## 8. Cluster Formation: Starting Nodes in Order

> ⚠️ **Order matters.** Always start the seed node first. Wait for it to fully initialize before starting subsequent nodes. Bringing up nodes simultaneously can result in a split-brain state or failed bootstrapping.

### Pre-Start: Reload systemd on All Nodes

Run this once on all 3 nodes before starting any Cassandra service. On Rocky Linux 9, the Cassandra RPM registers its systemd unit during install but systemd does not pick it up automatically — `daemon-reload` is required or `systemctl start cassandra` will return `Unit cassandra.service not found`.

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]
sudo systemctl daemon-reload
```

### Step 1 — Start Node 1 (Seed) First

```bash
# [Node1 - 192.168.109.156]
sudo systemctl start cassandra

# Watch the logs for successful startup
sudo journalctl -u cassandra -f
# OR
sudo tail -f /var/log/cassandra/system.log
```

Wait until you see in the logs:
```
INFO  [main] 2024-xx-xx ... Starting listening for CQL clients on /192.168.109.156:9042
INFO  [main] 2024-xx-xx ... Node /192.168.109.156 state jump to NORMAL
```

### Step 2 — Start Node 2

**Wait ~60 seconds** after Node 1 is in NORMAL state, then:

```bash
# [Node2 - 192.168.109.158]
sudo systemctl start cassandra
sudo tail -f /var/log/cassandra/system.log
```

Wait until you see:
```
INFO  [main] 2024-xx-xx ... Node /192.168.109.158 state jump to NORMAL
```

### Step 3 — Start Node 3

**Wait ~60 seconds** after Node 2 is in NORMAL state, then:

```bash
# [Node3 - 192.168.109.157]
sudo systemctl start cassandra
sudo tail -f /var/log/cassandra/system.log
```

### Enable Cassandra on Boot (All Nodes)

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]
sudo systemctl enable cassandra
```

---

## 9. Cluster Verification

### 9.1 Check Node Ring Status

```bash
# [Node1 - 192.168.109.156]
nodetool status
```

**Expected output (all nodes UN = Up/Normal):**
```
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens  Owns (effective)  Host ID                               Rack
UN  192.168.109.158  75.19 KiB  16      57.1%             d9f9e112-e15b-4206-897c-6389640423b1  rack1
UN  192.168.109.157  70.23 KiB  16      76.6%             e8272557-67bf-4847-b289-7e9683b72021  rack1
UN  192.168.109.156  125.4 KiB  16      66.3%             6b735d5f-70bc-474d-bde2-51c4276afd51  rack1
```

> Status `UN` = **U**p/**N**ormal. This is the healthy state.

> ℹ️ **`Owns` percentages do not need to sum to exactly 100% per node** in a 3-node cluster with `num_tokens=16`. The token ring is distributed probabilistically across vnodes — uneven splits (57%, 66%, 76%) are normal and will rebalance over time as data grows. What matters is that all nodes show `UN`.

> ℹ️ **Node 3 may briefly show `UJ` (Up/Joining)** while bootstrapping into the ring after a fresh start. Wait 60–120 seconds and re-run `nodetool status` — it will transition to `UN` once token streaming completes.

### 9.2 Additional Verification Commands

```bash
# [Node1 - 192.168.109.156]

# Show detailed ring topology
nodetool ring

# Show cluster info
nodetool info

# Check gossip state of all nodes
nodetool gossipinfo

# Verify cluster describes itself correctly
nodetool describecluster
```

**Expected `describecluster` output:**
```
Name: ProductionCluster
Snitch: org.apache.cassandra.locator.DynamicEndpointSnitch
DynamicEndpointSnitch: enabled
Partitioner: org.apache.cassandra.dht.Murmur3Partitioner
Schema versions:
        XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX: [192.168.109.156, 192.168.109.158, 192.168.109.157]
```

> ⚠️ **All 3 nodes must share the SAME schema version UUID.** If you see multiple UUIDs, there is a schema disagreement — investigate before writing data.

> ℹ️ **DC and rack names are lowercase (`dc1`, `rack1`)** as set in `cassandra-rackdc.properties`. This is what appears in `nodetool status` output. Ensure your keyspace replication strategy references the exact same DC name — e.g. `'dc1': 3` not `'DC1': 3`.

### 9.3 Create a Test Keyspace

```bash
# [Node1 - 192.168.109.156]
cqlsh 192.168.109.156 -u cassandra -p cassandra
```

```sql
-- In cqlsh shell:
-- Change default password immediately in production
ALTER USER cassandra WITH PASSWORD 'StrongPassword123!';

-- Create a test keyspace with replication factor 3 (all nodes)
CREATE KEYSPACE test_ks
  WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3
  };

-- Create a test table
USE test_ks;
CREATE TABLE users (
  user_id UUID PRIMARY KEY,
  name TEXT,
  email TEXT
);

-- Insert test data
INSERT INTO users (user_id, name, email) VALUES (uuid(), 'Alice', 'alice@example.com');
INSERT INTO users (user_id, name, email) VALUES (uuid(), 'Bob', 'bob@example.com');

-- Query
SELECT * FROM users;

EXIT;
```

---

## 10. Failover Test: Simulate Node 2 Failure

### Phase 1 — Stop Node 2 (Simulate Failure)

```bash
# [Node2 - 192.168.109.158]
sudo systemctl stop cassandra
```

### Phase 2 — Verify Cluster Handles the Failure

```bash
# [Node1 - 192.168.109.156]
nodetool status
```

**Expected output (Node 2 is DN = Down/Normal):**
```
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load       Tokens  Owns (effective)  Host ID                               Rack
UN  192.168.109.156  125.4 KiB  16      66.3%             6b735d5f-70bc-474d-bde2-51c4276afd51  rack1
DN  192.168.109.158  75.19 KiB  16      57.1%             d9f9e112-e15b-4206-897c-6389640423b1  rack1
UN  192.168.109.157  70.23 KiB  16      76.6%             e8272557-67bf-4847-b289-7e9683b72021  rack1
```

```bash
# [Node1 - 192.168.109.156]
# Verify reads/writes still work with RF=3 and consistency QUORUM (2 of 3 nodes available)
cqlsh 192.168.109.156 -u cassandra -p 'StrongPassword123!'

# In cqlsh:
# CONSISTENCY QUORUM requires (RF/2 + 1) = 2 nodes — still works with 2 UP nodes
CONSISTENCY QUORUM;
SELECT * FROM test_ks.users;
```

**Expected:** Data returns successfully. With `RF=3` and `QUORUM` consistency (2/3 nodes), the cluster tolerates 1 node failure.

> ⚠️ If you are using `CONSISTENCY ALL`, queries WILL fail when any node is down. Always use `QUORUM` or lower for production resilience.

### Phase 3 — Bring Node 2 Back

```bash
# [Node2 - 192.168.109.158]
sudo systemctl start cassandra
sudo tail -f /var/log/cassandra/system.log
```

Watch for:
```
INFO  [main] ... Node /192.168.109.158 state jump to NORMAL
```

### Phase 4 — Verify Node 2 Rejoined

```bash
# [Node1 - 192.168.109.156]
nodetool status
```

All 3 nodes should show `UN` again.

```bash
# [Node1 - 192.168.109.156]
# Run repair on the rejoining node to sync any missed writes
nodetool repair -h 192.168.109.158 test_ks
```

> ⚠️ After a node recovers from downtime, **always run `nodetool repair`** on it to catch up on any writes that were hinted (stored on other nodes while it was down). Hinted handoff only covers the `max_hint_window_in_ms` period (3 hours by default). For longer outages, full repair is essential.

---

## 11. Backup with `nodetool snapshot`

A Cassandra snapshot is a hard-link copy of SSTable files at a point in time. It is crash-consistent and non-blocking.

### 11.1 Pre-Backup: Flush Memtables to Disk

```bash
# Run on ALL 3 nodes before taking snapshots
# This ensures all in-memory data is flushed to SSTables

# [Node1 - 192.168.109.156]
nodetool flush

# [Node2 - 192.168.109.158]
nodetool flush

# [Node3 - 192.168.109.157]
nodetool flush
```

### 11.2 Take Snapshot of `test_ks` Keyspace on All Nodes

```bash
# [Node1 - 192.168.109.156]
nodetool snapshot --tag snapshot_20240822_0200 -- test_ks

# [Node2 - 192.168.109.158]
nodetool snapshot --tag snapshot_20240822_0200 -- test_ks

# [Node3 - 192.168.109.157]
nodetool snapshot --tag snapshot_20240822_0200 -- test_ks
```

**Expected output:**
```
Requested creating snapshot(s) for [test_ks] with snapshot name [snapshot_20240822_0200] and options {skipFlush=false}
Snapshot directory: snapshot_20240822_0200
```

### 11.3 Locate Snapshot Files

Snapshots are stored under the data directory:
```
/var/lib/cassandra/data/<keyspace>/<table-uuid>/snapshots/<tag>/
```

```bash
# [Node1 - 192.168.109.156]
find /var/lib/cassandra/data/test_ks -path "*/snapshots/snapshot_20240822_0200" -type d
```

**Example output:**
```
/var/lib/cassandra/data/test_ks/users-a1b2c3d4e5f640abcdef0123456789ab/snapshots/snapshot_20240822_0200
```

### 11.4 Copy Snapshot to Remote Backup Location

```bash
# [Node1 - 192.168.109.156]
SNAP_DATE="snapshot_20240822_0200"
BACKUP_DIR="/backups/cassandra/${SNAP_DATE}/node1"

mkdir -p ${BACKUP_DIR}

# Copy snapshot directories to backup location
find /var/lib/cassandra/data/test_ks -path "*/snapshots/${SNAP_DATE}" -type d | \
  while read snap_dir; do
    table_dir=$(echo "$snap_dir" | awk -F'/snapshots/' '{print $1}')
    table_name=$(basename "$table_dir")
    dest="${BACKUP_DIR}/${table_name}"
    mkdir -p "$dest"
    cp -r "${snap_dir}/"* "$dest/"
    echo "Copied: $snap_dir -> $dest"
  done

# [Node2 - 192.168.109.158]
SNAP_DATE="snapshot_20240822_0200"
BACKUP_DIR="/backups/cassandra/${SNAP_DATE}/node2"
mkdir -p ${BACKUP_DIR}
find /var/lib/cassandra/data/test_ks -path "*/snapshots/${SNAP_DATE}" -type d | \
  while read snap_dir; do
    table_dir=$(echo "$snap_dir" | awk -F'/snapshots/' '{print $1}')
    table_name=$(basename "$table_dir")
    dest="${BACKUP_DIR}/${table_name}"
    mkdir -p "$dest"
    cp -r "${snap_dir}/"* "$dest/"
  done

# [Node3 - 192.168.109.157]
SNAP_DATE="snapshot_20240822_0200"
BACKUP_DIR="/backups/cassandra/${SNAP_DATE}/node3"
mkdir -p ${BACKUP_DIR}
find /var/lib/cassandra/data/test_ks -path "*/snapshots/${SNAP_DATE}" -type d | \
  while read snap_dir; do
    table_dir=$(echo "$snap_dir" | awk -F'/snapshots/' '{print $1}')
    table_name=$(basename "$table_dir")
    dest="${BACKUP_DIR}/${table_name}"
    mkdir -p "$dest"
    cp -r "${snap_dir}/"* "$dest/"
  done
```

### 11.5 Verify and List Snapshots

```bash
# [Node1 - 192.168.109.156]
nodetool listsnapshots
```

**Expected output:**
```
Snapshot Details:
Tag Name                  Keyspace    Column Family  True Size  Size on Disk
snapshot_20240822_0200    test_ks     users          1.21 KiB   2.43 KiB

Total TrueDiskSpaceUsed: 1.21 KiB
```

### 11.6 Clean Up Old Snapshots

Snapshots are NOT automatically deleted. They accumulate and consume disk space.

```bash
# Delete a specific snapshot tag on all nodes
# [Node1 - 192.168.109.156]
nodetool clearsnapshot --tag snapshot_20240822_0200

# [Node2 - 192.168.109.158]
nodetool clearsnapshot --tag snapshot_20240822_0200

# [Node3 - 192.168.109.157]
nodetool clearsnapshot --tag snapshot_20240822_0200

# Delete ALL snapshots (use with caution!)
# nodetool clearsnapshot
```

> ⚠️ **Snapshots consume disk space equal to the size of the keyspace.** Because they use hard links, they don't cost extra space for unchanged files — but as compaction creates new SSTables, old snapshot files remain on disk until cleared. Monitor `/var/lib/cassandra/data` disk usage and implement a rotation policy.

---

## 12. Restore Procedure

> ⚠️ **Cassandra restore is a per-node operation.** Each node holds a portion of the data ring. You must restore each node individually and then run `nodetool repair` to ensure consistency.

### Scenario: Full Restore of `test_ks` to All 3 Nodes

### Step 1 — Re-create the Schema

Before restoring data, the keyspace and table schema must exist.

```bash
# [Node1 - 192.168.109.156]
cqlsh 192.168.109.156 -u cassandra -p 'StrongPassword123!'
```

```sql
-- If keyspace was dropped, recreate it first
CREATE KEYSPACE IF NOT EXISTS test_ks
  WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3
  };

-- Recreate the table schema
CREATE TABLE IF NOT EXISTS test_ks.users (
  user_id UUID PRIMARY KEY,
  name TEXT,
  email TEXT
);

EXIT;
```

### Step 2 — Stop Cassandra on All Nodes

```bash
# [Node1 - 192.168.109.156]
sudo systemctl stop cassandra

# [Node2 - 192.168.109.158]
sudo systemctl stop cassandra

# [Node3 - 192.168.109.157]
sudo systemctl stop cassandra
```

### Step 3 — Clear Existing Data for the Keyspace

```bash
# [Node1 - 192.168.109.156]
sudo rm -rf /var/lib/cassandra/data/test_ks/users-*/

# [Node2 - 192.168.109.158]
sudo rm -rf /var/lib/cassandra/data/test_ks/users-*/

# [Node3 - 192.168.109.157]
sudo rm -rf /var/lib/cassandra/data/test_ks/users-*/
```

### Step 4 — Restore Snapshot Files

The snapshot must be placed in the table's data directory (NOT the snapshots subdirectory).

```bash
# [Node1 - 192.168.109.156]
SNAP_DATE="snapshot_20240822_0200"
TABLE_DATA_DIR="/var/lib/cassandra/data/test_ks/users-a1b2c3d4e5f640abcdef0123456789ab"
# Note: Get the exact UUID directory name from: ls /var/lib/cassandra/data/test_ks/

sudo mkdir -p ${TABLE_DATA_DIR}

# Copy backup files into the data directory
sudo cp -r /backups/cassandra/${SNAP_DATE}/node1/users-*/. ${TABLE_DATA_DIR}/

# Fix ownership
sudo chown -R cassandra:cassandra /var/lib/cassandra/data/test_ks/


# [Node2 - 192.168.109.158]
TABLE_DATA_DIR="/var/lib/cassandra/data/test_ks/users-a1b2c3d4e5f640abcdef0123456789ab"
sudo mkdir -p ${TABLE_DATA_DIR}
sudo cp -r /backups/cassandra/${SNAP_DATE}/node2/users-*/. ${TABLE_DATA_DIR}/
sudo chown -R cassandra:cassandra /var/lib/cassandra/data/test_ks/


# [Node3 - 192.168.109.157]
TABLE_DATA_DIR="/var/lib/cassandra/data/test_ks/users-a1b2c3d4e5f640abcdef0123456789ab"
sudo mkdir -p ${TABLE_DATA_DIR}
sudo cp -r /backups/cassandra/${SNAP_DATE}/node3/users-*/. ${TABLE_DATA_DIR}/
sudo chown -R cassandra:cassandra /var/lib/cassandra/data/test_ks/
```

### Step 5 — Restart Cassandra Nodes (In Order)

```bash
# [Node1 - 192.168.109.156]  — Start seed first
sudo systemctl start cassandra
# Wait ~60 seconds and verify: nodetool status

# [Node2 - 192.168.109.158]
sudo systemctl start cassandra
# Wait ~60 seconds and verify: nodetool status

# [Node3 - 192.168.109.157]
sudo systemctl start cassandra
```

### Step 6 — Refresh SSTables

After placing snapshot files and restarting, tell Cassandra to load the new SSTables without a restart:

```bash
# [Node1 - 192.168.109.156]
nodetool refresh -- test_ks users

# [Node2 - 192.168.109.158]
nodetool refresh -- test_ks users

# [Node3 - 192.168.109.157]
nodetool refresh -- test_ks users
```

### Step 7 — Run Repair to Ensure Consistency

```bash
# [Node1 - 192.168.109.156]
nodetool repair test_ks

# [Node2 - 192.168.109.158]
nodetool repair test_ks

# [Node3 - 192.168.109.157]
nodetool repair test_ks
```

### Step 8 — Verify Restored Data

```bash
# [Node1 - 192.168.109.156]
cqlsh 192.168.109.156 -u cassandra -p 'StrongPassword123!'
```

```sql
SELECT COUNT(*) FROM test_ks.users;
SELECT * FROM test_ks.users LIMIT 10;
```

---

## 13. Production Gotchas & Checklist

### Critical Pre-Production Checklist

| # | Check | Command / Action |
|---|-------|-----------------|
| 1 | Swap disabled on all nodes | `cat /proc/swaps` — should be empty |
| 2 | SELinux set to permissive or disabled | `getenforce` — should return `Permissive` or `Disabled` |
| 3 | pgdg (PostgreSQL) repos disabled or removed | `sudo dnf repolist | grep pgdg` — should show nothing enabled |
| 4 | Cassandra repo `gpgcheck=0` OR GPG key imported | `cat /etc/yum.repos.d/cassandra.repo` |
| 5 | `JAVA_HOME` set in `/usr/share/cassandra/cassandra.in.sh` | `head -2 /usr/share/cassandra/cassandra.in.sh` |
| 6 | `JAVA_HOME` path verified with `readlink -f $(which java)` | Path must exist: `ls $JAVA_HOME/bin/java` |
| 7 | `saved_caches_directory` ≠ `hints_directory` in cassandra.yaml | `grep -E "saved_caches_dir\|hints_dir" /etc/cassandra/conf/cassandra.yaml` — must be different paths |
| 8 | `partitioner` explicitly set in cassandra.yaml | `grep partitioner /etc/cassandra/conf/cassandra.yaml` — must return `Murmur3Partitioner` |
| 9 | `cassandra -version` returns version string (not Java error) | `cassandra -version` |
| 10 | `cluster_name` matches across all nodes | `grep cluster_name /etc/cassandra/conf/cassandra.yaml` |
| 11 | `listen_address` is node-specific (not `localhost`) | `grep listen_address /etc/cassandra/conf/cassandra.yaml` |
| 12 | `rpc_address` is node-specific (not `0.0.0.0` in production) | `grep rpc_address /etc/cassandra/conf/cassandra.yaml` |
| 13 | Seed node is stable and listed consistently | Same IP across all nodes' `seeds` config |
| 14 | JVM heap set correctly in `jvm-server.options` | `grep -E "^-Xms\|^-Xmx" /etc/cassandra/conf/jvm-server.options` |
| 15 | `-Xms` equals `-Xmx` | Heap sizing is fixed — both values must match |
| 16 | Firewall rules allow port 7000, 9042 intra-cluster | `sudo firewall-cmd --list-all` |
| 17 | JMX port 7199 is NOT exposed publicly | Restricted to management subnet via rich rule |
| 18 | Schema versions agree across all nodes | `nodetool describecluster` shows one UUID |
| 19 | `NetworkTopologyStrategy` used for keyspaces (not `SimpleStrategy`) | `DESCRIBE KEYSPACES` in cqlsh |
| 20 | Default `cassandra/cassandra` password changed | `ALTER USER cassandra WITH PASSWORD ...` |
| 21 | `nodetool repair` scheduled weekly (via cron) | Prevents data divergence over time |
| 22 | Snapshot rotation policy in place | Old snapshots cleared after backup verified |
| 23 | NTP synchronized across all nodes | `chronyc tracking` — clocks must agree |

### NTP Synchronization

```bash
# [Node1 - 192.168.109.156]
# [Node2 - 192.168.109.158]
# [Node3 - 192.168.109.157]

# chrony is installed by default on Rocky Linux; just enable and verify
sudo systemctl enable --now chronyd
chronyc tracking
```

### Scheduled Repair via Cron

```bash
# [Node1 - 192.168.109.156]
# Run repair every Sunday at 2 AM
echo "0 2 * * 0 cassandra /usr/bin/nodetool repair" | sudo tee -a /etc/cron.d/cassandra-repair

# [Node2 - 192.168.109.158]
echo "0 3 * * 0 cassandra /usr/bin/nodetool repair" | sudo tee -a /etc/cron.d/cassandra-repair

# [Node3 - 192.168.109.157]
echo "0 4 * * 0 cassandra /usr/bin/nodetool repair" | sudo tee -a /etc/cron.d/cassandra-repair
```

> ⚠️ **Stagger repair times across nodes** to avoid all nodes running GC-heavy repair simultaneously.

### Key Log Locations

| File | Purpose |
|------|---------|
| `/var/log/cassandra/system.log` | Main Cassandra log |
| `/var/log/cassandra/debug.log` | Verbose debug log |
| `/var/log/cassandra/gc.log` | JVM garbage collection log |
| `/var/log/cassandra/cassandra.log` | Startup/shutdown log |

```bash
# Watch live cluster log for errors
# [Node1 - 192.168.109.156]
sudo tail -f /var/log/cassandra/system.log | grep -E "ERROR|WARN|DOWN|UP"
```

---

*Guide version: 8.0 | Cassandra: 4.1.x | Rocky Linux 9 | Last updated: 2024*
