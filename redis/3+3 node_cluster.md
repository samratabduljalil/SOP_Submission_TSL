# Redis Cluster Setup Guide — Rocky Linux
### 6-Node Cluster | 3 Primary + 3 Replica | Rocky Linux 8/9

---

## Cluster Topology

| Role    | Node   | IP Address      | Port |
|---------|--------|-----------------|------|
| Primary | node1  | 192.168.109.128 | 7000 |
| Primary | node2  | 192.168.109.159 | 7000 |
| Primary | node3  | 192.168.109.160 | 7000 |
| Replica | node1r | 192.168.109.161 | 7000 |
| Replica | node2r | 192.168.109.162 | 7000 |
| Replica | node3r | 192.168.109.163 | 7000 |

> **Run all steps on every node unless stated otherwise.**

> **Note on service:** `dnf install redis` already creates `redis.service` which runs:
> `/usr/bin/redis-server /etc/redis/redis.conf --daemonize no --supervised systemd`
> We edit `/etc/redis/redis.conf` directly and use `systemctl start redis` — **no custom service file needed.**

---

## Phase 1 — System Preparation (All 6 Nodes)

### 1.1 Update System & Install Dependencies

```bash
sudo dnf update -y
sudo dnf install -y epel-release
sudo dnf install -y gcc make tcl wget curl tar openssl openssl-devel
```

### 1.2 Set Hostnames (Run on each node with its own name)

```bash
# On node1 primary:
sudo hostnamectl set-hostname node1-primary

# On node2 primary:
sudo hostnamectl set-hostname node2-primary

# On node3 primary:
sudo hostnamectl set-hostname node3-primary

# On node1 replica:
sudo hostnamectl set-hostname node1-replica

# On node2 replica:
sudo hostnamectl set-hostname node2-replica

# On node3 replica:
sudo hostnamectl set-hostname node3-replica
```

### 1.3 Add /etc/hosts Entries (All 6 Nodes)

```bash
sudo tee -a /etc/hosts > /dev/null <<EOF
192.168.109.128 node1-primary
192.168.109.159 node2-primary
192.168.109.160 node3-primary
192.168.109.161 node1-replica
192.168.109.162 node2-replica
192.168.109.163 node3-replica
EOF
```

### 1.4 Kernel Tuning

```bash
# Disable THP (Transparent Huge Pages) — current session
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Persist THP disable via rc.local
sudo tee /etc/rc.d/rc.local > /dev/null <<'EOF'
#!/bin/bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF
sudo chmod +x /etc/rc.d/rc.local
sudo systemctl enable rc-local
sudo systemctl start rc-local

# Sysctl tuning — current session
sudo sysctl -w net.core.somaxconn=65535
sudo sysctl -w vm.overcommit_memory=1
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# Persist sysctl settings
sudo tee /etc/sysctl.d/99-redis.conf > /dev/null <<EOF
net.core.somaxconn=65535
vm.overcommit_memory=1
net.ipv4.tcp_max_syn_backlog=65535
EOF

sudo sysctl --system
```

### 1.5 Set System File Limits

```bash
sudo tee /etc/security/limits.d/redis.conf > /dev/null <<EOF
redis soft nofile 65535
redis hard nofile 65535
EOF
```

### 1.6 Open Required Ports (firewalld)

```bash
# Redis client port
sudo firewall-cmd --permanent --add-port=7000/tcp

# Redis cluster bus port (always client_port + 10000)
sudo firewall-cmd --permanent --add-port=17000/tcp

# Apply rules
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

### 1.7 Configure SELinux for Redis

```bash
# Check current SELinux mode
getenforce

# If semanage is not found, install it first:
#sudo dnf install -y policycoreutils-python-utils

# Allow Redis to listen on custom port 7000 and cluster bus 17000
sudo semanage port -a -t redis_port_t -p tcp 7000
sudo semanage port -a -t redis_port_t -p tcp 17000

# Verify
sudo semanage port -l | grep redis
```

> To temporarily set SELinux to permissive for troubleshooting (not for production):
> ```bash
> sudo setenforce 0
> ```

---

## Phase 2 — Install Redis (All 6 Nodes)

### 2.1 Install Redis via Remi Repository (Latest Stable)

```bash
# Enable Remi repo (provides Redis 7.x on Rocky Linux)
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-$(rpm -E %rhel).rpm

