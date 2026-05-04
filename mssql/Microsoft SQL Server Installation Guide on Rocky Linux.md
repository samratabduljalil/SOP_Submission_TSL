# 🗄️ Microsoft SQL Server Installation Guide on Rocky Linux

> **Tested on:** Rocky Linux 8.x / 9.x  
> **SQL Server Version:** SQL Server 2022 (latest stable)  
> **Prerequisite:** Must have `sudo` or root access

---

## 📋 Table of Contents

1. [System Requirements](#1-system-requirements)
2. [Update the System](#2-update-the-system)
3. [Add Microsoft SQL Server Repository](#3-add-microsoft-sql-server-repository)
4. [Install SQL Server](#4-install-sql-server)
5. [Configure SQL Server](#5-configure-sql-server)
6. [Start & Enable SQL Server Service](#6-start--enable-sql-server-service)
7. [Install SQL Server Command-Line Tools](#7-install-sql-server-command-line-tools)
8. [Configure Firewall](#8-configure-firewall)
9. [Connect to SQL Server](#9-connect-to-sql-server)
10. [Basic Verification Commands](#10-basic-verification-commands)
11. [Common Troubleshooting](#11-common-troubleshooting)

---

## 1. System Requirements

Before starting, ensure your system meets the minimum requirements:

| Component     | Minimum Requirement         |
|---------------|-----------------------------|
| OS            | Rocky Linux 8 or 9 (x86_64)|
| RAM           | 2 GB (4 GB recommended)     |
| Disk Space    | 6 GB free                   |
| CPU           | 2 GHz or faster             |

Check your OS version:
```bash
cat /etc/os-release
```

Check available RAM:
```bash
free -h
```

Check disk space:
```bash
df -h
```

---

## 2. Update the System

Always update your system packages before any installation.

```bash
sudo dnf update -y
```

> ⏳ This may take a few minutes depending on your internet speed.

---

## 3. Add Microsoft SQL Server Repository

### For Rocky Linux 8:
```bash
sudo curl -o /etc/yum.repos.d/mssql-server.repo \
  https://packages.microsoft.com/config/rhel/8/mssql-server-2022.repo
```

### For Rocky Linux 9:
```bash
sudo curl -o /etc/yum.repos.d/mssql-server.repo \
  https://packages.microsoft.com/config/rhel/9/mssql-server-2022.repo
```

Verify the repository was added:
```bash
cat /etc/yum.repos.d/mssql-server.repo
```

---

## 4. Install SQL Server

Now install the SQL Server package:

```bash
sudo dnf install -y mssql-server
```

> ⏳ This will download and install SQL Server (~200–400 MB). Please wait.

---

## 5. Configure SQL Server

After installation, run the setup script to configure SQL Server:

```bash
sudo /opt/mssql/bin/mssql-conf setup
```

You will be prompted to:

1. **Choose an edition:**
   ```
   1) Evaluation (free, 180-day trial)
   2) Developer   (free, not for production)
   3) Express     (free, limited features)
   4) Web
   5) Standard
   6) Enterprise
   7) Enter product key
   ```
   > 💡 For testing/learning, type `2` for **Developer** edition (free).

2. **Accept the license terms:**
   ```
   Do you accept the license terms? [Yes/No]: Yes
   ```

3. **Set the SA (System Administrator) password:**
   ```
   Enter the SQL Server system administrator password: ********
   Confirm the SQL Server system administrator password: ********
   ```
   > ⚠️ **Password must be strong:** At least 8 characters, including uppercase, lowercase, numbers, and symbols. Example: `MyP@ssw0rd!`

---

## 6. Start & Enable SQL Server Service

Start the SQL Server service:
```bash
sudo systemctl start mssql-server
```

Enable SQL Server to start automatically on boot:
```bash
sudo systemctl enable mssql-server
```

Check if the service is running:
```bash
sudo systemctl status mssql-server
```

✅ You should see **`Active: active (running)`** in the output.

---

## 7. Install SQL Server Command-Line Tools

The `sqlcmd` tool lets you interact with SQL Server from the terminal.

### Step 7.1 — Add the Microsoft tools repository

**Rocky Linux 8:**
```bash
sudo curl -o /etc/yum.repos.d/msprod.repo \
  https://packages.microsoft.com/config/rhel/8/prod.repo
```

**Rocky Linux 9:**
```bash
sudo curl -o /etc/yum.repos.d/msprod.repo \
  https://packages.microsoft.com/config/rhel/9/prod.repo
```

### Step 7.2 — Install the tools

```bash
sudo dnf install -y mssql-tools18 unixODBC-devel
```

> When prompted, type `YES` to accept the license agreements.

### Step 7.3 — Add tools to your PATH

Add `sqlcmd` to your shell PATH so you can use it from anywhere:

```bash
echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bashrc
source ~/.bashrc
```

Verify `sqlcmd` is accessible:
```bash
which sqlcmd
```

---

## 8. Configure Firewall

If you need to allow remote connections to SQL Server, open port **1433**:

```bash
sudo firewall-cmd --add-port=1433/tcp --permanent
sudo firewall-cmd --reload
```

Verify the rule was added:
```bash
sudo firewall-cmd --list-ports
```

---

## 9. Connect to SQL Server

Now connect to your local SQL Server instance using `sqlcmd`:

```bash
sqlcmd -S localhost -U SA -P 'YourPassword'
```

Replace `YourPassword` with the SA password you set in Step 5.

> ⚠️ If you get a TLS/certificate error, add `-C` flag to trust the server certificate:
> ```bash
> sqlcmd -S localhost -U SA -P 'YourPassword' -C
> ```

---

## 10. Basic Verification Commands

Once connected (you'll see a `1>` prompt), run these basic SQL commands:

### Check SQL Server version:
```sql
SELECT @@VERSION;
GO
```

### List all databases:
```sql
SELECT name FROM sys.databases;
GO
```

### Create a test database:
```sql
CREATE DATABASE TestDB;
GO
```

### Use the test database:
```sql
USE TestDB;
GO
```

### Create a simple table:
```sql
CREATE TABLE Users (
    ID INT PRIMARY KEY IDENTITY(1,1),
    Name NVARCHAR(100),
    Email NVARCHAR(200)
);
GO
```

### Insert some data:
```sql
INSERT INTO Users (Name, Email) VALUES ('Alice', 'alice@example.com');
INSERT INTO Users (Name, Email) VALUES ('Bob', 'bob@example.com');
GO
```

### Query the table:
```sql
SELECT * FROM Users;
GO
```

### Exit sqlcmd:
```sql
EXIT
```

---

## 11. Common Troubleshooting

### ❌ SQL Server service fails to start
Check the error log:
```bash
sudo cat /var/opt/mssql/log/errorlog
```

Most common cause: **not enough RAM**. SQL Server requires at least 2 GB.

Check memory:
```bash
free -h
```

### ❌ Cannot connect with sqlcmd
Ensure SQL Server is running:
```bash
sudo systemctl status mssql-server
```

Check if it's listening on port 1433:
```bash
sudo ss -tlnp | grep 1433
```

### ❌ Password rejected during setup
SQL Server enforces a strong password policy. Use a password with:
- Minimum 8 characters
- At least 1 uppercase letter (A–Z)
- At least 1 lowercase letter (a–z)
- At least 1 digit (0–9)
- At least 1 special character (!@#$%...)

### ❌ Firewall blocking remote connections
Re-check firewall rules:
```bash
sudo firewall-cmd --list-all
```

Re-apply port rule if missing:
```bash
sudo firewall-cmd --add-port=1433/tcp --permanent
sudo firewall-cmd --reload
```

### 🔄 Restart SQL Server
```bash
sudo systemctl restart mssql-server
```

### 🔄 Re-run configuration
```bash
sudo /opt/mssql/bin/mssql-conf setup
```

---

## ✅ Installation Complete!

You now have a fully working **Microsoft SQL Server 2022** running on **Rocky Linux**.

| What you installed         | Location                          |
|----------------------------|-----------------------------------|
| SQL Server binaries        | `/opt/mssql/`                     |
| SQL Server data files      | `/var/opt/mssql/data/`            |
| SQL Server logs            | `/var/opt/mssql/log/`             |
| `sqlcmd` tool              | `/opt/mssql-tools18/bin/sqlcmd`   |
| Config file                | `/var/opt/mssql/mssql.conf`       |

---

> 📌 **Tip:** For a GUI-based SQL client, consider installing **Azure Data Studio** on your local machine and connecting remotely to this server.
