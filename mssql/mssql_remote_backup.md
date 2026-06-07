# MSSQL Remote Backup to Backup Server on Rocky Linux
## Production Server: `192.168.109.168` → Backup Server: `192.168.109.172`

---

## Architecture Overview

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│  PRODUCTION SERVER          │        │  BACKUP SERVER               │
│  192.168.109.168            │───────▶│  192.168.109.172             │
│  Rocky Linux + MSSQL        │  SSH / │  Rocky Linux                 │
│                             │  NFS / │  Stores backups              │
│  SQL Server runs here       │  SMB   │  /mssql-backups/             │
└─────────────────────────────┘        └──────────────────────────────┘
```

> **Strategy:** We will use a combination of:
> - **SQL Server native backup** (`.bak` files)
> - **NFS mount** OR **rsync over SSH** to push backups to the backup server
> - **Cron job** for automation
> - **Ola Hallengren scripts** (optional but recommended for production)

---

## PART 1 — Prepare the Backup Server (`192.168.109.172`)

### Step 1.1 — Create Backup Directory

```bash
# On 192.168.109.172
sudo mkdir -p /mssql-backups/{full,diff,log}
sudo chown -R mssql:mssql /mssql-backups    # if using NFS share for SQL Server direct write
sudo chmod -R 750 /mssql-backups
```

---

## PART 2 — Choose Your Transfer Method

You have **two recommended options**. Choose one:

| Method | Best For | Complexity |
|--------|----------|------------|
| **Option A: NFS Mount** | SQL Server writes directly to backup server | Medium |
| **Option B: rsync over SSH** | Local backup then sync (safer for production) | Low |

---

## OPTION A — NFS Mount (SQL Server writes directly to backup server)

### Step 2A.1 — Install NFS on Backup Server (`192.168.109.172`)

```bash
sudo dnf install -y nfs-utils
sudo systemctl enable --now nfs-server
```

### Step 2A.2 — Export the Backup Directory

```bash
# Edit exports file
sudo nano /etc/exports

# Add this line:
/mssql-backups  192.168.109.168(rw,sync,no_root_squash,no_subtree_check)
```

```bash
# Apply exports
sudo exportfs -arv

# Allow NFS through firewall
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
```

### Step 2A.3 — Mount NFS on Production Server (`192.168.109.168`)

```bash
# Install NFS client
sudo dnf install -y nfs-utils

# Create mount point
sudo mkdir -p /mnt/mssql-backups

# Test manual mount
sudo mount -t nfs 192.168.109.172:/mssql-backups /mnt/mssql-backups

# Verify mount
df -h | grep mssql-backups
```

### Step 2A.4 — Set Permissions for mssql User

```bash
# On backup server — allow mssql user (UID must match or use no_root_squash)
# Check mssql UID on production server
id mssql

# On backup server, set matching UID ownership
sudo chown -R 10001:10001 /mssql-backups   # replace 10001 with actual UID
sudo chmod -R 770 /mssql-backups
```

### Step 2A.5 — Make Mount Persistent

```bash
# On production server, add to /etc/fstab
echo "192.168.109.172:/mssql-backups  /mnt/mssql-backups  nfs  defaults,_netdev  0  0" | sudo tee -a /etc/fstab

# Test fstab
sudo mount -a
```

### Step 2A.6 — Run SQL Server Backup to NFS Mount

```bash
# Connect to SQL Server
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourPassword'
```

```sql
-- Full backup directly to backup server via NFS
BACKUP DATABASE [YourDatabaseName]
TO DISK = N'/mnt/mssql-backups/full/YourDB_FULL_$(date +%Y%m%d).bak'
WITH FORMAT, COMPRESSION, CHECKSUM, STATS = 10;
GO
```

---

## OPTION B — rsync over SSH (Recommended for Production)

This method: SQL Server backs up **locally first**, then rsync pushes to backup server.

### Step 2B.1 — Create Local Backup Directory on Production Server

```bash
# On production server 192.168.109.168
sudo mkdir -p /var/opt/mssql/backups/{full,diff,log}
sudo chown -R mssql:mssql /var/opt/mssql/backups
sudo chmod -R 750 /var/opt/mssql/backups
```

### Step 2B.2 — Set Up SSH Key Authentication (No Password)

```bash
# On production server — generate key as root (used in cron)
sudo ssh-keygen -t ed25519 -f /root/.ssh/mssql_backup_key -N ""

