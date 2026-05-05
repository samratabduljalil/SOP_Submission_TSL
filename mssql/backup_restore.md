# MS SQL Server on Rocky Linux — Full Guide
## Create Database, Tables, Insert Data, Backup & Restore

---

## Prerequisites

- Rocky Linux 8/9 with MS SQL Server installed
- `sqlcmd` installed and in PATH
- SQL Server service running

```bash
# Verify SQL Server is running
sudo systemctl status mssql-server

# Verify sqlcmd works
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "SELECT @@VERSION"
```

---

## Step 1: Create the SQL Script Files

We will use `.sql` files to avoid copy-paste issues in sqlcmd.

### 1.1 — Create the Database

```bash
cat > /tmp/01_create_database.sql << 'EOF'
CREATE DATABASE CompanyDB;
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/01_create_database.sql
```

---

### 1.2 — Create 5 Tables

```bash
cat > /tmp/02_create_tables.sql << 'EOF'
USE CompanyDB;
GO

-- Table 1: Departments
CREATE TABLE Departments (
    DeptID   INT PRIMARY KEY IDENTITY(1,1),
    DeptName NVARCHAR(100) NOT NULL,
    Location NVARCHAR(100)
);
GO

-- Table 2: Employees
CREATE TABLE Employees (
    EmpID      INT PRIMARY KEY IDENTITY(1,1),
    FirstName  NVARCHAR(100) NOT NULL,
    LastName   NVARCHAR(100) NOT NULL,
    Email      NVARCHAR(200),
    DeptID     INT FOREIGN KEY REFERENCES Departments(DeptID),
    HireDate   DATE,
    Salary     DECIMAL(10,2)
);
GO

-- Table 3: Products
CREATE TABLE Products (
    ProductID    INT PRIMARY KEY IDENTITY(1,1),
    ProductName  NVARCHAR(150) NOT NULL,
    Category     NVARCHAR(100),
    Price        DECIMAL(10,2),
    Stock        INT
);
GO

-- Table 4: Orders
CREATE TABLE Orders (
    OrderID    INT PRIMARY KEY IDENTITY(1,1),
    EmpID      INT FOREIGN KEY REFERENCES Employees(EmpID),
    OrderDate  DATE,
    TotalAmount DECIMAL(10,2),
    Status     NVARCHAR(50)
);
GO

-- Table 5: OrderDetails
CREATE TABLE OrderDetails (
    DetailID   INT PRIMARY KEY IDENTITY(1,1),
    OrderID    INT FOREIGN KEY REFERENCES Orders(OrderID),
    ProductID  INT FOREIGN KEY REFERENCES Products(ProductID),
    Quantity   INT,
    UnitPrice  DECIMAL(10,2)
);
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/02_create_tables.sql
```

---

### 1.3 — Insert 10 Rows into Each Table