# Enable the Redis 7 module from Remi
sudo dnf module enable -y redis:remi-7.2

# Install Redis
sudo dnf install -y redis

# Verify installation
redis-server --version
# Expected: Redis server v=7.x.x
```

### Alternative: Install from AppStream (older version)

```bash
sudo dnf module enable -y redis:7
sudo dnf install -y redis
redis-server --version
```

### 2.2 Confirm the Existing Service & Config Path

```bash
# Confirm redis.service exists (installed by dnf)
sudo systemctl status redis

# Confirm the config path used by the service
cat /usr/lib/systemd/system/redis.service | grep ExecStart
# Output: ExecStart=/usr/bin/redis-server /etc/redis/redis.conf --daemonize no --supervised systemd

# We edit /etc/redis/redis.conf directly — no new service file needed
ls -lh /etc/redis/redis.conf
```

### 2.3 Prepare Data and Log Directories

```bash
# Create cluster data directory
sudo mkdir -p /var/lib/redis/cluster
sudo chown -R redis:redis /var/lib/redis/cluster

# Apply SELinux context to new directory
sudo semanage fcontext -a -t redis_var_lib_t "/var/lib/redis/cluster(/.*)?"
sudo restorecon -Rv /var/lib/redis/

# Verify log dir ownership (already exists after install)
sudo chown -R redis:redis /var/log/redis
sudo semanage fcontext -a -t redis_log_t "/var/log/redis(/.*)?"
sudo restorecon -Rv /var/log/redis/
```

---

## Phase 3 — Redis Cluster Configuration (All 6 Nodes)

### 3.1 Backup Default Config

```bash
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.orig
```

### 3.2 Edit /etc/redis/redis.conf

> Set `THIS_NODE_IP` to the IP of the node you are currently on, then run the block below.

```bash
# Set this variable to the current node's IP before running
THIS_NODE_IP="192.168.109.128"   # <-- Change per node

sudo tee /etc/redis/redis.conf > /dev/null <<EOF
# Network
bind 127.0.0.1 ${THIS_NODE_IP}
port 7000
protected-mode no
tcp-backlog 511

# General
# IMPORTANT: Keep daemonize no — redis.service passes --daemonize no --supervised systemd
daemonize no
loglevel notice
logfile /var/log/redis/redis.log
pidfile /var/run/redis/redis.pid

# Persistence
dir /var/lib/redis/cluster
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-rewrite-incremental-fsync yes
save 3600 1
save 300 100
save 60 10000
rdb-save-incremental-fsync yes
dbfilename dump.rdb

# Cluster
cluster-enabled yes
cluster-config-file /var/lib/redis/cluster/nodes.conf
cluster-node-timeout 5000
cluster-announce-ip ${THIS_NODE_IP}
cluster-announce-port 7000
cluster-announce-bus-port 17000
cluster-require-full-coverage no
cluster-replica-no-failover no
cluster-allow-reads-when-down no

# Security
requirepass YourStrongPassword123!
masterauth YourStrongPassword123!

# Memory
maxmemory 2gb
maxmemory-policy allkeys-lru

# Performance
hz 15
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes

# Disable dangerous commands (recommended)
rename-command FLUSHALL ""
rename-command FLUSHDB  ""
rename-command DEBUG    ""
EOF
```

> Repeat on all 6 nodes, changing `THIS_NODE_IP` each time:
> - node1 primary  → `192.168.109.128`
> - node2 primary  → `192.168.109.159`
> - node3 primary  → `192.168.109.160`
> - node1 replica  → `192.168.109.161`
> - node2 replica  → `192.168.109.162`
> - node3 replica  → `192.168.109.163`

> **Critical:** Keep `daemonize no` — the existing `redis.service` already passes `--daemonize no --supervised systemd`. Setting `daemonize yes` will break the service.

### 3.3 Start Redis

Kill any orphan redis-server process first, then start cleanly via systemctl.
This avoids the "Address already in use" error caused by leftover manual processes.

```bash
# Kill any running redis-server process (orphan from manual runs)
sudo pkill -9 redis-server 2>/dev/null
sleep 1

