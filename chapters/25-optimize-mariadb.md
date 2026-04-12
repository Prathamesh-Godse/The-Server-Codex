[← Back to Index](../index.md)

---

# Chapter 25 — Optimise MariaDB

*Tuning MariaDB for production on a 1GB RAM server — swappiness confirmation, Performance Schema, query cache verification, DNS lookup disabling, binary log retention, and InnoDB buffer pool and log file sizing.*

---

## Contents

- [25.1 Swap & Swappiness](#251-swap--swappiness)
  - [Why This Matters for MariaDB](#why-this-matters-for-mariadb)
  - [Check the Current Swappiness Value](#check-the-current-swappiness-value)
- [25.2 Database Schemas](#252-database-schemas)
  - [What is a Schema?](#what-is-a-schema)
  - [Verify the Default Databases](#verify-the-default-databases)
- [25.3 Enabling the Performance Schema](#253-enabling-the-performance-schema)
  - [Backing Up and Editing 50-server.cnf](#backing-up-and-editing-50-servercnf)
  - [Add Performance Schema Directives](#add-performance-schema-directives)
  - [Restart MariaDB and Add the mariare Alias](#restart-mariadb-and-add-the-mariare-alias)
- [25.4 MySQL Query Cache](#254-mysql-query-cache)
  - [Why the Query Cache Should Be Disabled](#why-the-query-cache-should-be-disabled)
  - [Verify Query Cache Status](#verify-query-cache-status)
- [25.5 Disabling DNS Lookups — skip-name-resolve](#255-disabling-dns-lookups--skip-name-resolve)
- [25.6 MariaDB Log Files — Reducing Binary Log Retention](#256-mariadb-log-files--reducing-binary-log-retention)
  - [The Problem with Default Log Retention](#the-problem-with-default-log-retention)
  - [Check the Current Setting](#check-the-current-setting)
  - [Apply the Change at Runtime](#apply-the-change-at-runtime)
  - [Make the Change Permanent](#make-the-change-permanent)
- [25.7 InnoDB Storage Engine Tuning](#257-innodb-storage-engine-tuning)
  - [InnoDB Buffer Pool Size](#innodb-buffer-pool-size)
  - [InnoDB Log File Size](#innodb-log-file-size)
  - [Check Current Values Before Changing](#check-current-values-before-changing)
  - [Stop MariaDB, Edit the Config, Start MariaDB](#stop-mariadb-edit-the-config-start-mariadb)
  - [Verify the New InnoDB Settings](#verify-the-new-innodb-settings)
- [25.8 Summary](#258-summary)

---

## 25.1 Swap & Swappiness

### Why This Matters for MariaDB

When MariaDB starts, it maps the InnoDB buffer pool from disk into memory. If the OS aggressively swaps memory pages to disk during this process, MariaDB can crash. The kernel's `vm.swappiness` setting controls how eagerly the OS moves memory pages to swap — a high value means it swaps frequently, a low value means it prefers to keep things in RAM.

### Check the Current Swappiness Value

```bash
sudo sysctl -a | grep swappiness
```

The value should already be set to `1` via the persistent sysctl override file applied in Chapter 15:

```
vm.swappiness = 1
```

Confirm the override file is still in place:

```bash
sudo nano /etc/sysctl.d/custom_overrides.conf
```

The relevant entries at the top of the file:

```ini
# SWAPPINESS AND CACHE PRESSURE
vm.swappiness = 1
vm.vfs_cache_pressure = 50
```

`vm.swappiness = 1` tells the kernel to use swap only as a last resort. `vm.vfs_cache_pressure = 50` reduces how aggressively the kernel reclaims memory used for filesystem caching. Together they ensure the OS keeps as much data in RAM as possible before reaching for swap.

> For a standard rotating disk server use `vm.swappiness = 1`. For an ultra-fast NVMe SSD-backed server, `vm.swappiness = 5` is acceptable.

---

## 25.2 Database Schemas

### What is a Schema?

In MariaDB, "schema" is synonymous with "database" — they refer to the same thing. Two special built-in schemas are worth understanding before tuning:

**Performance Schema** monitors internal server events — query execution stages, thread activity, memory usage, and wait events. It is disabled by default and must be explicitly enabled.

**Sys Schema** is a set of views and stored procedures that interpret the raw data collected by the Performance Schema into human-readable reports. It is installed by default and becomes useful once the Performance Schema is enabled.

### Verify the Default Databases

Log into MariaDB and list the existing databases:

```bash
sudo mysql
```

```sql
show databases;
```

Output on a clean post-hardening install:

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.012 sec)
```

The `test` database is gone (removed during hardening in Chapter 24). The `performance_schema` database exists but is not yet collecting data — it needs to be enabled in `50-server.cnf`.

---

## 25.3 Enabling the Performance Schema

### Backing Up and Editing `50-server.cnf`

The main MariaDB server configuration file lives in `/etc/mysql/mariadb.conf.d/`. Navigate there and back up the server config before making changes:

```bash
cd /etc/mysql/mariadb.conf.d/
ls
sudo cp 50-server.cnf 50-server.cnf.bak
```

Open the config file:

```bash
sudo nano 50-server.cnf
```

### Add Performance Schema Directives

Add the following block inside the `[mysqld]` section, directly below the Basic Settings block:

```ini
# Performance Schema
performance_schema=ON
performance-schema-instrument='stage/%=ON'
performance-schema-consumer-events-stages-current=ON
performance-schema-consumer-events-stages-history=ON
performance-schema-consumer-events-stages-history-long=ON
```

`performance_schema=ON` enables the feature. The `instrument` and `consumer` lines activate stage-level event tracking — these capture the execution stages of queries, which is what tools like MySQLTuner use when profiling query performance.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Restart MariaDB and Add the `mariare` Alias

Apply the changes by restarting MariaDB:

```bash
sudo systemctl restart mariadb
```

Since restarting MariaDB is something that gets done repeatedly during tuning, add a bash alias for it:

```bash
nano ~/.bash_aliases
```

Add `mariare` alongside the existing Nginx and PHP-FPM aliases:

```bash
alias ngt='sudo nginx -t'
alias ngr='sudo systemctl reload nginx'
alias fpmr='sudo systemctl restart php8.3-fpm'
alias mariare='sudo systemctl restart mariadb'
```

Save and exit. Exit and reconnect via SSH to load the alias, then test it:

```bash
mariare
```

MariaDB restarts cleanly, confirming the alias works and the Performance Schema configuration is valid.

---

## 25.4 MySQL Query Cache

### Why the Query Cache Should Be Disabled

The MySQL query cache stores the full result set of a `SELECT` query so subsequent identical queries can be served from cache rather than re-executing. In practice, returning results from the query cache is actually *slower* than querying the dataset directly. Every write to a table invalidates all cached queries that touch that table, causing constant cache invalidation and a global lock. For a WordPress site with mixed reads and writes, it becomes a bottleneck.

### Verify Query Cache Status

```bash
sudo mysql
```

Check whether query cache support is compiled in:

```sql
show variables like 'have_query_cache';
```

```
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| have_query_cache | YES   |
+------------------+-------+
```

`YES` means the binary was compiled with query cache support. Now check whether it's actually enabled:

```sql
show variables like 'query_cache_%';
```

```
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 1048576 |
| query_cache_strip_comments   | OFF     |
| query_cache_type             | OFF     |
+------------------------------+---------+
```

`query_cache_type = OFF` confirms the query cache is not active — no further action is needed.

```sql
exit
```

---

## 25.5 Disabling DNS Lookups — `skip-name-resolve`

### Why DNS Lookups Hurt Performance

Every time a new client connects to MariaDB, a thread performs a reverse DNS lookup — it resolves the client's IP address to a hostname, then resolves that hostname back to an IP to verify it matches. This double-lookup happens on every new connection and adds measurable latency, especially when DNS is slow or unreliable.

Navigate to the config directory and open the file:

```bash
cd /etc/mysql/mariadb.conf.d/
sudo nano 50-server.cnf
```

The file already has `#skip-name-resolve` commented out in the Basic Settings section. Uncomment it by removing the `#`:

```ini
skip-name-resolve
```

The final `[mysqld]` Basic Settings block now looks like this:

```ini
[mysqld]

# * Basic Settings
pid-file                = /run/mysqld/mysqld.pid
basedir                 = /usr

# Performance Schema
performance_schema=ON
performance-schema-instrument='stage/%=ON'
performance-schema-consumer-events-stages-current=ON
performance-schema-consumer-events-stages-history=ON
performance-schema-consumer-events-stages-history-long=ON

# Broken reverse DNS slows down connections considerably and name resolve is
# safe to skip if there are no "host by domain name" access grants
skip-name-resolve

bind-address            = 127.0.0.1
```

Save and exit, then restart MariaDB:

```bash
mariare
```

---

## 25.6 MariaDB Log Files — Reducing Binary Log Retention

### The Problem with Default Log Retention

MariaDB's binary logs record every data-changing SQL statement for replication and point-in-time recovery. By default, these are kept for 10 days. On a small 1GB RAM VPS with limited disk space, 10 days of binary logs can consume gigabytes of disk — potentially causing MariaDB to become unresponsive when the disk fills up.

### Check the Current Setting

```bash
sudo mysql
```

```sql
show variables like 'expire_logs_days';
```

```
+------------------+-----------+
| Variable_name    | Value     |
+------------------+-----------+
| expire_logs_days | 10.000000 |
+------------------+-----------+
```

### Apply the Change at Runtime

Reduce the retention to 3 days immediately without restarting MariaDB:

```sql
set global expire_logs_days = 3;
flush binary logs;
exit
```

### Make the Change Permanent

The runtime change takes effect immediately but won't survive a restart. Open the config file and change `expire_logs_days` from `10` to `3` in the Logging and Replication section:

```bash
sudo nano 50-server.cnf
```

```ini
expire_logs_days        = 3
```

Save and exit, then restart MariaDB and verify the persistent value:

```bash
mariare
sudo mysql
```

```sql
show variables like 'expire_logs_days';
```

```
+------------------+----------+
| Variable_name    | Value    |
+------------------+----------+
| expire_logs_days | 3.000000 |
+------------------+----------+
```

```sql
exit
```

---

## 25.7 InnoDB Storage Engine Tuning

### InnoDB Buffer Pool Size

The `innodb_buffer_pool_size` is the amount of RAM InnoDB uses to cache table data and indexes. The more data that fits in the buffer pool, the fewer disk reads MariaDB needs to perform. The recommended value is **80% of total server RAM**.

On a 1GB server: 80% of 1024MB = approximately **800M**.

### InnoDB Log File Size

The `innodb_log_file_size` is the size of InnoDB's transaction log (also called the redo log). It records changes before they're written to the actual data files, which is what makes crash recovery possible. The recommended value is **25% of `innodb_buffer_pool_size`**.

On this server: 25% of 800M = **200M**.

### Check Current Values Before Changing

```bash
sudo mysql
```

```sql
show variables like '%innodb_buffer%';
```

Default output (`innodb_buffer_pool_size` = 134217728 bytes = 128MB):

```
+-------------------------------------+-----------+
| Variable_name                       | Value     |
+-------------------------------------+-----------+
| innodb_buffer_pool_size             | 134217728 |
+-------------------------------------+-----------+
```

```sql
show variables like '%innodb_log%';
```

Default output (`innodb_log_file_size` = 100663296 bytes ≈ 96MB):

```
+----------------------------+-----------+
| Variable_name              | Value     |
+----------------------------+-----------+
| innodb_log_file_size       | 100663296 |
+----------------------------+-----------+
```

Also check memory usage in `htop` before applying changes to confirm there's headroom:

```bash
htop
```

Exit `htop` with `q`.

### ⚠️ Stop MariaDB Before Changing the Log File Size

Changing `innodb_log_file_size` while MariaDB is running and then restarting it can corrupt InnoDB tables. The safe procedure is:

1. Check the current log file size (done above)
2. **Stop MariaDB** first
3. Edit `50-server.cnf`
4. Start MariaDB
5. Verify the new values

### Stop MariaDB, Edit the Config, Start MariaDB

```bash
sudo systemctl stop mariadb
sudo nano 50-server.cnf
```

Navigate to the InnoDB section of the file (use `Ctrl+W` to search for `innodb`). Below the commented-out example, add the production values:

```ini
innodb_buffer_pool_size = 800M
innodb_log_file_size = 200M
```

Save and exit, then start MariaDB:

```bash
sudo systemctl start mariadb
```

### Verify the New InnoDB Settings

```bash
sudo mysql
```

```sql
show variables like '%innodb_buffer%';
```

Confirming the new buffer pool size (838860800 bytes = 800MB):

```
+-------------------------------------+-----------+
| Variable_name                       | Value     |
+-------------------------------------+-----------+
| innodb_buffer_pool_size             | 838860800 |
+-------------------------------------+-----------+
```

```sql
show variables like '%innodb_log%';
```

Confirming the new log file size (209715200 bytes = 200MB):

```
+----------------------------+-----------+
| Variable_name              | Value     |
+----------------------------+-----------+
| innodb_log_file_size       | 209715200 |
+----------------------------+-----------+
```

```sql
exit
```

---

## 25.8 Summary

| Setting | Before | After | Location |
|---------|--------|-------|----------|
| `vm.swappiness` | 60 (default) | 1 | `/etc/sysctl.d/custom_overrides.conf` |
| Performance Schema | OFF | ON | `50-server.cnf` |
| Query cache | Compiled in, type=OFF | Verified OFF | No change needed |
| `skip-name-resolve` | Commented out | Active | `50-server.cnf` |
| `expire_logs_days` | 10 | 3 | `50-server.cnf` + runtime |
| `innodb_buffer_pool_size` | 128M (default) | 800M | `50-server.cnf` |
| `innodb_log_file_size` | ~96M (default) | 200M | `50-server.cnf` |

| Task | Command |
|------|---------|
| Check swappiness | `sudo sysctl -a \| grep swappiness` |
| Log into MariaDB | `sudo mysql` |
| Restart MariaDB | `mariare` |
| Stop MariaDB | `sudo systemctl stop mariadb` |
| Start MariaDB | `sudo systemctl start mariadb` |
| Edit server config | `sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf` |
| Check InnoDB buffer pool | `show variables like '%innodb_buffer%';` |
| Check InnoDB log size | `show variables like '%innodb_log%';` |
| Check binary log retention | `show variables like 'expire_logs_days';` |
| Monitor memory usage | `htop` |

---

| | | |
|:---|:---:|---:|
| [← Chapter 24 — Harden MariaDB](./24-harden-mariadb.md) | [↑ Top](#chapter-25--optimise-mariadb) | [↑ Back to Index](../index.md) |
