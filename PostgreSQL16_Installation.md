# PostgreSQL 16 Installation (RHEL/Rocky Linux 9)

## Step 1 — Install PostgreSQL 16

Add the official PostgreSQL YUM repository and install the server and contrib packages:

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf install -y postgresql16-server
sudo dnf install -y postgresql16-contrib
```

---

## Step 2 — Create a Custom Data Directory

By default, PostgreSQL stores data in `/var/lib/pgsql/16/data/`. This guide uses a custom path at `/data/pgsql/16/data/`.

```bash
mkdir -p /data/pgsql/16/data/
chown -R postgres:postgres /data/pgsql/16/data/
chmod 700 /data/pgsql/16/data/
```

> **Why 700?** PostgreSQL requires that the data directory is only accessible by the `postgres` user for security reasons.

---

## Step 3 — Update the systemd Service File

Tell the PostgreSQL service where to find the new data directory by editing the environment variable in the service unit file:

```bash
nano /usr/lib/systemd/system/postgresql-16.service
```

Locate the line that sets `PGDATA` (inside the `[Service]` section) and update it to point to your custom directory:

```ini
Environment=PGDATA=/data/pgsql/16/data/
```

After saving, reload the systemd daemon to apply the change:

```bash
sudo systemctl daemon-reload
```

---

## Step 4 — Create a WAL Archive Directory

Set up a directory for Write-Ahead Log (WAL) archiving:

```bash
mkdir -p /log/archive/
chown -R postgres:postgres /log/archive/
chmod 700 /log/archive/
```

> **Note:** To activate archiving, you will also need to configure `archive_mode`, `archive_command`, and `wal_level` in `postgresql.conf` after initialization.

---

## Step 5 — Initialize the Database Cluster

Run `initdb` as the `postgres` user to initialize the database cluster in the custom data directory:

```bash
sudo -u postgres /usr/pgsql-16/bin/initdb -D /data/pgsql/16/data/
```

---

## Step 6 — Enable and Start the Service

Enable PostgreSQL to start on boot, then start the service:

```bash
sudo systemctl enable postgresql-16
sudo systemctl start postgresql-16
```

Verify the service is running:

```bash
sudo systemctl status postgresql-16
```

---

## Step 7 — Connect to PostgreSQL

Switch to the `postgres` system user and open the interactive `psql` shell:

```bash
sudo -i -u postgres
psql
```

You should see the PostgreSQL prompt:

```
psql (16.x)
Type "help" for help.

postgres=#
```
.
---
ntation](https://www.postgresql.org/docs/16/)
- [PostgreSQL YUM Repository](https://yum.postgresql.org/)