# Confirm port 7000 is free — should return nothing
sudo ss -tlnp | grep 7000

# Enable on boot and start
sudo systemctl enable redis
sudo systemctl start redis
sudo systemctl status redis
```

Expected status output:
```
Active: active (running)
Status: "Ready to accept connections"
```

If it still fails, check the log:
```bash
sudo tail -20 /var/log/redis/redis.log
```

### 3.4 Verify Each Node is Running

```bash
# Quick ping test
redis-cli -p 7000 -a YourStrongPassword123! ping
# Expected: PONG

# Confirm cluster mode is enabled
redis-cli -p 7000 -a YourStrongPassword123! cluster info
```

Expected output at this stage:
```
cluster_state:fail
cluster_slots_assigned:0
cluster_known_nodes:1
cluster_size:0
```

> `cluster_state:fail` is **completely normal here** — it means cluster mode is active
> but the cluster has not been formed yet. This gets resolved in Phase 4 when you run
> the `--cluster create` command. As long as you get `PONG`, this node is ready.

### 3.5 Repeat on All Remaining Nodes (node2, node3, node1r, node2r, node3r)

SSH into each node one by one and run:

```bash
# Run on node2, node3, node1r, node2r, node3r — one by one
sudo pkill -9 redis-server 2>/dev/null
sleep 1
sudo systemctl start redis
redis-cli -p 7000 -a YourStrongPassword123! ping
# Must return PONG on each node
```

All 6 nodes must return `PONG` before proceeding to Phase 4.

---

## Phase 4 — Create the Cluster (Run ONCE from node1)

> SSH into **node1 (192.168.109.128)** and run:

### 4.0 Confirm All 6 Nodes Are Up Before Creating the Cluster

```bash
for IP in 192.168.109.128 192.168.109.159 192.168.109.160 \
          192.168.109.161 192.168.109.162 192.168.109.163; do
  echo -n "$IP: "
  redis-cli -h $IP -p 7000 -a YourStrongPassword123! ping 2>/dev/null
done
# Every node must return PONG before proceeding
```

```bash
redis-cli -a YourStrongPassword123! --cluster create \
  192.168.109.128:7000 \
  192.168.109.159:7000 \
  192.168.109.160:7000 \
  192.168.109.161:7000 \
  192.168.109.162:7000 \
  192.168.109.163:7000 \
  --cluster-replicas 1
```

When prompted:
```
Can I set the above configuration? (type 'yes' to accept):
```
Type `yes` and press Enter.

Redis will automatically assign:
- node1, node2, node3 → **Primary** (16384 hash slots split equally)
- node1r, node2r, node3r → **Replica** (one per primary)

### 4.1 Verify Cluster Formation

```bash
redis-cli -p 7000 -a YourStrongPassword123! cluster info
```

Expected output (key fields):
```
cluster_enabled:1
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_known_nodes:6
cluster_size:3
```

```bash
# Full node listing with roles and slot ranges
redis-cli -p 7000 -a YourStrongPassword123! cluster nodes
```

---

## Phase 5 — Functional Verification

### 5.1 Write & Read Test

```bash
# -c enables cluster mode (auto-follows MOVED redirects)
redis-cli -c -p 7000 -a YourStrongPassword123! \
  -h 192.168.109.128 set testkey "hello-rocky-cluster"

redis-cli -c -p 7000 -a YourStrongPassword123! \
  -h 192.168.109.128 get testkey
# Expected: "hello-rocky-cluster"
```

### 5.2 Multi-Key Test with Hash Tags

```bash
# Hash tags force keys into the same slot
redis-cli -c -p 7000 -a YourStrongPassword123! \
  -h 192.168.109.128 mset "{user}:name" "Alice" "{user}:age" "30"

redis-cli -c -p 7000 -a YourStrongPassword123! \
  -h 192.168.109.128 mget "{user}:name" "{user}:age"
```

### 5.3 Full Cluster Health Check

```bash
redis-cli -a YourStrongPassword123! --cluster check 192.168.109.128:7000
# All checks must pass with [OK] messages
```

---

## Phase 6 — Failover Testing

### 6.1 Identify Node Roles and Mappings

```bash
redis-cli -p 7000 -a YourStrongPassword123! cluster nodes | \
  awk '{print $1, $2, $3, $9}' | column -t