```bash
cat > /tmp/03_insert_data.sql << 'EOF'
USE CompanyDB;
GO

-- Insert into Departments (10 rows)
INSERT INTO Departments (DeptName, Location) VALUES
('Human Resources', 'New York'),
('Engineering',     'San Francisco'),
('Marketing',       'Chicago'),
('Finance',         'New York'),
('Sales',           'Los Angeles'),
('IT Support',      'Austin'),
('Legal',           'Washington DC'),
('Operations',      'Dallas'),
('Research',        'Boston'),
('Customer Service','Seattle');
GO

-- Insert into Employees (10 rows)
INSERT INTO Employees (FirstName, LastName, Email, DeptID, HireDate, Salary) VALUES
('Alice',   'Johnson',  'alice@company.com',   1, '2020-01-15', 55000.00),
('Bob',     'Smith',    'bob@company.com',     2, '2019-03-22', 85000.00),
('Carol',   'Williams', 'carol@company.com',   3, '2021-07-01', 62000.00),
('David',   'Brown',    'david@company.com',   4, '2018-11-10', 72000.00),
('Eve',     'Jones',    'eve@company.com',     5, '2022-02-28', 58000.00),
('Frank',   'Garcia',   'frank@company.com',   6, '2020-09-14', 65000.00),
('Grace',   'Martinez', 'grace@company.com',   7, '2017-06-05', 90000.00),
('Henry',   'Davis',    'henry@company.com',   8, '2023-01-20', 50000.00),
('Irene',   'Wilson',   'irene@company.com',   9, '2021-04-17', 78000.00),
('James',   'Taylor',   'james@company.com',  10, '2022-08-30', 53000.00);
GO

-- Insert into Products (10 rows)
INSERT INTO Products (ProductName, Category, Price, Stock) VALUES
('Laptop Pro 15',    'Electronics',  1299.99, 50),
('Wireless Mouse',   'Accessories',    29.99, 200),
('USB-C Hub',        'Accessories',    49.99, 150),
('Monitor 27inch',   'Electronics',   399.99, 30),
('Keyboard Mech',    'Accessories',    89.99, 120),
('Webcam HD 1080p',  'Electronics',    79.99, 80),
('Desk Lamp LED',    'Office',         39.99, 60),
('Headphones Pro',   'Electronics',   199.99, 45),
('Notebook Pack',    'Stationery',      9.99, 500),
('Pen Set Premium',  'Stationery',     14.99, 300);
GO

-- Insert into Orders (10 rows)
INSERT INTO Orders (EmpID, OrderDate, TotalAmount, Status) VALUES
(1, '2024-01-10', 1329.98, 'Completed'),
(2, '2024-01-15',  449.98, 'Completed'),
(3, '2024-02-01',   79.99, 'Shipped'),
(4, '2024-02-10',  199.99, 'Completed'),
(5, '2024-02-20', 1299.99, 'Processing'),
(6, '2024-03-05',   89.99, 'Completed'),
(7, '2024-03-12',  479.98, 'Shipped'),
(8, '2024-03-25',   49.99, 'Cancelled'),
(9, '2024-04-01',  239.98, 'Completed'),
(10,'2024-04-10',   24.98, 'Processing');
GO

-- Insert into OrderDetails (10 rows)
INSERT INTO OrderDetails (OrderID, ProductID, Quantity, UnitPrice) VALUES
(1,  1, 1, 1299.99),
(1,  2, 1,   29.99),
(2,  4, 1,  399.99),
(2,  3, 1,   49.99),
(3,  6, 1,   79.99),
(4,  8, 1,  199.99),
(5,  1, 1, 1299.99),
(6,  5, 1,   89.99),
(7,  4, 1,  399.99),
(8,  3, 1,   49.99);
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/03_insert_data.sql
```

---

## Step 2: Verify the Data

```bash
cat > /tmp/04_verify.sql << 'EOF'
USE CompanyDB;
GO

SELECT 'Departments'  AS TableName, COUNT(*) AS RowCount FROM Departments  UNION ALL
SELECT 'Employees',                 COUNT(*)             FROM Employees     UNION ALL
SELECT 'Products',                  COUNT(*)             FROM Products      UNION ALL
SELECT 'Orders',                    COUNT(*)             FROM Orders        UNION ALL
SELECT 'OrderDetails',              COUNT(*)             FROM OrderDetails;
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/04_verify.sql
```

**Expected output:**

```
TableName       RowCount
--------------- --------
Departments     10
Employees       10
Products        10
Orders          10
OrderDetails    10
```

---

## Step 3: Create Backup Directory

```bash
# Create backup folder
sudo mkdir -p /var/opt/mssql/backup

# Give SQL Server permission to write there
sudo chown mssql:mssql /var/opt/mssql/backup
sudo chmod 755 /var/opt/mssql/backup
```

---

## Step 4: Full Backup

```bash
cat > /tmp/05_backup.sql << 'EOF'
USE master;
GO

BACKUP DATABASE CompanyDB
TO DISK = '/var/opt/mssql/backup/CompanyDB_FULL.bak'
WITH
    FORMAT,
    INIT,
    NAME        = 'CompanyDB Full Backup',
    DESCRIPTION = 'Full backup of CompanyDB',
    STATS       = 10;
GO

PRINT 'Full backup completed successfully!';
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/05_backup.sql
```

Verify the backup file was created:

```bash
ls -lh /var/opt/mssql/backup/
```

---

## Step 5: Differential Backup — Full Test

