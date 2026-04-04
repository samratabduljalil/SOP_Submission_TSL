# pgBadger Installation & Setup Guide — Rocky Linux

> **Prerequisites:** PostgreSQL is already installed. Run all commands as `root` or with `sudo`.

---

## 1. Install Dependencies

```bash
sudo dnf install -y epel-release
sudo dnf install -y perl perl-Text-CSV_XS perl-JSON perl-CGI
```

---

## 2. Install pgBadger

### Option A — Install via DNF (Recommended)

```bash
sudo dnf install -y pgbadger
```

### Option B — Install from Source (Latest Version)

```bash
# Download latest release
cd /tmp
curl -LO https://github.com/darold/pgbadger/archive/refs/tags/v12.2.tar.gz

# Extract
tar -xzf v12.2.tar.gz
cd pgbadger-12.2

# Install
perl Makefile.PL
make && sudo make install

# Verify installation
pgbadger --version
```

---

## 3. Configure PostgreSQL Logging

Edit your PostgreSQL configuration file:

```bash
sudo nano /data/pgsql/16/data/postgresql.conf
```

Add or update the following parameters:

```ini
# --- Logging destination ---
logging_collector = on
log_destination = 'stderr'
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 0

# --- What to log ---
log_min_duration_statement = 0       # Log ALL queries (set to e.g. 1000 for queries > 1s)
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0

# --- Log format (REQUIRED by pgBadger) ---
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
log_timezone = 'UTC'
```

> **Important:** `log_line_prefix` must match the format pgBadger expects. The value above is the recommended default.

---

## 4. Restart PostgreSQL to Apply Changes

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

---

## 5. Find Your PostgreSQL Log Directory

```bash
# Check configured log directory
sudo -u postgres psql -c "SHOW log_directory;"

# Show full log path
sudo -u postgres psql -c "SHOW data_directory;"

# List log files
ls -lh /data/pgsql/16/data/log/
```

---

## 6. Generate Your First pgBadger Report

### Basic Report (Single Log File)

```bash
pgbadger /data/pgsql/16/data/log/postgresql-*.log -o /tmp/pgbadger_report.html
```

### Incremental Report (Efficient for Large Logs)

```bash
# First run — generate base state file
pgbadger /data/pgsql/16/data/log/postgresql-*.log \
  -o /tmp/pgbadger_report.html \
  --last-parsed /tmp/pgbadger.last

# Subsequent runs — only process new log entries
pgbadger /data/pgsql/16/data/log/postgresql-*.log \
  -o /tmp/pgbadger_report.html \
  --last-parsed /tmp/pgbadger.last
```

---

## 7. Common pgBadger Options

| Option | Description |
|--------|-------------|
| `-o <file>` | Output HTML report file |
| `-f stderr` | Log format (stderr, syslog, csv, jsonlog) |
| `-p <prefix>` | Custom log_line_prefix pattern |
| `-b <datetime>` | Begin time filter (e.g. `2024-01-01 00:00:00`) |
| `-e <datetime>` | End time filter |
| `-d <dbname>` | Filter by database name |
| `-u <username>` | Filter by username |
| `-c <appname>` | Filter by application name |
| `--top <n>` | Number of top queries to show (default: 20) |
| `-j <n>` | Number of parallel CPU jobs |
| `-q` | Quiet mode (suppress progress output) |
| `-v` | Verbose/debug output |

---

## 8. Advanced Report Examples

### Filter by Database and Time Range

```bash
pgbadger /data/pgsql/16/data/log/postgresql-*.log -o /tmp/report.html -d cower -b "2026-03-29 00:00:00" -e "2026-03-30 23:59:59"
```

### Use Multiple CPU Cores

```bash
pgbadger /data/pgsql/16/data/log/postgresql-*.log -o /tmp/report.html -j 2
```

### CSV Log Format