# Columns: node-id  IP:port  flags  master-id
```

### 6.2 Simulate Primary Failure (Stop node1 Primary)

```bash
# On node1 (192.168.109.128):
sudo systemctl stop redis
```

### 6.3 Watch Automatic Failover (From node2)

```bash
# On node2 (192.168.109.159):
watch -n 1 "redis-cli -p 7000 -a YourStrongPassword123! cluster nodes | column -t"
```

Within `cluster-node-timeout` (5 seconds), `192.168.109.161` (node1 replica) will be **promoted to primary**. The old primary's flag changes to `fail`.

### 6.4 Verify Failover Success

```bash
# Cluster state must still be OK
redis-cli -p 7000 -a YourStrongPassword123! \
  -h 192.168.109.159 cluster info | grep cluster_state
# Expected: cluster_state:ok

# Data must still be readable
redis-cli -c -p 7000 -a YourStrongPassword123! \
  -h 192.168.109.159 get testkey
# Expected: "hello-rocky-cluster"
```

### 6.5 Rejoin Failed Node

```bash
# On node1 (192.168.109.128) — restart Redis:
sudo systemctl start redis

# Verify node1 rejoins as replica of 192.168.109.161 (now primary)
sleep 10
redis-cli -p 7000 -a YourStrongPassword123! cluster nodes | grep 192.168.109.128
```

### 6.6 Manual Failback (Force node1 Back to Primary — Optional)

```bash
# Trigger TAKEOVER failover from node1 (forces promotion without asking current primary)
redis-cli -p 7000 -h 192.168.109.128 -a YourStrongPassword123! cluster failover takeover
```

---

## Phase 7 — Backup & Restore

### 7.1 Manual RDB Snapshot (All Primaries)

```bash
for IP in 192.168.109.128 192.168.109.159 192.168.109.160; do
  echo "Triggering bgsave on $IP..."
  redis-cli -h $IP -p 7000 -a YourStrongPassword123! bgsave
  sleep 3
  redis-cli -h $IP -p 7000 -a YourStrongPassword123! lastsave
done
```

### 7.2 Copy Backup Files to Centralized Storage

```bash
BACKUP_DIR="/backup/redis/$(date +%Y%m%d_%H%M%S)"
sudo mkdir -p $BACKUP_DIR

for IP in 192.168.109.128 192.168.109.159 192.168.109.160; do
  SAFE_IP=$(echo $IP | tr '.' '_')
  sudo scp -i ~/.ssh/id_rsa \
    redis@${IP}:/var/lib/redis/cluster/dump.rdb \
    ${BACKUP_DIR}/dump_${SAFE_IP}.rdb
  sudo scp -i ~/.ssh/id_rsa \
    redis@${IP}:/var/lib/redis/cluster/appendonly.aof \
    ${BACKUP_DIR}/appendonly_${SAFE_IP}.aof
  echo "Backed up $IP"
done

ls -lh $BACKUP_DIR
```

### 7.3 Automated Backup Script with Cron

```bash
sudo tee /usr/local/bin/redis-cluster-backup.sh > /dev/null <<'SCRIPT'
#!/bin/bash
set -e
PASS="YourStrongPassword123!"
BACKUP_BASE="/backup/redis"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$BACKUP_BASE/$DATE"
LOG="/var/log/redis/backup.log"

mkdir -p "$BACKUP_DIR"
echo "[$(date)] Starting Redis cluster backup..." >> $LOG

PRIMARIES=(192.168.109.128 192.168.109.159 192.168.109.160)

for IP in "${PRIMARIES[@]}"; do
  SAFE_IP=$(echo $IP | tr '.' '_')
  echo "[$(date)] Saving $IP..." >> $LOG
  redis-cli -h $IP -p 7000 -a $PASS bgsave >> $LOG 2>&1
  sleep 5
  redis-cli -h $IP -p 7000 -a $PASS lastsave >> $LOG 2>&1
  scp -q -i /home/redis/.ssh/id_rsa \
    redis@${IP}:/var/lib/redis/cluster/dump.rdb \
    "$BACKUP_DIR/dump_${SAFE_IP}.rdb" >> $LOG 2>&1 || \
    echo "[$(date)] WARNING: Failed to copy RDB from $IP" >> $LOG