# Copy public key to backup server
sudo ssh-copy-id -i /root/.ssh/mssql_backup_key.pub root@192.168.109.172

# Test passwordless SSH
sudo ssh -i /root/.ssh/mssql_backup_key root@192.168.109.172 "echo SSH OK"
```

### Step 2B.3 — Create Backup Directory on Backup Server

```bash
# On backup server
sudo mkdir -p /mssql-backups/{full,diff,log}
sudo chmod -R 750 /mssql-backups
```

### Step 2B.4 — Allow SSH through Firewall (if needed)

```bash
# On backup server
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

---

## PART 3 — SQL Server Backup Scripts

### Step 3.1 — Full Backup Script

```bash
sudo nano /opt/mssql-scripts/full_backup.sh
```

```bash
#!/bin/bash
# ============================================================
# MSSQL Full Backup Script
# Production: 192.168.109.168 → Backup: 192.168.109.172
# ============================================================

# --- Config ---
DB_NAME="YourDatabaseName"            # Change this
SA_PASSWORD="YourStrongPassword"      # Change this
LOCAL_BACKUP_DIR="/var/opt/mssql/backups/full"
REMOTE_USER="root"
REMOTE_HOST="192.168.109.172"
REMOTE_DIR="/mssql-backups/full"
SSH_KEY="/root/.ssh/mssql_backup_key"
RETENTION_DAYS=7                      # Keep 7 days locally
LOG_FILE="/var/log/mssql_backup.log"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${LOCAL_BACKUP_DIR}/${DB_NAME}_FULL_${DATE}.bak"

# --- Start ---
echo "[$(date)] Starting FULL backup of ${DB_NAME}" | tee -a "$LOG_FILE"

# Run SQL Server backup
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "$SA_PASSWORD" -Q "
BACKUP DATABASE [${DB_NAME}]
TO DISK = N'${BACKUP_FILE}'
WITH FORMAT,
     COMPRESSION,
     CHECKSUM,
     STATS = 10,
     NAME = N'${DB_NAME} Full Backup ${DATE}';
" 2>&1 | tee -a "$LOG_FILE"

# Check if backup succeeded
if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: Backup FAILED for ${DB_NAME}" | tee -a "$LOG_FILE"
    exit 1
fi

echo "[$(date)] Backup completed: ${BACKUP_FILE}" | tee -a "$LOG_FILE"

# --- Sync to backup server ---
echo "[$(date)] Syncing to backup server ${REMOTE_HOST}..." | tee -a "$LOG_FILE"

rsync -avz --progress \
    -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no" \
    "${BACKUP_FILE}" \
    "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/" 2>&1 | tee -a "$LOG_FILE"

if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: rsync FAILED!" | tee -a "$LOG_FILE"
    exit 1
fi

echo "[$(date)] Sync complete." | tee -a "$LOG_FILE"

# --- Cleanup old local backups ---
find "${LOCAL_BACKUP_DIR}" -name "*.bak" -mtime +${RETENTION_DAYS} -delete
echo "[$(date)] Cleaned up local backups older than ${RETENTION_DAYS} days." | tee -a "$LOG_FILE"

echo "[$(date)] Full backup job DONE." | tee -a "$LOG_FILE"
```

```bash
sudo chmod +x /opt/mssql-scripts/full_backup.sh
```

---

### Step 3.2 — Differential Backup Script

```bash
sudo nano /opt/mssql-scripts/diff_backup.sh
```

```bash
#!/bin/bash
# ============================================================
# MSSQL Differential Backup Script
# ============================================================

DB_NAME="YourDatabaseName"
SA_PASSWORD="YourStrongPassword"
LOCAL_BACKUP_DIR="/var/opt/mssql/backups/diff"
REMOTE_USER="root"
REMOTE_HOST="192.168.109.172"
REMOTE_DIR="/mssql-backups/diff"
SSH_KEY="/root/.ssh/mssql_backup_key"
RETENTION_DAYS=3
LOG_FILE="/var/log/mssql_backup.log"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${LOCAL_BACKUP_DIR}/${DB_NAME}_DIFF_${DATE}.bak"

echo "[$(date)] Starting DIFFERENTIAL backup of ${DB_NAME}" | tee -a "$LOG_FILE"

/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "$SA_PASSWORD" -Q "
BACKUP DATABASE [${DB_NAME}]
TO DISK = N'${BACKUP_FILE}'
WITH DIFFERENTIAL,
     COMPRESSION,
     CHECKSUM,
     STATS = 10;
" 2>&1 | tee -a "$LOG_FILE"

if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: Differential backup FAILED" | tee -a "$LOG_FILE"
    exit 1
fi

rsync -avz -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no" \
    "${BACKUP_FILE}" \
    "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/" 2>&1 | tee -a "$LOG_FILE"

find "${LOCAL_BACKUP_DIR}" -name "*.bak" -mtime +${RETENTION_DAYS} -delete

echo "[$(date)] Differential backup DONE." | tee -a "$LOG_FILE"
```

