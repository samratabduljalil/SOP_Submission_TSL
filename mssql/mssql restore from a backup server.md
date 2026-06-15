# MSSQL Restore Guide
## Backup Server: `192.168.109.172` → Production Server: `192.168.109.168`

---

## Overview

```
┌──────────────────────────────┐        ┌─────────────────────────────┐
│  BACKUP SERVER               │        │  PRODUCTION SERVER          │
│  192.168.109.172             │───────▶│  192.168.109.168            │
│  /mssql-backups/             │  rsync │  Rocky Linux + MSSQL        │
│  full / diff / log           │        │  Restore happens here       │
└──────────────────────────────┘        └─────────────────────────────┘
```

> **Restore Order is Critical:**
> `Full Backup` → `Differential Backup` → `Transaction Log(s)` → `RECOVERY`

---

## PART 1 — Copy Backup Files from Backup Server to Production

### Step 1.1 — Pull files from backup server using rsync

```bash
# On production server (192.168.109.168)

# Pull full backup
rsync -avz \
    -e "ssh -i /root/.ssh/mssql_backup_key -o StrictHostKeyChecking=no" \
    samrat@192.168.109.172:/mssql-backups/full/StackOverflowMini_FULL_20260607_163318.bak \
    /var/opt/mssql/backups/full/

# Pull differential backup (if needed)
rsync -avz \
    -e "ssh -i /root/.ssh/mssql_backup_key -o StrictHostKeyChecking=no" \
    samrat@192.168.109.172:/mssql-backups/diff/StackOverflowMini_DIFF_20260607_010000.bak \
    /var/opt/mssql/backups/diff/

# Pull transaction log backups (if needed)
rsync -avz \
    -e "ssh -i /root/.ssh/mssql_backup_key -o StrictHostKeyChecking=no" \
    samrat@192.168.109.172:/mssql-backups/log/ \
    /var/opt/mssql/backups/log/
```

### Step 1.2 — Fix permissions after copy

```bash
sudo chown -R mssql:mssql /var/opt/mssql/backups/
sudo chmod -R 750 /var/opt/mssql/backups/
```

### Step 1.3 — List available backups to find the right file

```bash
# List full backups
ls -lh /var/opt/mssql/backups/full/

# List differential backups
ls -lh /var/opt/mssql/backups/diff/

# List log backups
ls -lh /var/opt/mssql/backups/log/
```

---

## PART 2 — Check Backup File Before Restoring

### Step 2.1 — Read backup header (shows backup info)

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE HEADERONLY
FROM DISK = N'/var/opt/mssql/backups/full/StackOverflowMini_FULL_20260607_163318.bak';
"
```

Output shows: backup type, database name, date, LSN numbers — confirm it's the right file.

### Step 2.2 — Verify backup integrity (no corruption)

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE VERIFYONLY
FROM DISK = N'/var/opt/mssql/backups/full/StackOverflowMini_FULL_20260607_163318.bak'
WITH CHECKSUM;
"
```

> If output says `The backup set on file 1 is valid.` — you are good to proceed.

### Step 2.3 — Check file list inside backup

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE FILELISTONLY
FROM DISK = N'/var/opt/mssql/backups/full/StackOverflowMini_FULL_20260607_163318.bak';
"
```

> Note the `LogicalName` and `PhysicalName` columns — you need them if restoring to a different path.

---

## PART 3 — Restore Scenarios

---

### SCENARIO A — Restore Full Backup Only (Simplest)

> Use when: You only have a full backup and want to restore to that point.

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/full/StackOverflowMini_FULL_20260607_163318.bak'
WITH REPLACE,
     RECOVERY,
     STATS = 10;
"
```

- `REPLACE` → overwrites existing database
- `RECOVERY` → brings database online immediately
- `STATS = 10` → shows progress every 10%

---

### SCENARIO B — Full + Differential Restore

> Use when: You want to restore to the latest differential backup point.

```bash
# Step 1: Restore full backup (keep in NORECOVERY — do not bring online yet)
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/full/StackOverflowMini_FULL_20260607_163318.bak'
WITH REPLACE,
     NORECOVERY,
     STATS = 10;
"

# Step 2: Restore differential backup (bring online after this)
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/diff/StackOverflowMini_DIFF_20260607_010000.bak'
WITH NORECOVERY,
     STATS = 10;
"

# Step 3: Bring database online
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini] WITH RECOVERY;
"
```

---

### SCENARIO C — Full + Differential + Transaction Logs (Point-in-Time)

> Use when: You need to restore to a specific time (e.g. just before data was deleted).

```bash
# Step 1: Restore full backup
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/full/StackOverflowMini_FULL_20260607_163318.bak'
WITH REPLACE,
     NORECOVERY,
     STATS = 10;
"

# Step 2: Restore differential backup
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/diff/StackOverflowMini_DIFF_20260607_010000.bak'
WITH NORECOVERY,
     STATS = 10;
"

# Step 3: Restore each log file IN ORDER (repeat for each .trn file)
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE LOG [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/log/StackOverflowMini_LOG_20260607_040000.trn'
WITH NORECOVERY,
     STATS = 10;
"

/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE LOG [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/log/StackOverflowMini_LOG_20260607_080000.trn'
WITH NORECOVERY,
     STATS = 10;
"

# Step 4: Bring database online (run AFTER last log is applied)
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini] WITH RECOVERY;
"
```

---

### SCENARIO D — Point-in-Time Restore (Stop at exact time)

> Use when: You know exactly when data was lost (e.g. accidental DELETE at 14:35).

```bash
# On the last log restore, add STOPAT with the target time
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE LOG [StackOverflowMini]
FROM DISK = N'/var/opt/mssql/backups/log/StackOverflowMini_LOG_20260607_120000.trn'
WITH NORECOVERY,
     STOPAT = '2026-06-07 14:34:59';
"

# Then bring online
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini] WITH RECOVERY;
"
```