done

# Retain only last 7 days
find "$BACKUP_BASE" -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \; 2>/dev/null
echo "[$(date)] Backup complete: $BACKUP_DIR" >> $LOG
SCRIPT

sudo chmod +x /usr/local/bin/redis-cluster-backup.sh

# Schedule daily at 2:00 AM
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/redis-cluster-backup.sh") | crontab -

# Verify cron entry
crontab -l | grep redis
```

### 7.4 Restore from RDB Backup (Single Node)

```bash
# Step 1 — Stop Redis
sudo systemctl stop redis

# Step 2 — Safety backup of current data
sudo cp /var/lib/redis/cluster/dump.rdb /var/lib/redis/cluster/dump.rdb.bak
sudo cp /var/lib/redis/cluster/appendonly.aof /var/lib/redis/cluster/appendonly.aof.bak

# Step 3 — Copy backup RDB into place
sudo cp /backup/redis/<BACKUP_DATE>/dump_192_168_109_128.rdb \
        /var/lib/redis/cluster/dump.rdb
sudo chown redis:redis /var/lib/redis/cluster/dump.rdb
sudo chmod 660 /var/lib/redis/cluster/dump.rdb
sudo restorecon -v /var/lib/redis/cluster/dump.rdb

# Step 4 — Disable AOF temporarily for clean RDB load
sudo sed -i 's/^appendonly yes/appendonly no/' /etc/redis/redis.conf

# Step 5 — Validate RDB file
redis-check-rdb /var/lib/redis/cluster/dump.rdb
# Must show: RDB looks OK!

# Step 6 — Start Redis (loads from RDB)
sudo systemctl start redis
sleep 5

# Step 7 — Re-enable AOF from loaded data
redis-cli -p 7000 -a YourStrongPassword123! config set appendonly yes
redis-cli -p 7000 -a YourStrongPassword123! bgrewriteaof

# Step 8 — Re-enable AOF in config file
sudo sed -i 's/^appendonly no/appendonly yes/' /etc/redis/redis.conf

# Step 9 — Verify data
redis-cli -p 7000 -a YourStrongPassword123! dbsize
redis-cli -p 7000 -a YourStrongPassword123! info keyspace
```

---

## Phase 8 — Monitoring & Health Commands

### Cluster Status

```bash
redis-cli -p 7000 -a YourStrongPassword123! cluster info
redis-cli -p 7000 -a YourStrongPassword123! cluster nodes
redis-cli -p 7000 -a YourStrongPassword123! cluster shards
```

### Node Stats

```bash
# Memory
redis-cli -p 7000 -a YourStrongPassword123! info memory | \
  grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"

# Replication
redis-cli -p 7000 -a YourStrongPassword123! info replication | \
  grep -E "role|master_host|master_link_status|slave_repl_offset"

# Throughput
redis-cli -p 7000 -a YourStrongPassword123! info stats | \
  grep -E "instantaneous_ops_per_sec|total_commands_processed|connected_clients"