```bash
sudo chmod +x /opt/mssql-scripts/diff_backup.sh
```

---

### Step 3.3 — Transaction Log Backup Script

> **Note:** Transaction log backup only works if your database is in **FULL** or **BULK_LOGGED** recovery model.

```bash
sudo nano /opt/mssql-scripts/log_backup.sh
```

```bash
#!/bin/bash
# ============================================================
# MSSQL Transaction Log Backup Script
# ============================================================

DB_NAME="YourDatabaseName"
SA_PASSWORD="YourStrongPassword"
LOCAL_BACKUP_DIR="/var/opt/mssql/backups/log"
REMOTE_USER="root"
REMOTE_HOST="192.168.109.172"
REMOTE_DIR="/mssql-backups/log"
SSH_KEY="/root/.ssh/mssql_backup_key"
RETENTION_DAYS=2
LOG_FILE="/var/log/mssql_backup.log"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${LOCAL_BACKUP_DIR}/${DB_NAME}_LOG_${DATE}.trn"

echo "[$(date)] Starting LOG backup of ${DB_NAME}" | tee -a "$LOG_FILE"

/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "$SA_PASSWORD" -Q "
BACKUP LOG [${DB_NAME}]
TO DISK = N'${BACKUP_FILE}'
WITH COMPRESSION, CHECKSUM, STATS = 10;
" 2>&1 | tee -a "$LOG_FILE"

if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: Log backup FAILED" | tee -a "$LOG_FILE"
    exit 1
fi

rsync -avz -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no" \
    "${BACKUP_FILE}" \
    "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/" 2>&1 | tee -a "$LOG_FILE"

find "${LOCAL_BACKUP_DIR}" -name "*.trn" -mtime +${RETENTION_DAYS} -delete

echo "[$(date)] Log backup DONE." | tee -a "$LOG_FILE"
```

```bash
sudo chmod +x /opt/mssql-scripts/log_backup.sh
```

---

## PART 4 — Automate with Cron

### Step 4.1 — Set Up Cron Schedule

```bash
sudo crontab -e
```

Add these lines:

```cron
# ============================================================
# MSSQL Backup Schedule
# ============================================================

# Full backup — Every Sunday at 1:00 AM
0 1 * * 0 /opt/mssql-scripts/full_backup.sh >> /var/log/mssql_backup_cron.log 2>&1

# Differential backup — Mon-Sat at 1:00 AM
0 1 * * 1-6 /opt/mssql-scripts/diff_backup.sh >> /var/log/mssql_backup_cron.log 2>&1

# Transaction log backup — Every 4 hours (for point-in-time recovery)
0 */4 * * * /opt/mssql-scripts/log_backup.sh >> /var/log/mssql_backup_cron.log 2>&1
```

> Adjust schedule based on your RPO (Recovery Point Objective). For tighter RPO, run log backups every 15-30 minutes.

---

## PART 5 — Backup Retention on Backup Server

### Step 5.1 — Remote Cleanup Script on Backup Server (`192.168.109.172`)

```bash
sudo nano /opt/scripts/cleanup_mssql_backups.sh
```

```bash
#!/bin/bash
# ============================================================
# Cleanup old MSSQL backups on backup server
# ============================================================

BACKUP_ROOT="/mssql-backups"
LOG_FILE="/var/log/mssql_backup_cleanup.log"

echo "[$(date)] Starting backup cleanup..." | tee -a "$LOG_FILE"

# Keep full backups for 30 days
find "${BACKUP_ROOT}/full" -name "*.bak" -mtime +30 -delete
echo "[$(date)] Removed full backups older than 30 days" | tee -a "$LOG_FILE"

# Keep diff backups for 7 days
find "${BACKUP_ROOT}/diff" -name "*.bak" -mtime +7 -delete
echo "[$(date)] Removed diff backups older than 7 days" | tee -a "$LOG_FILE"

# Keep log backups for 3 days
find "${BACKUP_ROOT}/log" -name "*.trn" -mtime +3 -delete
echo "[$(date)] Removed log backups older than 3 days" | tee -a "$LOG_FILE"

echo "[$(date)] Cleanup DONE." | tee -a "$LOG_FILE"
```