A differential backup only saves **changes since the last full backup** — faster and smaller than a full backup.

> ⚠️ **Golden Rule:** You can NEVER restore a differential backup alone.
> You must always restore: **Full first (NORECOVERY) → then Differential (RECOVERY).**

---

### 5.1 — Make Some Changes to the Data

First, make real changes so the differential backup has something to capture:

```bash
cat > /tmp/06_make_changes.sql << 'EOF'
USE CompanyDB;
GO

-- Add 2 new departments
INSERT INTO Departments (DeptName, Location) VALUES
('New Dept Alpha', 'Miami'),
('New Dept Beta',  'Phoenix');
GO

-- Update Alice's salary
UPDATE Employees SET Salary = 99999.00 WHERE EmpID = 1;
GO

-- Delete a product
DELETE FROM Products WHERE ProductID = 10;
GO

PRINT 'Changes applied successfully!';
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/06_make_changes.sql
```

---

### 5.2 — Take the Differential Backup

```bash
cat > /tmp/07_diff_backup.sql << 'EOF'
USE master;
GO

BACKUP DATABASE CompanyDB
TO DISK = '/var/opt/mssql/backup/CompanyDB_DIFF.bak'
WITH
    DIFFERENTIAL,
    FORMAT,
    INIT,
    NAME  = 'CompanyDB Differential Backup After Changes',
    STATS = 10;
GO

PRINT 'Differential backup completed!';
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/07_diff_backup.sql
```

---

### 5.3 — Simulate Disaster — Drop the Database

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "DROP DATABASE CompanyDB;"
```

Verify it is gone:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "SELECT name FROM sys.databases;"
```

`CompanyDB` should NOT appear in the list.

---

### 5.4 — Restore Full Backup with `NORECOVERY`

> ⚠️ `NORECOVERY` keeps the database in standby mode so the differential can be applied on top.
> Do NOT use `RECOVERY` here or the differential restore will fail.

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "
RESTORE DATABASE CompanyDB
FROM DISK = '/var/opt/mssql/backup/CompanyDB_FULL.bak'
WITH NORECOVERY, REPLACE, STATS = 10;
"
```

---

### 5.5 — Apply Differential Backup with `RECOVERY`

> ✅ `RECOVERY` on this final step brings the database fully online.

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "
RESTORE DATABASE CompanyDB
FROM DISK = '/var/opt/mssql/backup/CompanyDB_DIFF.bak'
WITH RECOVERY, STATS = 10;
"
```

---

### 5.6 — Verify the Differential Restore

```bash
cat > /tmp/08_verify_diff.sql << 'EOF'
USE CompanyDB;
GO

-- Departments should be 12 (10 original + 2 new)
-- Products should be 9 (1 deleted)
SELECT 'Departments' AS TableName, COUNT(*) AS RowCount FROM Departments UNION ALL
SELECT 'Products',                 COUNT(*)             FROM Products;
GO

-- Alice salary should be 99999.00 (was updated)
SELECT FirstName, LastName, Salary
FROM Employees
WHERE EmpID = 1;
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/08_verify_diff.sql
```

**Expected output:**

```
TableName     RowCount
------------- --------
Departments   12        ← 10 original + 2 new ✅
Products      9         ← 1 deleted ✅

FirstName  LastName  Salary
---------- --------- ----------
Alice      Johnson   99999.00  ← updated salary ✅
```

If you see these values — your differential backup and restore worked perfectly! ✅

---

### NORECOVERY vs RECOVERY — Summary

| Step | Option | Meaning |
|---|---|---|
| Full restore (not final) | `NORECOVERY` | Keep DB in standby, more backups coming |
| Differential restore (final) | `RECOVERY` | This is the last backup, bring DB online |
| Full restore only (no diff) | `RECOVERY` | Bring DB online immediately |

---

## Step 6: (Optional) Transaction Log Backup

```bash
cat > /tmp/09_log_backup.sql << 'EOF'
USE master;
GO

BACKUP LOG CompanyDB
TO DISK = '/var/opt/mssql/backup/CompanyDB_LOG.trn'
WITH
    FORMAT,
    INIT,
    NAME  = 'CompanyDB Log Backup',
    STATS = 10;
GO

PRINT 'Log backup completed!';
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/09_log_backup.sql
```