# Live rolling stats
redis-cli -p 7000 -a YourStrongPassword123! --stat
```

### Slot & Key Inspection

```bash
redis-cli -p 7000 -a YourStrongPassword123! cluster keyslot testkey
redis-cli -p 7000 -a YourStrongPassword123! cluster countkeysinslot 5649
redis-cli -a YourStrongPassword123! --cluster check 192.168.109.128:7000
redis-cli -a YourStrongPassword123! --cluster fix 192.168.109.128:7000
redis-cli -a YourStrongPassword123! --cluster rebalance 192.168.109.128:7000
```

### Slow Log

```bash
redis-cli -p 7000 -a YourStrongPassword123! slowlog get 10
redis-cli -p 7000 -a YourStrongPassword123! slowlog reset
redis-cli -p 7000 -a YourStrongPassword123! config set slowlog-log-slower-than 5000
```

---

## Troubleshooting Reference

| Problem | Cause | Fix |
|---|---|---|
| `Connection refused` on port 7000 | Redis not running or firewall blocking | `systemctl status redis` + `firewall-cmd --list-ports` |
| `CLUSTERDOWN` error | Not all slots assigned or nodes failed | `--cluster fix 192.168.109.128:7000` |
| Replica not syncing | `masterauth` mismatch | Ensure `masterauth` == `requirepass` on all nodes |
| Redis fails to start | `daemonize yes` set in conf | Must be `daemonize no` — service handles supervision |
| `bind: Address already in use` | Orphan redis-server process holding port 7000 | `sudo pkill -9 redis-server && sleep 1 && sudo systemctl start redis` |
| `WRONGTYPE` / slot redirect | Client not in cluster mode | Use `redis-cli -c` flag |
| THP warning in logs | THP not disabled | Re-run Phase 1.4 and reboot |
| Node stuck in `handshake` | Wrong `cluster-announce-ip` | Fix IP in conf, delete `nodes.conf`, restart |
| SELinux blocking Redis | AVC denial in audit log | Run `semanage port` commands in Phase 1.7 |
| `Permission denied` on data dir | Wrong ownership | `chown -R redis:redis /var/lib/redis` + `restorecon` |
| AOF file corrupted | Unclean shutdown | `redis-check-aof --fix /var/lib/redis/cluster/appendonly.aof` |
| RDB file corrupted | Disk error | `redis-check-rdb /var/lib/redis/cluster/dump.rdb` |
| Cluster split-brain | Network partition | Check connectivity on port 17000 between all nodes |

---

## Cluster Management Cheatsheet

```bash
# Helpful alias — add to ~/.bashrc on each node
alias rcli='redis-cli -p 7000 -a YourStrongPassword123! -c'
source ~/.bashrc

# ---- Daily Operations ----
rcli cluster info                             # Cluster health summary
rcli cluster nodes                            # All nodes + roles
rcli info all                                 # Full server info
rcli dbsize                                   # Total keys

# ---- Slot Operations ----
rcli cluster keyslot <key>                    # Find key's slot
rcli cluster countkeysinslot <slot>           # Keys in slot

# ---- Node Management ----
redis-cli -a YourStrongPassword123! --cluster add-node \
  <new_ip>:7000 192.168.109.128:7000          # Add new primary node

redis-cli -a YourStrongPassword123! --cluster add-node \
  <new_ip>:7000 192.168.109.128:7000 \
  --cluster-slave --cluster-master-id <id>   # Add as replica

redis-cli -a YourStrongPassword123! --cluster del-node \
  192.168.109.128:7000 <node-id>              # Remove node (must be slot-empty)

redis-cli -a YourStrongPassword123! --cluster reshard \
  192.168.109.128:7000                        # Reshard slots interactively

# ---- Service Management (uses existing redis.service) ----
sudo systemctl start redis                    # Start
sudo systemctl stop redis                     # Stop
sudo systemctl restart redis                  # Restart
sudo systemctl status redis                   # Status
sudo journalctl -u redis -f                   # Live systemd logs
sudo tail -f /var/log/redis/redis.log         # File logs
```

---

## Rocky Linux Specific Notes

| Topic | Detail |
|---|---|
| Package manager | `dnf` (replaces `yum`) |
| Firewall | `firewall-cmd` (replaces `ufw`) |
| SELinux | **Enforcing by default** — must configure port contexts |
| Redis repo | Use **Remi** repo for Redis 7.x; AppStream may give older version |
| Service file | Use existing **`redis.service`** from `dnf` — no custom service needed |
| Config file | Edit **`/etc/redis/redis.conf`** directly |
| daemonize | Must stay **`no`** — `redis.service` passes `--daemonize no --supervised systemd` |
| rc.local path | `/etc/rc.d/rc.local` — needs `chmod +x` and `rc-local` service enabled |
| File contexts | Run `semanage fcontext` + `restorecon` after creating new Redis directories |
| Audit log | SELinux denials: `sudo ausearch -m avc -ts recent` |

---

*Guide — Redis 7.x on Rocky Linux 8/9 | Cluster: 3 Primary + 3 Replica | Port 7000*