```bash
# First set in postgresql.conf:
# log_destination = 'csvlog'

pgbadger /data/pgsql/16/data/log/postgresql-*.csv  -f csv  -o /tmp/report.html
```

---

## 9. Verify Everything Is Working

```bash
# 1. Check PostgreSQL is logging
sudo ls -lh /data/pgsql/16/data/log/

# 2. Run a test query to generate log entries
sudo -u postgres psql -c "SELECT pg_sleep(0.1);"

# 3. Generate a fresh report
pgbadger /data/pgsql/16/data/log/postgresql-*.log -o /tmp/test_report.html

# 4. Check the report was created
ls -lh /tmp/test_report.html

# 5. Open the report (if running locally with a browser)
xdg-open /tmp/test_report.html
```

---
## 10. Testing Lock Types in pgBadger
 
If the Lock Types section in your report is empty, you need to generate real lock contention in PostgreSQL. Open **two terminal sessions** side by side and follow the steps below.
 
### Prerequisites — Ensure Lock Logging Is Enabled
 
Edit `/data/pgsql/16/data/postgresql.conf` and confirm these are set:
 
```ini
log_lock_waits = on
deadlock_timeout = 1s    # log waits longer than this
```
 
Then restart PostgreSQL:
 
```bash
sudo systemctl restart postgresql-16
```
 
---
 
### Step 1 — Create a Test Table (run once)
 
```sql
sudo -u postgres psql -c "
  CREATE TABLE IF NOT EXISTS lock_test (id INT PRIMARY KEY, val TEXT);
  INSERT INTO lock_test VALUES (1, 'hello') ON CONFLICT DO NOTHING;
"
```
 
---
 
### Step 2 — Session A: Hold a Row Lock
 
```sql
sudo -u postgres psql -c "
BEGIN;
SELECT * FROM lock_test WHERE id = 1 FOR UPDATE;
SELECT pg_sleep(30);
COMMIT;
"
```
 
> This holds the lock for 30 seconds — keep this session open.
 
---
 
### Step 3 — Session B: Block on the Same Row (new terminal)
 
```sql
sudo -u postgres psql -c "
BEGIN;
UPDATE lock_test SET val = 'blocked' WHERE id = 1;
COMMIT;
"
```
 
> Session B will hang waiting — this is the lock wait event pgBadger needs to see.
 
---
 
### Step 4 — Verify the Lock Was Logged
 
```bash
grep -i "lock\|wait" /data/pgsql/16/data/log/postgresql-*.log | tail -20
```
 
---
 
### Step 5 — Generate the pgBadger Report
 
```bash
pgbadger /data/pgsql/16/data/log/postgresql-*.log -o /tmp/lock_report.html 
```
 
---
 
### Step 6 — View the Report
 
```bash
# Copy to web-accessible location
sudo cp /tmp/lock_report.html /var/www/html/lock_report.html
 
# Or verify it was created
ls -lh /tmp/lock_report.html
```
 
Open the report in your browser and look for the **"Lock Types"** and **"Lock Waits"** sections.
 
---


## 11. Troubleshooting

| Problem | Fix |
|---------|-----|
| `pgbadger: command not found` | Add `/usr/local/bin` to PATH or use full path `/usr/local/bin/pgbadger` |
| Report is empty | Check `log_min_duration_statement` and restart PostgreSQL |
| `invalid log line prefix` | Ensure `log_line_prefix` in `postgresql.conf` matches pgBadger's expected format |
| No log files found | Verify `logging_collector = on` and check `log_directory` path |
| Permission denied on log files | Run pgBadger as `postgres` user: `sudo -u postgres pgbadger ...` |
| Slow report generation | Use `-j <cpu_count>` for parallel processing |

---

## 12. Uninstall pgBadger (If Needed)

```bash
# If installed via DNF
sudo dnf remove pgbadger

# If installed from source
sudo rm /usr/local/bin/pgbadger
sudo rm -rf /usr/local/share/man/man1/pgbadger.1p
```

---

