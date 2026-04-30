# MySQL Performance Tuning: Complete Parameter Reference Guide

> A production-ready reference for database administrators and backend engineers who want to squeeze maximum performance from MySQL.

---

## Table of Contents

1. [Why Parameter Tuning Matters](#why-parameter-tuning-matters)
2. [InnoDB Buffer Pool](#1-innodb-buffer-pool)
3. [InnoDB Log Settings](#2-innodb-log-settings)
4. [InnoDB I/O & Flush Settings](#3-innodb-io--flush-settings)
5. [Query Cache (MySQL 5.x)](#4-query-cache-mysql-5x)
6. [Connection Management](#5-connection-management)
7. [Thread & Cache Settings](#6-thread--cache-settings)
8. [Temporary Tables & Sort Buffers](#7-temporary-tables--sort-buffers)
9. [Table & File Descriptor Cache](#8-table--file-descriptor-cache)
10. [Slow Query & General Logging](#9-slow-query--general-logging)
11. [Replication Tuning](#10-replication-tuning)
12. [Binary Log Settings](#11-binary-log-settings)
13. [Network & Timeout Settings](#12-network--timeout-settings)
14. [Character Set & Collation](#13-character-set--collation)
15. [Security Parameters](#14-security-parameters)
16. [Full Production Config Example](#full-production-config-example)
17. [Quick-Reference Cheat Sheet](#quick-reference-cheat-sheet)
18. [Tuning Workflow](#tuning-workflow)

---

## Why Parameter Tuning Matters

MySQL ships with conservative default values designed to run on minimal hardware. In production, these defaults are almost always wrong for your workload. A single well-tuned `innodb_buffer_pool_size` can make queries **10–100x faster** by eliminating disk I/O. Poor connection limits can cause cascading failures under load. Incorrect flush policies can destroy both durability and throughput simultaneously.

This guide covers every parameter that meaningfully impacts production performance, explains **what it controls**, **why it matters**, and gives you **concrete recommended values** with the reasoning behind them.

---

## 1. InnoDB Buffer Pool

### `innodb_buffer_pool_size`

**What it does:**
The single most important MySQL parameter. This is the in-memory cache where InnoDB stores table data (B-tree pages) and index pages. Every read that hits the buffer pool instead of disk is orders of magnitude faster.

**Why you need it:**
If this is too small, MySQL constantly evicts pages and re-reads them from disk — a pattern called "buffer pool thrashing." This causes high disk I/O, slow queries, and poor throughput.

**Recommended value:**

| Server Role | Value |
|---|---|
| Dedicated MySQL server | 70–80% of total RAM |
| Shared server (app + DB) | 40–50% of total RAM |
| Very small instance (≤4 GB RAM) | 50–60% of total RAM |

```ini
# Server with 32 GB RAM (dedicated MySQL)
innodb_buffer_pool_size = 24G

# Server with 8 GB RAM
innodb_buffer_pool_size = 6G
```

**Important note:** Don't set this so high that the OS starts swapping. Leave at least 1–2 GB for the OS, connections, and other MySQL structures.

---

### `innodb_buffer_pool_instances`

**What it does:**
Divides the buffer pool into multiple independent memory regions. Each instance has its own mutex (lock), which reduces contention on multi-core systems.

**Why you need it:**
A single large buffer pool becomes a bottleneck because all threads compete for the same lock when reading/writing pages. Multiple instances allow parallel access.

**Recommended value:**
- 1 instance per GB of buffer pool, up to a maximum of 64
- Minimum pool size per instance should be 1 GB

```ini
# For a 24 GB buffer pool
innodb_buffer_pool_instances = 16

# For an 8 GB buffer pool
innodb_buffer_pool_instances = 8
```

---

### `innodb_buffer_pool_chunk_size`

**What it does:**
The unit of memory allocation for the buffer pool. The total buffer pool size must be a multiple of `chunk_size × instances`.

**Why you need it:**
Controls memory allocation granularity. Incorrect values can cause MySQL to silently round down your configured pool size.

**Recommended value:**
```ini
innodb_buffer_pool_chunk_size = 128M  # Default; usually fine
```

Keep the default unless you have specific NUMA memory topology requirements.

---

## 2. InnoDB Log Settings

### `innodb_log_file_size`

**What it does:**
Controls the size of each InnoDB redo log file (`ib_logfile0`, `ib_logfile1`). The redo log records all changes to InnoDB data before they are written to the actual data files. This is what enables crash recovery.

**Why you need it:**
- **Too small:** MySQL flushes dirty pages from the buffer pool to disk too aggressively to avoid overwriting unprocessed log entries. This causes high write I/O and spikes in query latency.
- **Too large:** Crash recovery takes longer (must replay more log entries). Also, the first startup after a size change requires MySQL to recreate log files.

**Recommended value:**

```ini
# For most production servers (256 MB – 2 GB is typical)
innodb_log_file_size = 512M

# For write-heavy workloads
innodb_log_file_size = 1G

# For very high-write OLTP
innodb_log_file_size = 2G
```

**Rule of thumb:** Size it so the log can hold ~1 hour of writes without wrapping. Check `Innodb_os_log_written` status variable to estimate your write rate.

---

### `innodb_log_buffer_size`

**What it does:**
The in-memory buffer where InnoDB accumulates redo log data before writing it to the log files on disk.

**Why you need it:**
Without a buffer, every transaction commit would require a disk write. The buffer absorbs bursts of writes and flushes them together, reducing I/O operations.

**Recommended value:**
```ini
innodb_log_buffer_size = 64M   # For moderate workloads
innodb_log_buffer_size = 256M  # For large transactions or high-throughput writes
```

Increase this if you have large transactions (e.g., bulk inserts of millions of rows) or very high write concurrency.

---

### `innodb_log_files_in_group`

**What it does:**
The number of redo log files in the log group. InnoDB writes to them in a circular fashion.

**Recommended value:**
```ini
innodb_log_files_in_group = 2  # Default; almost always sufficient
```

The total redo log capacity is `innodb_log_file_size × innodb_log_files_in_group`.

---

## 3. InnoDB I/O & Flush Settings

### `innodb_flush_log_at_trx_commit`

**What it does:**
Controls when the redo log buffer is flushed to disk. This is the most critical **durability vs. performance** tradeoff in MySQL.

**Values:**

| Value | Behavior | Durability | Performance |
|---|---|---|---|
| `1` (default) | Flush to disk on every commit | **Full ACID** — no data loss | Slowest |
| `2` | Write to OS buffer on commit, flush to disk every second | Lose up to 1 second of data on OS crash | Fast |
| `0` | Flush every second (not per commit) | Lose up to 1 second on MySQL crash | Fastest |

**Recommended value:**

```ini
# Financial / e-commerce — data integrity is critical
innodb_flush_log_at_trx_commit = 1

# Analytics / reporting / can tolerate 1s data loss
innodb_flush_log_at_trx_commit = 2

# Staging / development / bulk load (NEVER in production where data matters)
innodb_flush_log_at_trx_commit = 0
```

---

### `innodb_flush_method`

**What it does:**
Controls how InnoDB flushes data to disk — whether it uses the OS page cache or bypasses it with Direct I/O.

**Values:**

| Value | Description |
|---|---|
| `fsync` | Uses OS cache; calls fsync() to flush |
| `O_DIRECT` | Bypasses OS page cache for data files |
| `O_DIRECT_NO_FSYNC` | Like O_DIRECT but skips fsync on log files (use only on ext4/XFS) |

**Recommended value:**

```ini
# Best for most Linux production systems with InnoDB buffer pool
innodb_flush_method = O_DIRECT
```

`O_DIRECT` prevents double-buffering (data stored in both the InnoDB buffer pool AND the OS page cache). This is almost always the right choice on Linux when the buffer pool is large.

---

### `innodb_io_capacity` and `innodb_io_capacity_max`

**What it does:**
Tells InnoDB how many I/O operations per second your storage can handle. InnoDB uses this to rate-limit background tasks like flushing dirty pages and merging change buffer entries.

**Why you need it:**
If set too low: dirty pages accumulate, eventually causing sudden large flush bursts that spike latency.
If set too high: background I/O starves foreground queries.

**Recommended value:**

| Storage Type | `innodb_io_capacity` | `innodb_io_capacity_max` |
|---|---|---|
| HDD (7200 RPM) | 200 | 400 |
| SSD (SATA) | 1000–2000 | 4000 |
| NVMe SSD | 5000–10000 | 20000 |
| Cloud SSD (gp3) | 3000 | 6000 |

```ini
# Example: NVMe SSD
innodb_io_capacity     = 8000
innodb_io_capacity_max = 16000
```

---

### `innodb_read_io_threads` and `innodb_write_io_threads`

**What it does:**
The number of background I/O threads dedicated to read and write operations respectively.

**Recommended value:**
```ini
innodb_read_io_threads  = 8   # Default 4; increase for read-heavy workloads
innodb_write_io_threads = 8   # Default 4; increase for write-heavy workloads
```

On SSD/NVMe storage, 8–16 threads per direction is a reasonable target. Beyond 16, returns diminish.

---

### `innodb_doublewrite`

**What it does:**
Before writing data pages to their final location, InnoDB first writes them to a special "doublewrite buffer" area. If a crash happens mid-write (partial page write), MySQL can recover the full page from the doublewrite buffer.

**Why you need it:**
Protects against torn pages on storage that does not guarantee atomic 16 KB page writes. Most HDDs and many SSDs fall into this category.

**Recommended value:**
```ini
innodb_doublewrite = ON   # Keep ON in production (default)
```

Only disable on storage that guarantees atomic writes (e.g., ZFS with `recordsize=16K`). The performance overhead is small (~5–10%) and the protection is valuable.

---

## 4. Query Cache (MySQL 5.x)

> **Note:** The query cache was **deprecated in MySQL 5.7.20** and **removed in MySQL 8.0**. Skip this section if you are on MySQL 8.0+.

### `query_cache_type` and `query_cache_size`

**What it does:**
Caches the results of SELECT queries. Subsequent identical queries are served from memory without touching tables.

**Why you often want it OFF:**
- Every write to a table invalidates ALL cached queries for that table — a global mutex.
- Under write-heavy or concurrent workloads, the cache becomes a serialization bottleneck that actually *slows* MySQL down.

**Recommended value:**
```ini
# MySQL 5.7 — disable completely
query_cache_type  = 0
query_cache_size  = 0
```

Use application-level caching (Redis, Memcached) instead. It scales far better.

---

## 5. Connection Management

### `max_connections`

**What it does:**
The maximum number of simultaneous client connections MySQL will accept. Beyond this limit, new connection attempts receive "Too many connections" error.

**Why you need it:**
- Too low: legitimate clients get rejected under peak load
- Too high: each idle connection consumes ~1 MB of memory for thread stack; a sudden spike of 10,000 connections can OOM-kill MySQL

**Recommended value:**
```ini
# Formula: (Available RAM - system overhead) / memory per connection
# Each connection: ~1 MB thread stack + per-session buffers

# Small server (8 GB RAM)
max_connections = 200

# Medium server (32 GB RAM)
max_connections = 500

# Large server (128 GB RAM) with connection pooler (ProxySQL/PgBouncer)
max_connections = 1000
```

**Important:** Always use a connection pooler (ProxySQL, MaxScale) in production. Let the pooler absorb thousands of app connections and maintain a smaller pool to MySQL.

---

### `max_connect_errors`

**What it does:**
If a host fails to connect this many times consecutively (due to network interruptions, not auth failures), MySQL blocks all future connections from that host.

**Recommended value:**
```ini
max_connect_errors = 1000000  # Effectively disable this feature; handle blocking at the firewall level
```

---

### `wait_timeout` and `interactive_timeout`

**What it does:**
Number of seconds MySQL waits on an idle non-interactive (`wait_timeout`) or interactive (`interactive_timeout`) connection before closing it.

**Why you need it:**
Leaked connections (app bugs, crash without cleanup) hold memory and file descriptors. Short timeouts reclaim them automatically.

**Recommended value:**
```ini
wait_timeout        = 28800   # 8 hours (default); reduce for high-connection apps
interactive_timeout = 28800   # 8 hours

# For apps with connection pools and many idle connections
wait_timeout        = 600     # 10 minutes
interactive_timeout = 600
```

---

## 6. Thread & Cache Settings

### `thread_cache_size`

**What it does:**
MySQL can cache thread objects after a connection closes, so new connections reuse cached threads instead of creating new OS threads (which is expensive).

**Why you need it:**
Thread creation involves OS-level system calls. Under high connection churn (connections opening and closing rapidly), caching threads dramatically reduces CPU overhead.

**Recommended value:**
```ini
# Formula: ~8 + (max_connections / 100)

thread_cache_size = 50   # For max_connections = 500
thread_cache_size = 100  # For max_connections = 1000
```

Monitor `Threads_created` status variable. If it grows rapidly, increase this value.

---

### `thread_stack`

**What it does:**
The stack size for each connection thread. Needed for stored procedures, functions, and deep recursive operations.

**Recommended value:**
```ini
thread_stack = 256K   # Default; increase to 512K if using complex stored procedures
```

---

## 7. Temporary Tables & Sort Buffers

### `tmp_table_size` and `max_heap_table_size`

**What it does:**
Controls the maximum size of in-memory temporary tables. When a query needs to create a temporary table (for GROUP BY, ORDER BY, DISTINCT, etc.), MySQL first tries to create it in memory. If the table exceeds this size, it spills to disk as a MyISAM/TempTable file.

**Why you need it:**
Disk-based temp tables are dramatically slower. Setting these values high keeps temp tables in RAM.

**Note:** The effective limit is `MIN(tmp_table_size, max_heap_table_size)`.

**Recommended value:**
```ini
tmp_table_size      = 256M
max_heap_table_size = 256M

# For analytics / reporting workloads with large GROUP BY operations
tmp_table_size      = 512M
max_heap_table_size = 512M
```

Monitor `Created_tmp_disk_tables` vs `Created_tmp_tables`. If disk ratio is high (>5%), increase these values.

---

### `sort_buffer_size`

**What it does:**
Per-connection buffer used for ORDER BY and GROUP BY operations. Each sort operation allocates this much memory **per connection that needs sorting**.

**Why you need it carefully:**
Unlike the buffer pool (one global allocation), sort_buffer_size is allocated per-connection, per-sort. Setting it to 256 MB with 500 connections could allocate 128 GB on a worst-case query.

**Recommended value:**
```ini
sort_buffer_size = 2M    # Good default for OLTP
sort_buffer_size = 4M    # For analytics with large result sets
```

Do **not** set this above 16M without careful memory planning.

---

### `join_buffer_size`

**What it does:**
Per-connection buffer used for full joins (joins that can't use an index). Each join that lacks an index key allocates this buffer.

**Why you need it:**
It's tempting to increase this to speed up bad queries, but the right fix is almost always to **add the missing index**. A large join buffer hides a performance problem instead of fixing it.

**Recommended value:**
```ini
join_buffer_size = 2M   # Keep this small; fix missing indexes instead
```

---

### `read_buffer_size` and `read_rnd_buffer_size`

**What they do:**
- `read_buffer_size`: Buffer for sequential table scans
- `read_rnd_buffer_size`: Buffer for reading rows in sorted order after a sort operation

**Recommended value:**
```ini
read_buffer_size     = 256K   # Default; increase slightly for bulk scans
read_rnd_buffer_size = 512K
```

---

## 8. Table & File Descriptor Cache

### `table_open_cache`

**What it does:**
MySQL caches open table file descriptors. When a query accesses a table, MySQL checks this cache first. If the table isn't cached, it opens the file (which costs an OS file open call).

**Why you need it:**
With hundreds of tables and concurrent connections, running out of open table cache causes frequent file open/close operations that tax the OS.

**Recommended value:**
```ini
# Formula: max_connections × tables_per_query (typically 2-3)
table_open_cache = 4000   # For max_connections = 500
table_open_cache = 8000   # For max_connections = 1000 or many tables
```

Monitor `Opened_tables` status variable. If it grows fast, increase this value.

---

### `table_definition_cache`

**What it does:**
Caches the parsed table definitions (.frm file contents in MySQL 5.x, or data dictionary in MySQL 8.0) in memory.

**Recommended value:**
```ini
table_definition_cache = 2000   # At least equal to the number of tables in your database
```

---

### `open_files_limit`

**What it does:**
The number of file descriptors MySQL requests from the OS. Each table, log file, and socket consumes file descriptors.

**Recommended value:**
```ini
open_files_limit = 65535   # Set high; also set ulimit in the OS
```

Also set in `/etc/security/limits.conf`:
```
mysql soft nofile 65535
mysql hard nofile 65535
```

---

## 9. Slow Query & General Logging

### `slow_query_log` and `long_query_time`

**What it does:**
Logs queries that take longer than `long_query_time` seconds to the slow query log. This is your primary tool for identifying performance problems.

**Why you need it:**
Without the slow query log, you're flying blind. Every production MySQL instance should have this enabled.

**Recommended value:**
```ini
slow_query_log       = ON
slow_query_log_file  = /var/log/mysql/slow.log
long_query_time      = 1     # Log queries taking > 1 second (start here)
# long_query_time    = 0.1   # Aggressive — log queries > 100ms
```

Analyze with `pt-query-digest` (Percona Toolkit) or `mysqldumpslow`.

---

### `log_queries_not_using_indexes`

**What it does:**
Logs queries to the slow query log that perform full table scans (no index used), regardless of their execution time.

**Recommended value:**
```ini
log_queries_not_using_indexes = ON   # Enable during optimization phases
```

Be careful — this can flood the slow log on large databases. Enable it during a controlled tuning window, not permanently on high-traffic servers.

---

### `min_examined_row_limit`

**What it does:**
Only log slow queries that examined at least this many rows. Prevents tiny fast-but-unindexed queries from filling the log.

**Recommended value:**
```ini
min_examined_row_limit = 1000   # Only flag queries that scanned at least 1000 rows
```

---

## 10. Replication Tuning

### `sync_binlog`

**What it does:**
Controls how often MySQL syncs the binary log to disk.

| Value | Behavior |
|---|---|
| `0` | OS decides when to flush (fastest, least safe) |
| `1` | Sync after every transaction commit (safest, slowest) |
| `N` | Sync every N commits |

**Recommended value:**
```ini
sync_binlog = 1   # Full durability — recommended for primary servers with replicas
sync_binlog = 0   # For replica servers or analytics instances
```

---

### `slave_parallel_workers` / `replica_parallel_workers` (MySQL 8.0+)

**What it does:**
Number of parallel SQL threads on a replica for applying relay log events. Single-threaded replication (default) often cannot keep up with a busy primary.

**Recommended value:**
```ini
# MySQL 5.7
slave_parallel_workers = 4
slave_parallel_type    = LOGICAL_CLOCK

# MySQL 8.0
replica_parallel_workers = 8
replica_parallel_type    = LOGICAL_CLOCK
```

---

### `slave_net_timeout` / `replica_net_timeout`

**What it does:**
Number of seconds the replica waits for data from the primary before reconnecting.

**Recommended value:**
```ini
replica_net_timeout = 30   # Reconnect if no data for 30 seconds
```

---

## 11. Binary Log Settings

### `binlog_format`

**What it does:**
Controls how changes are recorded in the binary log.

| Format | Description | Use Case |
|---|---|---|
| `STATEMENT` | Logs the SQL statement | Legacy; can cause replication inconsistencies |
| `ROW` | Logs the actual row data changed | Safest; recommended for most cases |
| `MIXED` | Uses statement when safe, row otherwise | Compromise option |

**Recommended value:**
```ini
binlog_format = ROW   # Default in MySQL 8.0; safest for replication
```

---

### `expire_logs_days` / `binlog_expire_logs_seconds`

**What it does:**
Automatic cleanup of old binary log files.

**Recommended value:**
```ini
# MySQL 5.7
expire_logs_days = 7   # Keep 7 days of binary logs

# MySQL 8.0 (in seconds)
binlog_expire_logs_seconds = 604800   # 7 days = 7 × 86400
```

Keep at least 3–7 days for point-in-time recovery (PITR). Don't set too high or binary logs will fill your disk.

---

### `max_binlog_size`

**What it does:**
Maximum size of a single binary log file before MySQL rotates to a new one.

**Recommended value:**
```ini
max_binlog_size = 512M   # Rotate every 512 MB (default is 1 GB)
```

---

## 12. Network & Timeout Settings

### `net_buffer_length` and `max_allowed_packet`

**What they do:**
- `net_buffer_length`: Initial size of the network communication buffer per connection
- `max_allowed_packet`: Maximum size of a single network packet (query, row, or BLOB)

**Why you need it:**
If `max_allowed_packet` is too small, large queries, INSERT with large BLOBs, or replication of large row events will fail with `Packet too large` errors.

**Recommended value:**
```ini
net_buffer_length   = 16K
max_allowed_packet  = 64M    # Increase if you store BLOBs or large TEXT fields
# max_allowed_packet = 256M  # For very large binary objects
```

---

### `connect_timeout`

**What it does:**
Seconds MySQL waits for the client to send the first packet during connection (authentication phase).

**Recommended value:**
```ini
connect_timeout = 10   # Reject slow-connecting clients quickly
```

---

## 13. Character Set & Collation

### `character_set_server` and `collation_server`

**What it does:**
Default character set and collation for new databases and tables.

**Why you need it:**
Using the wrong charset at server level leads to character corruption, failed imports, and inconsistent comparisons between columns.

**Recommended value:**
```ini
# Modern standard — supports full Unicode including emoji (4-byte characters)
character_set_server = utf8mb4
collation_server     = utf8mb4_unicode_ci    # Case-insensitive, linguistically correct sorting
# collation_server   = utf8mb4_0900_ai_ci   # MySQL 8.0 default — faster, better Unicode 9 support
```

**Warning:** Never use `utf8` in MySQL — it is actually a 3-byte subset of UTF-8 that cannot store emoji and many Unicode characters. Always use `utf8mb4`.

---

## 14. Security Parameters

### `local_infile`

**What it does:**
Allows LOAD DATA LOCAL INFILE to read files from the client machine.

**Recommended value:**
```ini
local_infile = OFF   # Disable unless explicitly needed — security risk
```

---

### `skip_name_resolve`

**What it does:**
When ON, MySQL skips reverse DNS lookup for incoming connections and uses IP addresses only in the grant tables.

**Why you need it:**
DNS lookups on every connection can add 10–100ms of latency, especially if the DNS server is slow or unreachable.

**Recommended value:**
```ini
skip_name_resolve = ON   # Always enable in production for performance
```

Note: If you have grants like `user@hostname` (using hostnames, not IPs), those grants will break. Change them to `user@'192.168.1.%'` format first.

---

### `sql_mode`

**What it does:**
Controls MySQL's strictness in data validation and SQL compliance.

**Recommended value:**
```ini
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

This prevents silent data truncation, zero-date storage, and division-by-zero silent errors — all common sources of subtle bugs in production.

---

## Full Production Config Example

Below is a complete `my.cnf` / `my.ini` for a **dedicated MySQL server with 32 GB RAM** running a mixed OLTP workload on NVMe SSD.

```ini
[mysqld]
# ─── Identity ─────────────────────────────────────────────────────────────────
server-id                        = 1
port                             = 3306
datadir                          = /var/lib/mysql
socket                           = /var/lib/mysql/mysql.sock
pid-file                         = /var/run/mysqld/mysqld.pid

# ─── Character Set ────────────────────────────────────────────────────────────
character_set_server             = utf8mb4
collation_server                 = utf8mb4_unicode_ci

# ─── InnoDB Buffer Pool ───────────────────────────────────────────────────────
innodb_buffer_pool_size          = 24G
innodb_buffer_pool_instances     = 16
innodb_buffer_pool_chunk_size    = 128M

# ─── InnoDB Redo Log ──────────────────────────────────────────────────────────
innodb_log_file_size             = 1G
innodb_log_files_in_group        = 2
innodb_log_buffer_size           = 128M

# ─── InnoDB Flush & I/O ───────────────────────────────────────────────────────
innodb_flush_log_at_trx_commit   = 1
innodb_flush_method              = O_DIRECT
innodb_io_capacity               = 8000
innodb_io_capacity_max           = 16000
innodb_read_io_threads           = 8
innodb_write_io_threads          = 8
innodb_doublewrite               = ON

# ─── Connections ──────────────────────────────────────────────────────────────
max_connections                  = 500
max_connect_errors               = 1000000
wait_timeout                     = 600
interactive_timeout              = 600
connect_timeout                  = 10
skip_name_resolve                = ON

# ─── Thread Cache ─────────────────────────────────────────────────────────────
thread_cache_size                = 50
thread_stack                     = 256K

# ─── Table Cache ──────────────────────────────────────────────────────────────
table_open_cache                 = 4000
table_definition_cache           = 2000
open_files_limit                 = 65535

# ─── Temp Tables & Buffers ────────────────────────────────────────────────────
tmp_table_size                   = 256M
max_heap_table_size              = 256M
sort_buffer_size                 = 2M
join_buffer_size                 = 2M
read_buffer_size                 = 256K
read_rnd_buffer_size             = 512K

# ─── Network ──────────────────────────────────────────────────────────────────
net_buffer_length                = 16K
max_allowed_packet               = 64M

# ─── Binary Logging ───────────────────────────────────────────────────────────
log_bin                          = /var/log/mysql/mysql-bin
binlog_format                    = ROW
sync_binlog                      = 1
max_binlog_size                  = 512M
binlog_expire_logs_seconds       = 604800   # 7 days (MySQL 8.0)
# expire_logs_days               = 7        # Use this for MySQL 5.7

# ─── Slow Query Log ───────────────────────────────────────────────────────────
slow_query_log                   = ON
slow_query_log_file              = /var/log/mysql/slow.log
long_query_time                  = 1
log_queries_not_using_indexes    = OFF   # Enable during tuning only

# ─── Replication (if replica) ─────────────────────────────────────────────────
# replica_parallel_workers       = 8
# replica_parallel_type          = LOGICAL_CLOCK
# replica_net_timeout            = 30

# ─── Security & SQL Mode ──────────────────────────────────────────────────────
local_infile                     = OFF
sql_mode                         = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

---

## Quick-Reference Cheat Sheet

| Parameter | What It Affects | Recommended Value |
|---|---|---|
| `innodb_buffer_pool_size` | Primary data/index cache | 70–80% of RAM |
| `innodb_buffer_pool_instances` | Buffer pool concurrency | 1 per GB of pool (max 64) |
| `innodb_log_file_size` | Redo log capacity | 512 MB – 2 GB |
| `innodb_log_buffer_size` | In-memory log buffer | 64–256 MB |
| `innodb_flush_log_at_trx_commit` | Durability vs. performance | `1` (production), `2` (analytics) |
| `innodb_flush_method` | OS cache bypass | `O_DIRECT` |
| `innodb_io_capacity` | Background I/O rate | 1000 (SATA SSD), 8000 (NVMe) |
| `innodb_doublewrite` | Torn page protection | `ON` |
| `max_connections` | Max simultaneous clients | 200–1000 (use a pooler) |
| `wait_timeout` | Idle connection cleanup | 600 (10 min) |
| `thread_cache_size` | Thread reuse | `8 + (max_connections / 100)` |
| `table_open_cache` | File descriptor cache | `max_connections × 4` |
| `tmp_table_size` | In-memory temp table limit | 256 MB |
| `sort_buffer_size` | Per-sort memory | 2–4 MB |
| `slow_query_log` | Query performance visibility | `ON` always |
| `long_query_time` | Slow query threshold | `1` second |
| `skip_name_resolve` | Connection DNS lookups | `ON` |
| `max_allowed_packet` | Max query/blob size | 64 MB |
| `character_set_server` | Default charset | `utf8mb4` |
| `binlog_format` | Replication safety | `ROW` |
| `sync_binlog` | Binlog durability | `1` (primary) |

---

## Tuning Workflow

Follow this systematic process when tuning a new production server:

**Step 1 — Establish a Baseline**
```sql
SHOW GLOBAL STATUS;
SHOW GLOBAL VARIABLES;
```
Record key metrics before making any changes.

**Step 2 — Size the Buffer Pool First**
This gives you the biggest win. Set `innodb_buffer_pool_size` to 70–80% of RAM and restart.

**Step 3 — Check Buffer Pool Hit Rate**
```sql
SELECT
  (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
    AS buffer_pool_hit_rate_pct
FROM (
  SELECT
    SUM(IF(variable_name = 'Innodb_buffer_pool_reads', variable_value, 0)) AS Innodb_buffer_pool_reads,
    SUM(IF(variable_name = 'Innodb_buffer_pool_read_requests', variable_value, 0)) AS Innodb_buffer_pool_read_requests
  FROM information_schema.global_status
) x;
```
Target: **>99%** hit rate. Below 95% — increase the buffer pool.

**Step 4 — Analyze the Slow Query Log**
```bash
pt-query-digest /var/log/mysql/slow.log | head -200
```
Fix the top 5 worst queries — add indexes, rewrite queries, or denormalize data. This often gives bigger wins than any configuration change.

**Step 5 — Check Temp Table Spills**
```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
-- Created_tmp_disk_tables / Created_tmp_tables > 5% → increase tmp_table_size
```

**Step 6 — Check Thread Creation Rate**
```sql
SHOW GLOBAL STATUS LIKE 'Threads_created';
-- Growing fast? Increase thread_cache_size
```

**Step 7 — Check Connection Utilization**
```sql
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
-- Compare to max_connections — if > 80%, either increase max_connections or add a pooler
```

**Step 8 — Monitor and Iterate**
Use tools like:
- **Percona Monitoring and Management (PMM)** — free, full observability stack
- **MySQLTuner** — quick automated recommendations
- **pt-query-digest** — slow query analysis
- **Grafana + mysqld_exporter** — dashboards

---

*Last updated: 2026 | Applicable to MySQL 5.7 and MySQL 8.0+*