---

## Step 7: Simulate Data Loss (for Restore Test)

```bash
# Drop the database to simulate disaster
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "DROP DATABASE CompanyDB;"
```

Verify it's gone:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "SELECT name FROM sys.databases;"
```

`CompanyDB` should NOT appear in the list.

---

## Step 8: Restore the Database

```bash
cat > /tmp/10_restore.sql << 'EOF'
USE master;
GO

-- Kill any existing connections (just in case)
ALTER DATABASE CompanyDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

RESTORE DATABASE CompanyDB
FROM DISK = '/var/opt/mssql/backup/CompanyDB_FULL.bak'
WITH
    REPLACE,
    RECOVERY,
    STATS = 10;
GO

-- Bring back to multi-user mode
ALTER DATABASE CompanyDB SET MULTI_USER;
GO

PRINT 'Database restored successfully!';
GO
EOF
```

Run it:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/10_restore.sql
```

---

## Step 9: Verify Restore Was Successful

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/04_verify.sql
```

You should see all 5 tables with 10 rows each — confirming the restore was successful.

---

## Step 10: Automate Backup with Cron (Optional)

Schedule automatic nightly backups at 2:00 AM:

```bash
# Create automated backup script
cat > /usr/local/bin/mssql_backup.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/opt/mssql/backup"
PASSWORD="YourPassword"

sqlcmd -S localhost -U SA -P "$PASSWORD" -C -Q "
BACKUP DATABASE CompanyDB
TO DISK = '${BACKUP_DIR}/CompanyDB_${DATE}.bak'
WITH FORMAT, INIT, NAME = 'Auto Backup ${DATE}', STATS = 10;
"

echo "Backup completed: CompanyDB_${DATE}.bak"

# Keep only last 7 days of backups
find $BACKUP_DIR -name "*.bak" -mtime +7 -delete
EOF

# Make it executable
chmod +x /usr/local/bin/mssql_backup.sh

# Schedule with cron (runs every night at 2:00 AM)
echo "0 2 * * * root /usr/local/bin/mssql_backup.sh >> /var/log/mssql_backup.log 2>&1" \
  | sudo tee /etc/cron.d/mssql_backup
```

---

## Step 11: Remote Backup & Restore

There are **3 methods** to take remote backups depending on your setup.

---

### Method 1: 🏆 sqlcmd from a Remote Machine (Recommended)

Run sqlcmd on your **local/admin machine** pointing to the remote SQL Server.
The backup file is saved on the **remote server's disk**.

```
[Your Local Machine] ──sqlcmd──▶ [Rocky Linux SQL Server :1433]
                                        │
                                        └── saves .bak to remote disk
```

**Prerequisites on remote server — open firewall port:**

```bash
sudo firewall-cmd --zone=public --add-port=1433/tcp --permanent
sudo firewall-cmd --reload
```

**Take Full Backup remotely:**

```bash
# Run this from YOUR local machine
sqlcmd -S 192.168.1.100 -U SA -P 'YourPassword' -C -Q "
BACKUP DATABASE CompanyDB
TO DISK = '/var/opt/mssql/backup/CompanyDB_REMOTE_FULL.bak'
WITH FORMAT, INIT, NAME = 'Remote Full Backup', STATS = 10;
"
```

**Take Differential Backup remotely:**

```bash
sqlcmd -S 192.168.1.100 -U SA -P 'YourPassword' -C -Q "
BACKUP DATABASE CompanyDB
TO DISK = '/var/opt/mssql/backup/CompanyDB_REMOTE_DIFF.bak'
WITH DIFFERENTIAL, FORMAT, INIT, NAME = 'Remote Diff Backup', STATS = 10;
"
```

> Replace `192.168.1.100` with your remote server IP or hostname.

---

### Method 2: 📁 Backup to a Network Share (UNC Path)

SQL Server can write the backup directly to a **network shared folder (NFS/SMB)**.

```
[Rocky Linux SQL Server] ──writes .bak──▶ [Network Share \\backup-server\backups]
```

**Step 1 — On the backup/NAS server, create and share a folder:**

```bash
# On backup server (example: another Linux machine)
sudo mkdir -p /mnt/backups
sudo chmod 777 /mnt/backups

