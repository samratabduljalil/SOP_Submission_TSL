# Standard Operating Procedure: pgpool-II Installation & Configuration
**Version:** 1.0  
**Platform:** RHEL 9 / AlmaLinux 9 / Rocky Linux 9  
**PostgreSQL Version:** 16  
**pgpool-II Version:** 4.5  

---

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Enable CRB Repository](#step-1-enable-crb-repository)
4. [Step 2: Install Dependencies](#step-2-install-dependencies)
5. [Step 3: Install pgpool-II](#step-3-install-pgpool-ii)
6. [Step 4: Enable and Start pgpool Service](#step-4-enable-and-start-pgpool-service)
7. [Step 5: Configure pgpool.conf](#step-5-configure-pgpoolconf)
8. [Step 6: Configure pool_hba.conf](#step-6-configure-pool_hbaconf)
9. [Step 7: Configure pg_hba.conf on PostgreSQL Servers](#step-7-configure-pg_hbaconf-on-postgresql-servers)
10. [Step 8: Create pgpool Role in PostgreSQL](#step-8-create-pgpool-role-in-postgresql)
11. [Step 9: Set Up PCP User Authentication](#step-9-set-up-pcp-user-authentication)
12. [Step 10: Configure Password Encryption](#step-10-configure-password-encryption)
13. [Step 11: Verify Setup](#step-11-verify-setup)


---

## Overview

This SOP documents the process of installing and configuring **pgpool-II** as a connection pooler and load balancer for a PostgreSQL 16 streaming replication setup with one master and one replica node.

| Component        | IP Address        | Role         |
|------------------|-------------------|--------------|
| pgpool-II server | 192.168.109.130   | Proxy/Pooler |
| PostgreSQL Master| 192.168.109.128   | Primary DB   |
| PostgreSQL Replica| 192.168.109.129  | Standby DB   |

---

## Streaming Replication Reference

> 📎 **Related SOP:** [PostgreSQL Streaming Replication Setup](https://github.com/samratabduljalil/SOP_Submission_TSL/blob/main/Streaming_Replication.md)

### Architecture Diagram

<img width="1041" height="1042" alt="streaming replication with pgpool drawio" src="https://github.com/user-attachments/assets/a9b6675b-64d0-458e-8a1e-9e3fc0de1ae2" />


*pgpool-II (192.168.109.130) sits between the client and both PostgreSQL nodes. It routes **write** queries to the Master and **read** queries to the Slave via load balancing. WAL logs stream continuously from the Master's WAL Sender to the Slave's WAL Receiver over TCP/IP. pgpool-II also performs SR health checks against both nodes to detect failures and trigger failover.*

---

## Prerequisites

- RHEL 9 (or compatible) OS on all nodes
- PostgreSQL 16 already installed and streaming replication configured [(PostgreSQL Streaming Replication Setup)](https://github.com/samratabduljalil/SOP_Submission_TSL/blob/main/Streaming_Replication.md)
- Root or sudo access on the pgpool server
- Network connectivity between all nodes

---

## Step 1: Enable CRB Repository

The CodeReady Builder (CRB) repository is required for some pgpool dependencies.

```bash
sudo dnf config-manager --set-enabled crb

# Verify CRB is enabled
sudo dnf repolist | grep crb
```

---

## Step 2: Install Dependencies

```bash
# Refresh package metadata
sudo dnf makecache

# Install libmemcached (required by pgpool)
sudo dnf install -y libmemcached-awesome
```

---

## Step 3: Install pgpool-II

```bash
# Add the official pgpool-II YUM repository for RHEL 9 / x86_64
sudo dnf install -y https://www.pgpool.net/yum/rpms/4.5/redhat/rhel-9-x86_64/pgpool-II-release-4.5-1.noarch.rpm

# Install pgpool-II for PostgreSQL 16
sudo dnf install -y pgpool-II-pg16

# Install pgpool-II extensions (required for show_pool_nodes, etc.)
sudo dnf install -y pgpool-II-pg16-extensions

# Confirm repositories are registered correctly
sudo dnf repolist
```

---

## Step 4: Enable and Start pgpool Service

```bash
# Enable pgpool to start on boot
sudo systemctl enable pgpool

# Start pgpool service
sudo systemctl start pgpool

# Verify pgpool is running
sudo systemctl status pgpool
```

---

## Step 5: Configure pgpool.conf

Edit `/etc/pgpool-II/pgpool.conf` and update the following parameters:

```bash
sudo nano /etc/pgpool-II/pgpool.conf
```

### Connection Settings

```ini
listen_addresses = '*'
port = 9999
unix_socket_directories = '/tmp'
reserved_connections = 3
listen_backlog_multiplier = 2
serialize_accept = off
```

### PCP (pgpool Control Port) Settings

```ini
pcp_listen_addresses = '*'
pcp_port = 9898
pcp_socket_dir = '/tmp'
```

### Clustering Mode

```ini
backend_clustering_mode = 'streaming_replication'
```

### Backend Node 0 — Master (Primary)

```ini
backend_hostname0 = '192.168.109.128'
backend_port0 = 5432
backend_weight0 = 0
backend_data_directory0 = '/data/pgsql/16/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_application_name0 = 'master'
```

> **Note:** `backend_weight0 = 0` means no read queries are routed to the master; all reads go to the replica.

### Backend Node 1 — Replica (Standby)

```ini
backend_hostname1 = '192.168.109.129'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/data/pgsql/16/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'
backend_application_name1 = 'replica1'
```

### Streaming Replication Health Check

```ini
sr_check_user = 'pgpool'
sr_check_password = 'strong_password'
sr_check_database = 'postgres'
```

### Health Check Settings

```ini
health_check_period = 10
health_check_timeout = 20
health_check_user = 'pgpool'
health_check_password = 'strong_password'
health_check_database = 'postgres'
```


### Password & Key File Settings

```ini
pool_passwd = 'pool_passwd'
pgpool_key = '/var/lib/pgsql/.pgpoolkey'
```

---

## Step 6: Configure pool_hba.conf

Edit `/etc/pgpool-II/pool_hba.conf` to allow connections from all relevant nodes:

```bash
sudo nano /etc/pgpool-II/pool_hba.conf
```

Add the following entries:

```
# TYPE  DATABASE    USER        ADDRESS                 METHOD
host    all         all         192.168.109.128/32      scram-sha-256
host    all         all         192.168.109.129/32      scram-sha-256
host    all         all         192.168.109.130/32      scram-sha-256
```
Restart Pgpool servers after the edit:

```bash
systemctl restart pgpool
```
---

## Step 7: Configure pg_hba.conf on PostgreSQL Servers

On **both the master and replica** PostgreSQL servers, edit `pg_hba.conf` to allow connections from the pgpool server:

```bash
sudo nano /data/pgsql/16/data/pg_hba.conf
```

Add the following entry:

```
# Allow pgpool server to connect
host    all         all         192.168.109.130/32      scram-sha-256
```

Reload PostgreSQL on both servers after the edit:

```bash
systemctl reload postgresql-16
```

---

## Step 8: Create pgpool Role in PostgreSQL

Run the following on the **master** PostgreSQL server:

```bash
sudo -u postgres psql
```

```sql
CREATE ROLE pgpool WITH LOGIN PASSWORD 'strong_password';
```

> This user is used by pgpool for health checks and streaming replication status checks.

---

## Step 9: Set Up PCP User Authentication

PCP (pgpool Control Protocol) is used for administrative commands. Add a PCP user on the **pgpool server**:

```bash
# Generate MD5 hash and append user entry to pcp.conf
echo "samrat:$(pg_md5 your_password)" >> /etc/pgpool-II/pcp.conf
```

### Verify PCP Connectivity

Run the following from any authorized node (e.g., master):

```bash
pcp_node_info -U samrat -W -h 192.168.109.130 -p 9898
```

You will be prompted for the password. A successful response lists node information.

---

## Step 10: Configure Password Encryption

pgpool stores encrypted backend passwords in `pool_passwd`. Follow these steps on the **pgpool server**:

### Step 10.1 — Create the pgpoolkey File

```bash
echo "strong_password" > ~/.pgpoolkey
chmod 600 ~/.pgpoolkey
```

> The `.pgpoolkey` file is used to encrypt/decrypt passwords stored in `pool_passwd`.

### Step 10.2 — Encrypt and Register the pgpool User Password

```bash
pg_enc -m -k ~/.pgpoolkey -u pgpool -p
```

You will be prompted to enter the password for the `pgpool` database user.

### Step 10.3 — Verify the pool_passwd File

```bash
cat /etc/pgpool-II/pool_passwd
```

You should see an entry like:

```
pgpool:AES256hash...
```

---

## Step 11: Verify Setup

Connect through pgpool and verify all backend nodes are visible:

```bash
psql -h 192.168.109.130 -p 9999 -U pgpool -d postgres -c "SHOW pool_nodes;"
```

Expected output shows both backend nodes (master and replica) with their status, roles, and weights.


<img width="1677" height="97" alt="Screenshot 2026-03-03 114555" src="https://github.com/user-attachments/assets/a26dc3b1-b87f-4a77-a2df-6191b1eae845" />



---