---

### SCENARIO E — Restore to a Different Database Name (Non-destructive test)

> Use when: You want to verify restore without touching the live database.

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
RESTORE DATABASE [StackOverflowMini_Test]
FROM DISK = N'/var/opt/mssql/backups/full/StackOverflowMini_FULL_20260607_163318.bak'
WITH REPLACE,
     RECOVERY,
     MOVE N'StackOverflowMini' TO N'/var/opt/mssql/data/StackOverflowMini_Test.mdf',
     MOVE N'StackOverflowMini_log' TO N'/var/opt/mssql/data/StackOverflowMini_Test_log.ldf',
     STATS = 10;
"
```

> Replace `StackOverflowMini` and `StackOverflowMini_log` with the LogicalName values from `RESTORE FILELISTONLY` output (Part 2, Step 2.3).

---

## PART 4 — Verify After Restore

### Step 4.1 — Check database is online

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
SELECT name, state_desc, recovery_model_desc
FROM sys.databases
WHERE name = 'StackOverflowMini';
"
```

Expected output:
```
name                state_desc   recovery_model_desc
StackOverflowMini   ONLINE       FULL
```

### Step 4.2 — Check row counts to confirm data is there

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
USE [StackOverflowMini];
SELECT TABLE_NAME, TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE';
"
```

### Step 4.3 — Check backup restore history

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
SELECT TOP 10
    destination_database_name,
    restore_date,
    type,
    user_name
FROM msdb.dbo.restorehistory
ORDER BY restore_date DESC;
"
```

---

## PART 5 — Automated Restore Script

> Useful for disaster recovery — restores full backup in one command.

```bash
sudo nano /opt/mssql-scripts/restore_full.sh
```

```bash
#!/bin/bash
# ============================================================
# MSSQL Full Restore Script
# ============================================================

DB_NAME="StackOverflowMini"
SA_PASSWORD="YourStrongPassword"
BACKUP_SERVER_USER="samrat"
BACKUP_SERVER_HOST="192.168.109.172"
BACKUP_SERVER_DIR="/mssql-backups/full"
SSH_KEY="/root/.ssh/mssql_backup_key"
LOCAL_RESTORE_DIR="/var/opt/mssql/backups/full"
LOG_FILE="/var/log/mssql_restore.log"

echo "[$(date)] === Starting Restore ===" | tee -a "$LOG_FILE"

# Step 1: Find latest full backup on backup server
echo "[$(date)] Finding latest backup on backup server..." | tee -a "$LOG_FILE"
LATEST=$(ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no \
    "${BACKUP_SERVER_USER}@${BACKUP_SERVER_HOST}" \
    "ls -t ${BACKUP_SERVER_DIR}/*.bak 2>/dev/null | head -1")

if [ -z "$LATEST" ]; then
    echo "[$(date)] ERROR: No backup file found on backup server!" | tee -a "$LOG_FILE"
    exit 1
fi

echo "[$(date)] Latest backup: $LATEST" | tee -a "$LOG_FILE"

# Step 2: Pull backup file
FILENAME=$(basename "$LATEST")
rsync -avz \
    -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no" \
    "${BACKUP_SERVER_USER}@${BACKUP_SERVER_HOST}:${LATEST}" \
    "${LOCAL_RESTORE_DIR}/" 2>&1 | tee -a "$LOG_FILE"

# Step 3: Fix permissions
sudo chown mssql:mssql "${LOCAL_RESTORE_DIR}/${FILENAME}"

# Step 4: Restore database
echo "[$(date)] Restoring database ${DB_NAME}..." | tee -a "$LOG_FILE"
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "$SA_PASSWORD" -No -C -Q "
RESTORE DATABASE [${DB_NAME}]
FROM DISK = N'${LOCAL_RESTORE_DIR}/${FILENAME}'
WITH REPLACE, RECOVERY, STATS = 10;
" 2>&1 | tee -a "$LOG_FILE"

if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: Restore FAILED!" | tee -a "$LOG_FILE"
    exit 1
fi

echo "[$(date)] Restore COMPLETED successfully." | tee -a "$LOG_FILE"
```

```bash
sudo chmod +x /opt/mssql-scripts/restore_full.sh
```

Run it:
```bash
sudo /opt/mssql-scripts/restore_full.sh
```

---

## Quick Reference

| Scenario | NORECOVERY on Full? | NORECOVERY on Diff? | Final Step |
|----------|--------------------|--------------------|------------|
| Full only | No (use RECOVERY) | — | Done |
| Full + Diff | Yes | No (use RECOVERY) | Done |
| Full + Diff + Logs | Yes | Yes | `WITH RECOVERY` after last log |
| Point-in-time | Yes | Yes | `STOPAT` on last log + RECOVERY |

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `SSL certificate verify failed` | Add `-No -C` flags to sqlcmd |
| `The backup set holds a backup of a database other than the existing database` | Add `WITH REPLACE` to restore command |
| `Database is in use` | Kill active connections first (see below) |
| `RESTORE cannot operate on database because it is configured for database mirroring` | Remove mirroring before restore |
| `Cannot open backup device — Permission denied` | Run `chown mssql:mssql` on the `.bak` file |

### Kill active connections before restore

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
ALTER DATABASE [StackOverflowMini] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
"
```

Then run your restore, then after restore bring back to multi-user:

```bash
/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -No -C -Q "
ALTER DATABASE [StackOverflowMini] SET MULTI_USER;
"
```

---

*Production: `192.168.109.168` | Backup Server: `192.168.109.172` | Rocky Linux + SQL Server*