# Install and configure Samba
sudo dnf install -y samba samba-common
sudo systemctl enable --now smb nmb
```

Add to `/etc/samba/smb.conf`:

```ini
[backups]
   path = /mnt/backups
   browsable = yes
   writable = yes
   guest ok = yes
   read only = no
```

**Step 2 — On Rocky Linux SQL Server, mount the share:**

```bash
sudo mkdir -p /mnt/netbackup

# Mount the network share
sudo mount -t cifs //192.168.1.200/backups /mnt/netbackup \
  -o username=guest,password=,uid=mssql,gid=mssql

# Give mssql user access
sudo chown mssql:mssql /mnt/netbackup
```

**Step 3 — Backup directly to the network path:**

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "
BACKUP DATABASE CompanyDB
TO DISK = '/mnt/netbackup/CompanyDB_NET_FULL.bak'
WITH FORMAT, INIT, NAME = 'Network Full Backup', STATS = 10;
"
```

**Make the mount permanent** — add to `/etc/fstab`:

```bash
echo "//192.168.1.200/backups /mnt/netbackup cifs username=guest,password=,uid=mssql,gid=mssql 0 0" \
  | sudo tee -a /etc/fstab
```

---

### Method 3: 📤 Backup Locally then Copy to Remote (SCP/rsync)

Take the backup locally first, then transfer the `.bak` file to a remote server.

```
[Rocky Linux SQL Server]
        │
        ├── 1. Backup to local disk → /var/opt/mssql/backup/CompanyDB.bak
        └── 2. SCP/rsync .bak ──▶ [Remote Backup Server]
```

**Step 1 — Take local backup as normal:**

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "
BACKUP DATABASE CompanyDB
TO DISK = '/var/opt/mssql/backup/CompanyDB_FULL.bak'
WITH FORMAT, INIT, NAME = 'Full Backup', STATS = 10;
"
```

**Step 2 — Copy to remote server using SCP:**

```bash
# Copy to remote backup server
scp /var/opt/mssql/backup/CompanyDB_FULL.bak \
    user@192.168.1.200:/home/user/backups/
```

**Step 3 — Or use rsync for efficient sync (only transfers changes):**

```bash
rsync -avz --progress \
  /var/opt/mssql/backup/ \
  user@192.168.1.200:/home/user/backups/
```

**Automate with cron — backup + transfer every night:**

```bash
cat > /usr/local/bin/mssql_remote_backup.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
LOCAL_DIR="/var/opt/mssql/backup"
REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.200"
REMOTE_DIR="/home/backupuser/mssql-backups"
PASSWORD="YourPassword"

# Step 1: Take local backup
sqlcmd -S localhost -U SA -P "$PASSWORD" -C -Q "
BACKUP DATABASE CompanyDB
TO DISK = '${LOCAL_DIR}/CompanyDB_${DATE}.bak'
WITH FORMAT, INIT, NAME = 'Auto Remote Backup ${DATE}', STATS = 10;
"

# Step 2: Transfer to remote server
rsync -avz --progress \
  ${LOCAL_DIR}/CompanyDB_${DATE}.bak \
  ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/

# Step 3: Clean local backups older than 3 days (remote keeps 30 days)
find $LOCAL_DIR -name "*.bak" -mtime +3 -delete

echo "[$(date)] Backup and transfer completed: CompanyDB_${DATE}.bak"
EOF

chmod +x /usr/local/bin/mssql_remote_backup.sh

# Schedule nightly at 1:00 AM
echo "0 1 * * * root /usr/local/bin/mssql_remote_backup.sh >> /var/log/mssql_remote_backup.log 2>&1" \
  | sudo tee /etc/cron.d/mssql_remote_backup
```

---

### Remote Restore — Copy .bak Back and Restore

When disaster strikes and you need to restore from a remote backup:

**Step 1 — Copy the `.bak` from remote server back:**

```bash
# Pull the backup file from remote server
scp user@192.168.1.200:/home/user/backups/CompanyDB_FULL.bak \
    /var/opt/mssql/backup/

