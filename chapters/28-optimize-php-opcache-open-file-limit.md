[← Back to Index](../index.md)

---

# Chapter 28 — Optimise PHP: OPcache & PHP-FPM Open File Limit

*Completing the `server_override.ini` optimisation block, understanding OPcache and why it is deferred to per-site pool configuration, and raising the PHP-FPM open file limit from 1024 to 32768.*

---

## Contents

- [28.1 Part 1 — Completing server_override.ini](#281-part-1--completing-server_overrideini)
- [28.2 Part 2 — OPcache](#282-part-2--opcache)
  - [What OPcache Is](#what-opcache-is)
  - [Key OPcache Directives](#key-opcache-directives)
  - [Development vs Production Timestamp Behaviour](#development-vs-production-timestamp-behaviour)
  - [Clearing OPcache](#clearing-opcache)
  - [Why OPcache is Not Fully Configured at This Stage](#why-opcache-is-not-fully-configured-at-this-stage)
- [28.3 Part 3 — PHP Error Log File](#283-part-3--php-error-log-file)
- [28.4 Part 4 — Raising the PHP-FPM Open File Limit](#284-part-4--raising-the-php-fpm-open-file-limit)
  - [Step 1 — Identify the Current Limit](#step-1--identify-the-current-limit)
  - [Step 2 — Back Up and Edit php-fpm.conf](#step-2--back-up-and-edit-php-fpmconf)
  - [Step 3 — Reload Nginx and Restart PHP-FPM](#step-3--reload-nginx-and-restart-php-fpm)
  - [Step 4 — Verify the New Limits](#step-4--verify-the-new-limits)
- [28.5 Summary](#285-summary)

---

## 28.1 Part 1 — Completing `server_override.ini`

Navigate back to the `conf.d` directory and reopen the override file created in Chapter 27 to verify or add the optimisation block:

```bash
cd /etc/php/8.3/fpm/conf.d/
ls
sudo nano server_override.ini
```

`ls` confirms `server_override.ini` is already present alongside the extension `.ini` files. The complete file with both the hardening and optimisation blocks:

```ini
# HARDEN PHP
allow_url_fopen = Off
cgi.fix_pathinfo = 0
expose_php = Off

# OPTIMIZE PHP
upload_max_filesize = 100M
post_max_size = 125M
max_input_vars = 3000
memory_limit = 256M
#max_execution_time = 90
#max_input_time = 60
```

> **Note:** The commented-out value for `max_execution_time` is `90` rather than `30`. `90` seconds is a more practical ceiling for WordPress, which can have long-running admin operations such as plugin bulk updates or theme switching.

Save and exit (`Ctrl+X`, `Y`, `Enter`), then reload PHP-FPM:

```bash
fpmr
```

---

## 28.2 Part 2 — OPcache

### What OPcache Is

OPcache is a bytecode caching engine built directly into PHP since version 5.5. Every time PHP processes a `.php` file, it compiles the source code into bytecode before executing it. Without caching, this compilation happens on every single request, even if the file has not changed. OPcache stores the compiled bytecode in shared memory so that subsequent requests can skip the compile step entirely and execute the cached bytecode directly.

For a WordPress server, OPcache is one of the most impactful performance improvements available. WordPress core, theme files, and plugin code are all static between deployments, making them ideal candidates for aggressive bytecode caching.

### Key OPcache Directives

| Directive | Purpose |
|-----------|---------|
| `opcache.memory_consumption` | Total shared memory (in MB) allocated for storing cached bytecode. `128M` is a good starting value for a typical WordPress site. |
| `opcache.interned_strings_buffer` | Memory (in MB) for caching interned strings — frequently repeated string literals in PHP code. `16M` is a sensible default. |
| `opcache.max_accelerated_files` | Maximum number of PHP files OPcache will track. WordPress with a set of plugins can easily have thousands of files — set this to at least `10000`. |
| `opcache.validate_timestamps` | Whether OPcache checks if a file has changed on disk before serving cached bytecode. |
| `opcache.validate_permission` | Whether OPcache checks file read permissions before serving cached bytecode. |

### Development vs Production Timestamp Behaviour

`opcache.validate_timestamps` is the most important directive to understand correctly, because its correct value differs between development and production environments.

**Development server:**

```ini
opcache.validate_timestamps = 1
opcache.revalidate_freq = 2
```

With `validate_timestamps = 1`, PHP checks whether each cached file has been modified every `revalidate_freq` seconds (2 seconds in this case). File changes are picked up quickly — essential when actively editing PHP code on the server.

**Production server:**

```ini
opcache.validate_timestamps = 0
opcache.revalidate_freq = 2
```

With `validate_timestamps = 0`, OPcache **never** checks for file changes — the cache is only cleared when PHP-FPM is restarted or reloaded, or when a plugin explicitly clears it. This gives maximum performance on a server where files only change during controlled deployments.

> **This project targets a production server**, so `validate_timestamps = 0` will be used when OPcache is enabled per-site.

### Clearing OPcache

**From the command line:** Restarting or reloading the `php8.3-fpm` service clears the entire OPcache for all sites sharing that FPM instance.

```bash
fpmr
# or
sudo systemctl restart php8.3-fpm
```

**Per-site (without restarting FPM):** Use a WordPress plugin to clear OPcache for a specific site only. Suitable options include WP OPcache, OPcache Reset, W3 Total Cache, and OPcache Manager. This is the recommended approach on a multi-site server where a full FPM restart would clear every site's cache simultaneously.

### Why OPcache is Not Fully Configured at This Stage

OPcache configuration is deferred intentionally. In this project, each WordPress site will be isolated using its own **PHP-FPM pool**, and each pool will have its own dedicated OPcache instance. Configuring OPcache inside a shared `conf.d` override file here would apply a single global cache to all pools — which removes the per-site isolation benefit. The full OPcache configuration is done inside each site's individual pool configuration file, covered in a later chapter.

---

## 28.3 Part 3 — PHP Error Log File

On a production server, PHP errors must never be displayed to the browser — both for security reasons (error messages can expose file paths, code logic, and database structure) and for user experience. Instead, errors are written to a dedicated log file.

The approach in this project is to configure a per-site PHP error log path inside each site's PHP-FPM pool configuration, since each site is isolated in its own pool. The `display_errors = Off` directive ensures errors are suppressed from browser output even if logging is not yet configured. The full per-site error log setup is covered during the site configuration stage.

---

## 28.4 Part 4 — Raising the PHP-FPM Open File Limit

### The Problem

By default, PHP-FPM's open file limit is inherited from the system and is often just `1024` — significantly lower than the limits set for Nginx (`45000`) and MariaDB (`40000`). On a busy server this causes the same class of `"too many open files"` errors seen with the other services.

The fix for PHP-FPM is handled differently from Nginx and MariaDB. Rather than using a systemd drop-in, PHP-FPM exposes `rlimit_files` and `rlimit_core` as native directives directly inside `php-fpm.conf`.

### Step 1 — Identify the Current Limit

Navigate to the PHP-FPM directory and list its contents:

```bash
cd /etc/php/8.3/fpm/
ls
```

```
conf.d  php-fpm.conf  php.ini  pool.d
```

Get the PHP-FPM process IDs. Note that PHP-FPM has multiple processes — a **master** process running as `root`, and **worker** processes running as `www-data`:

```bash
ps aux | grep php-fpm
```

Example output:

```
root     24832  0.0  4.1  266596  40408 ?   Ss  Jul11  0:30 php-fpm: master process (/etc/php/8.3/fpm/php-fpm.conf)
www-data 54734  0.0  1.6  267156  15992 ?   S   21:00  0:00 php-fpm: pool www
www-data 54735  0.0  1.6  267156  15992 ?   S   21:00  0:00 php-fpm: pool www
```

Read the limits for the **master process** PID:

```bash
cat /proc/24832/limits
```

The **Max open files** row shows the current state:

```
Limit                  Soft Limit   Hard Limit   Units
...
Max open files         1024         524288       files
...
```

The soft limit of `1024` is what matters at runtime — it is far too low for a production server. Check a worker process as well to confirm it inherits the same low soft limit:

```bash
cat /proc/54734/limits
```

The worker shows `Max open files = 1024` on both soft and hard limits.

### Step 2 — Back Up and Edit `php-fpm.conf`

Always back up the original file before making changes:

```bash
sudo cp php-fpm.conf php-fpm.conf.bak
sudo nano php-fpm.conf
```

Use `Ctrl+W` to search for `rlimit_files`. The directives are present but commented out:

**Before (default, commented out):**

```ini
; Set open file descriptor rlimit for the master process.
; Default Value: system defined value
;rlimit_files = 1024

; Set max core size rlimit for the master process.
; Possible Values: 'unlimited' or an integer greater or equal to 0
; Default Value: system defined value
;rlimit_core = 0
```

Uncomment both lines and set the new values:

**After:**

```ini
; Set open file descriptor rlimit for the master process.
; Default Value: system defined value
rlimit_files = 32768

; Set max core size rlimit for the master process.
; Possible Values: 'unlimited' or an integer greater or equal to 0
; Default Value: system defined value
rlimit_core = unlimited
```

Save and exit (`Ctrl+X`, `Y`, `Enter`).

**`rlimit_files = 32768`** sets the maximum number of open file descriptors for the PHP-FPM master process. Worker processes inherit this value.

**`rlimit_core = unlimited`** removes the cap on core dump file size. If a PHP-FPM worker crashes unexpectedly, an unlimited core dump allows the crash to be fully diagnosed.

### Step 3 — Reload Nginx and Restart PHP-FPM

```bash
ngr
fpmr
```

Which expands to `sudo systemctl reload nginx` and `sudo systemctl restart php8.3-fpm`.

> **Important:** PHP-FPM must be **restarted** (not just reloaded) for `rlimit` changes to take effect. A reload applies configuration changes but does not re-launch the master process, so the process-level limits do not change until a full restart.

### Step 4 — Verify the New Limits

Because PHP-FPM was restarted, all PIDs have changed. Get the new process list:

```bash
ps aux | grep php-fpm
```

Check the master process limits:

```bash
cat /proc/<MASTER_PID>/limits
```

The **Max open files** row now shows:

```
Max open files         32768        32768        files
```

Then verify both worker processes inherit the same limit:

```bash
cat /proc/<WORKER_1_PID>/limits
cat /proc/<WORKER_2_PID>/limits
```

Also verify the `Max core file size` row has changed from `0` to `unlimited` on the master process, confirming `rlimit_core = unlimited` was applied correctly.

Return to the home directory:

```bash
cd
```

---

## 28.5 Summary

| Task | File / Command | Value |
|------|---------------|-------|
| Complete optimisation block | `/etc/php/8.3/fpm/conf.d/server_override.ini` | See file contents above |
| OPcache production setting | Per-site pool config (deferred) | `opcache.validate_timestamps = 0` |
| OPcache clear (all sites) | `fpmr` | Restarts FPM, flushes entire cache |
| PHP-FPM open file limit | `/etc/php/8.3/fpm/php-fpm.conf` | `rlimit_files = 32768` |
| Core dump limit | `/etc/php/8.3/fpm/php-fpm.conf` | `rlimit_core = unlimited` |
| Apply limit changes | `ngr && fpmr` | Reload Nginx, restart PHP-FPM |
| Verify master process | `ps aux \| grep php-fpm` → `cat /proc/<PID>/limits` | Max open files: 32768 |
| Verify worker processes | `cat /proc/<worker_PID>/limits` | Max open files: 32768 |

> **Automation note:** The `rlimit_files` and `rlimit_core` changes can be scripted with `sed`:
> ```bash
> sudo sed -i 's/^;rlimit_files = 1024/rlimit_files = 32768/' /etc/php/8.3/fpm/php-fpm.conf
> sudo sed -i 's/^;rlimit_core = 0/rlimit_core = unlimited/' /etc/php/8.3/fpm/php-fpm.conf
> ```
> Always back up the file with `cp php-fpm.conf php-fpm.conf.bak` before running `sed`. After the edit, run `sudo systemctl restart php8.3-fpm` — a reload alone is not sufficient for rlimit changes.

---

| | | |
|:---|:---:|---:|
| [← Chapter 27 — Harden & Optimise PHP](./27-harden-and-optimize-php.md) | [↑ Top](#chapter-28--optimise-php-opcache--php-fpm-open-file-limit) | [Chapter 29 — Web Root Directory Structure →](./29-web-root-directory-structure.md) |
