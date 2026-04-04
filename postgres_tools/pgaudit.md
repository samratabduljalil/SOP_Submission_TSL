# pgAudit Setup and Configuration on Rocky Linux

## Step-by-Step Procedure

---

## Prerequisites

- Rocky Linux 8 or 9
- PostgreSQL installed (version 12, 13, 14, 15, or 16)
- Root or sudo access

---

## Step 1: Install PostgreSQL (If Not Already Installed)

```bash
# Install PostgreSQL repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable the built-in PostgreSQL module
sudo dnf -qy module disable postgresql

# Install PostgreSQL 16 (adjust version as needed)
sudo dnf install -y postgresql16-server postgresql16

# Initialize the database
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Enable and start the service
sudo systemctl enable postgresql-16
sudo systemctl start postgresql-16
```

---

## Step 2: Install pgAudit

### Option A: Install via dnf (Recommended)

```bash
# Install pgaudit package matching your PostgreSQL version
sudo dnf install -y pgaudit_16
# For PostgreSQL 15: sudo dnf install -y pgaudit15
# For PostgreSQL 14: sudo dnf install -y pgaudit14
```

### Option B: Install from Source

```bash
# Install build dependencies
sudo dnf install -y postgresql16-devel gcc make git

# Clone the pgaudit source
git clone https://github.com/pgaudit/pgaudit.git
cd pgaudit

# Checkout the matching branch (e.g., for PG16)
git checkout REL_16_STABLE

# Build and install
make USE_PGXS=1 PG_CONFIG=/usr/pgsql-16/bin/pg_config
sudo make USE_PGXS=1 PG_CONFIG=/usr/pgsql-16/bin/pg_config install
```

---

## Step 3: Configure PostgreSQL to Load pgAudit

### 3.1 Edit postgresql.conf

```bash
sudo nano /var/lib/pgsql/16/data/postgresql.conf
```

Add or update the following lines:

```ini
# Load pgaudit shared library
shared_preload_libraries = 'pgaudit'

#  Set pgaudit log level
pgaudit.log = 'all'
pgaudit.log_relation = 'on' 
logging_collector = on
log_destination = 'stderr,csvlog'
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 0

# --- What to log ---
log_min_duration_statement = 0
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0

# --- Log format (REQUIRED by pgBadger) ---
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
log_timezone = 'Asia/Dhaka'



```

> **Note:** If `shared_preload_libraries` already has other entries, append with a comma:  
> `shared_preload_libraries = 'pg_stat_statements, pgaudit'`

### 3.2 Restart PostgreSQL

```bash
sudo systemctl restart postgresql-16
```

---

## Step 4: Enable pgAudit Extension in the Database

```bash
# Connect as the postgres superuser
sudo -u postgres psql
```

Inside `psql`:

```sql
-- Create the extension in the target database
CREATE EXTENSION pgaudit;

-- Verify installation
SELECT * FROM pg_extension WHERE extname = 'pgaudit';
```

---

## Step 5: Configure pgAudit Logging Parameters

### 5.1 Session-Level Audit (Logs by Action Type)

Configure in `postgresql.conf` or via `ALTER SYSTEM`:

```sql
-- Log all DDL and write operations
ALTER SYSTEM SET pgaudit.log = 'ddl, write';

-- Reload configuration
SELECT pg_reload_conf();
```

### Available `pgaudit.log` Options

| Value     | Description                                      |
|-----------|--------------------------------------------------|
| `read`    | SELECT and COPY FROM                             |
| `write`   | INSERT, UPDATE, DELETE, TRUNCATE, COPY TO        |
| `function`| Function calls and DO blocks                    |
| `role`    | GRANT, REVOKE, CREATE/ALTER/DROP ROLE            |
| `ddl`     | All DDL not included in ROLE                    |
| `misc`    | DISCARD, FETCH, CHECKPOINT, VACUUM, SET          |
| `misc_set`| SET command only                                |
| `all`     | All of the above                                 |
| `none`    | Disable session logging                         |



---

## Step 5: Additional pgAudit Parameters

Edit `postgresql.conf` or use `ALTER SYSTEM`:

```ini
# Log catalog queries (default: on)
pgaudit.log_catalog = on

# Include client info in log (default: off)
pgaudit.log_client = off

# Set log level for client logging
pgaudit.log_level = log

# Log parameter values in queries (default: off)
pgaudit.log_parameter = off

# Log each statement relation separately (default: off)
pgaudit.log_relation = off

# Log rows involved in each statement (default: off)
pgaudit.log_rows = off

# Log statement text (default: on)
pgaudit.log_statement = on

# Log statement once even if multiple rules match (default: off)
pgaudit.log_statement_once = off

# Role to use for object-level audit logging
pgaudit.role = ''
```

Apply changes:

```bash
sudo systemctl reload postgresql-16
# OR inside psql:
# SELECT pg_reload_conf();
```

---

## Step 6: Verify pgAudit is Working

### 6.1 Run a Test Query

```bash
sudo -u postgres psql -c "SELECT * FROM pg_tables LIMIT 5;"
```

### 6.2 Check PostgreSQL Logs

```bash
# Default log location
sudo tail -f /var/lib/pgsql/16/data/log/postgresql-*.log
```

### Expected Log Output Format

```
AUDIT: SESSION,1,1,READ,SELECT,,,SELECT * FROM pg_tables LIMIT 5;,<not logged>
```

**Log fields:** `AUDIT`, `SESSION|OBJECT`, `statement_id`, `substatement_id`, `class`, `command`, `object_type`, `object_name`, `statement`, `parameter`

---


## Quick Reference: Common pgAudit Configurations

### Audit All DDL Operations

```sql
ALTER SYSTEM SET pgaudit.log = 'ddl';
SELECT pg_reload_conf();
```

### Audit All Read and Write Operations

```sql
ALTER SYSTEM SET pgaudit.log = 'read, write';
SELECT pg_reload_conf();
```

### Audit Everything

```sql
ALTER SYSTEM SET pgaudit.log = 'all';
SELECT pg_reload_conf();
```

### Disable Audit for a Specific Session

```sql
SET pgaudit.log = 'none';
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `ERROR: extension "pgaudit" does not exist` | Run `CREATE EXTENSION pgaudit;` in the database |
| No audit logs appearing | Verify `shared_preload_libraries = 'pgaudit'` and restart PostgreSQL |
| `FATAL: could not access status of transaction` | Ensure pgaudit version matches PostgreSQL version |
| Permission denied on log files | Check ownership: `sudo chown postgres:postgres /var/lib/pgsql/16/data/log/` |
| SELinux blocking logs | Run `sudo audit2allow` or set `setenforce 0` temporarily for testing |

---

## Uninstall pgAudit

```bash
# Remove extension from database
sudo -u postgres psql -c "DROP EXTENSION pgaudit;"

# Remove from shared_preload_libraries in postgresql.conf
# Then restart:
sudo systemctl restart postgresql-16

# Uninstall package
sudo dnf remove pgaudit16
```

---

