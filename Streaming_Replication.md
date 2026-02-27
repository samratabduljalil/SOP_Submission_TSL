# PostgreSQL Streaming Replication visual diagram 

<img width="1092" height="829" alt="master slave diagram" src="https://github.com/user-attachments/assets/552272a2-4f7c-47e1-b9e9-92491efec572" />




# PostgreSQL Streaming Replication Setup Guide

## Overview

This guide covers setting up **Physical Streaming Replication** between two Rocky Linux servers using PostgreSQL.

| Role   | Hostname     | IP Address       |
|--------|--------------|------------------|
| Master | rocky-master | 192.168.109.128  |
| Slave  | rocky-slave  | 192.168.109.129  |


---

## Step 1 — Configure the Master Server (`192.168.109.128`)

### 1.1 Edit `postgresql.conf`

Open the PostgreSQL configuration file:

```bash
sudo nano /data/pgsql/16/data/postgresql.conf
```

Set the following parameters:

```conf
listen_addresses = '*'
port = 5432
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 512MB
archive_mode = on
archive_command = 'rsync %p /log/archive/%f'
```

---

### 1.2 Create the WAL Archive Directory (if not already created)

```bash
sudo mkdir -p /log/archive/
sudo chown -R postgres:postgres /log/archive/
sudo chmod 700 /log/archive/
```

---

### 1.3 Edit `pg_hba.conf`

Open the host-based authentication file:

```bash
sudo nano /data/pgsql/16/data/pg_hba.conf
```

Add the following line to allow the slave to connect for replication:

```conf
host    replication     replicator      192.168.109.129/32      scram-sha-256
```

---

### 1.4 Create the Replication User

Connect to PostgreSQL and create the replication role:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replica123';
```

---

### 1.5 Restart PostgreSQL on Master

```bash
sudo systemctl restart postgresql-16
```

---

### 1.6 Open Firewall Port 5432

**Option A — Open only port 5432:**

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

**Option B — Disable the firewall entirely (not recommended for production):**

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

---

## Step 2 — Configure the Slave Server (`192.168.109.129`)

### 2.1 Stop PostgreSQL on Slave

```bash
sudo systemctl stop postgresql-16
```

---

### 2.2 Rename the Existing Data Directory

```bash
mv /data/pgsql/16/data /data/pgsql/16/data_old
```

---

### 2.3 Create a New Empty Data Directory

```bash
mkdir -p /data/pgsql/16/data
chown -R postgres:postgres /data/pgsql/16/data
chmod 700 /data/pgsql/16/data
```

---

### 2.4 Take a Base Backup from the Master

```bash
sudo -u postgres pg_basebackup \
  -h 192.168.109.128 \
  -D /data/pgsql/16/data \
  -U replicator \
  -Fp -Xs -P -R
```

| Flag  | Description                                          |
|-------|------------------------------------------------------|
| `-h`  | Master server IP                                     |
| `-D`  | Destination data directory on slave                  |
| `-U`  | Replication user                                     |
| `-Fp` | Plain format (file-based)                            |
| `-Xs` | Stream WAL during backup                             |
| `-P`  | Show progress                                        |
| `-R`  | Write `standby.signal` and `primary_conninfo` automatically |

---

### 2.5 Start PostgreSQL on Slave

```bash
sudo systemctl start postgresql-16
```

---

## Step 3 — Verify Replication on Master

Run the following query on the **master** to check replication status:

```sql
SELECT 
    application_name, 
    client_addr, 
    state, 
    sync_state, 
    write_lag, 
    flush_lag, 
    replay_lag 
FROM pg_stat_replication;
```

If replication is working, you should see the slave's IP (`192.168.109.129`) listed with `state = streaming`.

---

> ⚠️ **Important Warning**
> - **Logical replication** is possible between different PostgreSQL versions.
> - **Physical (streaming) replication** is **NOT** possible between different PostgreSQL versions. Both master and slave must run the **same version**.




## Debugging & Troubleshooting

Check the PostgreSQL log on either server for errors:

```bash
tail -n 50 /data/pgsql/16/data/log/postgresql-*.log
```

### Common Issues
| Issue | Possible Cause |
|-------|---------------|
| Slave cannot connect to master | Firewall blocking port 5432, or wrong IP in `pg_hba.conf` |
| `pg_basebackup` fails with auth error | Wrong username or password; check `pg_hba.conf` entry |
| Replication not showing in `pg_stat_replication` | PostgreSQL not restarted after config changes on master |
| `FATAL: requested WAL segment has been removed` | Increase `wal_keep_size` on master |
| Version mismatch error | Physical replication requires identical PostgreSQL versions |