# Fix permissions so SQL Server can read it
sudo chown mssql:mssql /var/opt/mssql/backup/CompanyDB_FULL.bak
```

**Step 2 — Restore normally:**

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -Q "
RESTORE DATABASE CompanyDB
FROM DISK = '/var/opt/mssql/backup/CompanyDB_FULL.bak'
WITH REPLACE, RECOVERY, STATS = 10;
"
```

**Step 3 — Verify:**

```bash
sqlcmd -S localhost -U SA -P 'YourPassword' -C -i /tmp/04_verify.sql
```

---

### Setup Passwordless SSH (for automated rsync/SCP)

For automated scripts to work without password prompts:

```bash
# On the SQL Server machine — generate SSH key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Copy the key to remote backup server
ssh-copy-id backupuser@192.168.1.200

# Test — should connect without password
ssh backupuser@192.168.1.200 "echo Connection OK"
```

---

### Remote Backup Methods — Comparison

| Method | Backup Saved On | Best For | Complexity |
|---|---|---|---|
| **sqlcmd remote** | Remote SQL Server disk | Admin tasks from Windows/Linux | ⭐ Easy |
| **Network Share** | NAS / shared folder | Centralized backup server | ⭐⭐ Medium |
| **SCP / rsync** | Any remote server | Offsite / cloud backup | ⭐⭐ Medium |
| **All 3 combined** | Multiple locations | Production 3-2-1 strategy | ⭐⭐⭐ Advanced |

---

### 🏭 Production 3-2-1 Backup Rule

> **3** copies of data
> **2** different storage types
> **1** offsite / remote location

```
CompanyDB Backup Strategy
│
├── Copy 1 → Local disk           (/var/opt/mssql/backup/)
├── Copy 2 → Network Share        (//nas-server/backups/)
└── Copy 3 → Remote/Offsite server (rsync to 192.168.1.200 or cloud)
```

This ensures even if the SQL Server machine is completely destroyed, you can restore from the remote copy.

---

## Quick Reference — All Commands

| Task | Command |
|---|---|
| Run a `.sql` file | `sqlcmd -S localhost -U SA -P 'Pass' -C -i /path/file.sql` |
| Quick inline query | `sqlcmd -S localhost -U SA -P 'Pass' -C -Q "SELECT 1"` |
| Full backup | `BACKUP DATABASE db TO DISK = '/path/file.bak' WITH FORMAT, INIT` |
| Differential backup | Add `WITH DIFFERENTIAL` |
| Log backup | `BACKUP LOG db TO DISK = '/path/file.trn'` |
| Restore full only | `RESTORE DATABASE db FROM DISK = 'full.bak' WITH REPLACE, RECOVERY` |
| Restore full + diff | Full with `NORECOVERY`, then diff with `RECOVERY` |
| Remote backup via sqlcmd | `sqlcmd -S REMOTE_IP -U SA -P 'Pass' -C -Q "BACKUP DATABASE ..."` |
| Copy backup to remote | `scp /var/opt/mssql/backup/file.bak user@remote:/path/` |
| Sync backup folder remotely | `rsync -avz /var/opt/mssql/backup/ user@remote:/path/` |
| Mount network share | `mount -t cifs //server/share /mnt/netbackup -o uid=mssql` |
| Check backup history | `SELECT * FROM msdb.dbo.backupset ORDER BY backup_finish_date DESC` |
| List all databases | `SELECT name FROM sys.databases` |
| Open firewall for SQL | `firewall-cmd --add-port=1433/tcp --permanent && firewall-cmd --reload` |

---

## Backup Types Summary

| Type | When to Use | Speed | Size |
|---|---|---|---|
| **Full** | Weekly / initial | Slow | Large |
| **Differential** | Daily | Medium | Medium |
| **Log** | Hourly (point-in-time recovery) | Fast | Small |

---

> ⚠️ **Security Tip:** Never hardcode passwords in scripts for production.
> Use environment variables or SQL Server credentials file instead:
> ```bash
> export MSSQL_SA_PASSWORD='YourPassword'
> sqlcmd -S localhost -U SA -P "$MSSQL_SA_PASSWORD" -C -i script.sql
> ```