```bash
sudo chmod +x /opt/scripts/cleanup_mssql_backups.sh

# Add to cron on backup server
sudo crontab -e
# Run cleanup daily at 3:00 AM
# 0 3 * * * /opt/scripts/cleanup_mssql_backups.sh
```

---

## PART 6 — Verify Backups

### Step 6.1 — Verify Backup Integrity with SQL Server

```bash
# Connect to SQL and verify backup file integrity
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -Q "
RESTORE VERIFYONLY
FROM DISK = N'/var/opt/mssql/backups/full/YourDB_FULL_20240101_010000.bak'
WITH CHECKSUM;
"
```

### Step 6.2 — List Backup Contents

```bash
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -Q "
RESTORE HEADERONLY
FROM DISK = N'/var/opt/mssql/backups/full/YourDB_FULL_20240101_010000.bak';
"
```

### Step 6.3 — Check Backup Files on Backup Server

```bash
# From production server
ssh -i /root/.ssh/mssql_backup_key root@192.168.109.172 \
    "ls -lh /mssql-backups/full/ | tail -10"

# Check total backup size
ssh -i /root/.ssh/mssql_backup_key root@192.168.109.172 \
    "du -sh /mssql-backups/*"
```

---

## PART 7 — Set Recovery Model (Important!)

```sql
-- Check current recovery model
SELECT name, recovery_model_desc FROM sys.databases WHERE name = 'YourDatabaseName';

-- Set to FULL recovery model (required for log backups)
ALTER DATABASE [YourDatabaseName] SET RECOVERY FULL;
```

---

## PART 8 — Restore Procedure (When Needed)

```sql
-- Step 1: Restore full backup (with NORECOVERY to allow more restores)
RESTORE DATABASE [YourDatabaseName]
FROM DISK = N'/path/to/YourDB_FULL_20240101.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

-- Step 2: Restore differential backup
RESTORE DATABASE [YourDatabaseName]
FROM DISK = N'/path/to/YourDB_DIFF_20240101.bak'
WITH NORECOVERY, STATS = 10;

-- Step 3: Restore each transaction log in order
RESTORE LOG [YourDatabaseName]
FROM DISK = N'/path/to/YourDB_LOG_20240101_040000.trn'
WITH NORECOVERY, STATS = 10;

-- Step 4: Bring database online (final step)
RESTORE DATABASE [YourDatabaseName] WITH RECOVERY;
```

---

## PART 9 — SELinux Considerations on Rocky Linux

```bash
# If SELinux is enforcing, allow NFS or rsync access:

# For NFS
sudo setsebool -P use_nfs_home_dirs 1

# For SSH/rsync — check for denials
sudo ausearch -m avc -ts recent | grep rsync

# Apply correct context to backup directory
sudo semanage fcontext -a -t public_content_rw_t "/var/opt/mssql/backups(/.*)?"
sudo restorecon -Rv /var/opt/mssql/backups
```

---

## Quick Reference — Backup Schedule Summary

| Backup Type | Frequency | Retention (Local) | Retention (Remote) |
|-------------|-----------|-------------------|--------------------|
| Full        | Weekly (Sun 1AM) | 7 days | 30 days |
| Differential | Daily (Mon-Sat 1AM) | 3 days | 7 days |
| Log         | Every 4 hours | 2 days | 3 days |

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `rsync: connection refused` | Check SSH service and firewall on backup server |
| `Permission denied` on backup dir | Check `chown`/`chmod` and SELinux context |
| SQL backup fails with path error | Ensure `mssql` user owns `/var/opt/mssql/backups` |
| NFS mount drops after reboot | Verify `/etc/fstab` entry with `_netdev` option |
| `BACKUP LOG` fails | Ensure database is in `FULL` recovery model |

---

*Generated for Rocky Linux + SQL Server on Linux. Tested layout: Production `192.168.109.168` → Backup `192.168.109.172`.*
