[← Back to Index](../index.md)

---

# Chapter 27 — Harden & Optimise PHP

*Hardening PHP 8.3 FPM by disabling insecure defaults, optimising upload limits and memory settings for WordPress, and raising the PHP-FPM open file limit.*

---

## Contents

- [27.1 PHP 8.3 on Ubuntu 24.04 — Version Context](#271-php-83-on-ubuntu-2404--version-context)
- [27.2 PHP Configuration File Hierarchy](#272-php-configuration-file-hierarchy)
- [27.3 Dangerous PHP Functions](#273-dangerous-php-functions)
- [27.4 Part 1 — Exploring the PHP Directory Structure](#274-part-1--exploring-the-php-directory-structure)
- [27.5 Part 2 — Creating the Server Override File](#275-part-2--creating-the-server-override-file)
- [27.6 Directive Reference](#276-directive-reference)
  - [Hardening Directives](#hardening-directives)
  - [Optimisation Directives](#optimisation-directives)
- [27.7 Part 3 — Reloading PHP-FPM](#277-part-3--reloading-php-fpm)
- [27.8 Part 4 — Raising the PHP-FPM Open File Limit](#278-part-4--raising-the-php-fpm-open-file-limit)
- [27.9 Summary](#279-summary)

---

## 27.1 PHP 8.3 on Ubuntu 24.04 — Version Context

Ubuntu 24.04 LTS ships PHP 8.3 as its default PHP version. This is a good choice for a production WordPress server for several reasons:

- PHP 8.3 delivers up to **50% performance improvement** over PHP 7.4 and 8.0, and around **15% over 8.1 and 8.2**, depending on site workload.
- The official PHP 8.3 end-of-life date is **23 November 2026**.
- Ubuntu's own packaging of PHP 8.3 for 24.04 LTS extends support to **April 2029**, with Ubuntu Pro further extending that to **2034**.

This means the stack built in this project has a long supported lifespan without requiring any out-of-band PHP upgrades.

> **Plugin and theme compatibility note:** Before deploying sites on PHP 8.3, verify that all themes and plugins you plan to use are compatible. Test in a staging environment first — some older plugins may still rely on deprecated functions.

---

## 27.2 PHP Configuration File Hierarchy

PHP reads its configuration from multiple files in a defined order. Understanding this hierarchy is important before making any changes, because editing the wrong file can either have no effect or get overwritten on the next package update.

| File | Purpose |
|------|---------|
| `/etc/php/8.3/fpm/php.ini` | Main PHP configuration file. Package-managed — avoid direct edits where possible. |
| `/etc/php/8.3/fpm/conf.d/server_override.ini` | Server-wide override file. Any valid `.ini` file dropped in `conf.d/` is loaded automatically and its values override `php.ini`. This is the correct place for server-wide tuning. |
| `.user.ini` (in site root) | Per-site overrides. Allows site-specific values without affecting other sites on the server. |
| `php_admin_value` (in Nginx pool config) | The method used in this project for per-site PHP settings, since each WordPress site gets its own PHP-FPM pool. `php_admin_value` directives set in the pool config cannot be overridden by `.user.ini`. |

The approach used throughout this project is to leave `php.ini` untouched, write server-wide settings into `server_override.ini` inside `conf.d/`, and set site-specific overrides using `php_admin_value` in each Nginx site's PHP-FPM pool configuration.

---

## 27.3 Dangerous PHP Functions

Some built-in PHP functions — such as `exec()`, `passthru()`, `shell_exec()`, and `system()` — can be used to run arbitrary OS commands from within PHP code. If a site is compromised, an attacker can leverage these functions to take control of the server.

These functions are disabled on a **per-site basis** using `php_admin_value disable_functions` in each site's PHP-FPM pool configuration, which is covered in the WordPress hardening section later in the project. Disabling them globally here would be too aggressive and could break legitimate plugin functionality — the error log is the right tool to diagnose any breakage after per-site disabling is applied.

---

## 27.4 Part 1 — Exploring the PHP Directory Structure

Before creating any configuration files, navigate through the PHP directory tree to understand what is already in place.

```bash
cd /etc/php/
ls
cd 8.3/
ls
```

Output:

```
cli  fpm  mods-available
```

The `fpm/` directory contains everything relevant to PHP-FPM. Navigate into it:

```bash
cd fpm/
ls
```

```
conf.d  php-fpm.conf  php.ini  pool.d
```

The `conf.d/` directory is where the server override file will live. Navigate into it to see what extension `.ini` files are already loaded:

```bash
cd conf.d/
ls
```

The output shows a large number of extension-specific `.ini` files (e.g. `20-curl.ini`, `20-gd.ini`, `20-imagick.ini`, `20-mysqli.ini`, etc.) that were automatically installed alongside the PHP extension packages. Any new `.ini` file created in this directory with a valid filename will be picked up automatically by PHP-FPM on restart.

---

## 27.5 Part 2 — Creating the Server Override File

Create a new file called `server_override.ini` directly inside `conf.d/`. The name is arbitrary — any `.ini` extension will work — but a descriptive name makes its purpose clear.

```bash
sudo nano server_override.ini
```

Add the following content in two clearly labelled sections:

```ini
# HARDEN PHP
allow_url_fopen = Off
cgi.fix_pathinfo = 0
expose_php = Off

# OPTIMIZE PHP
upload_max_filesize = 100M
post_max_size = 125M
max_input_vars = 3000
# Memory limit also needs to be set in wp-config.php
memory_limit = 256M
#max_execution_time = 30
#max_input_time = 60
```

Save and exit nano (`Ctrl+X`, `Y`, `Enter`).

---

## 27.6 Directive Reference

### Hardening Directives

**`allow_url_fopen = Off`**
Disables the ability for PHP's file functions (`fopen`, `file_get_contents`, etc.) to open remote URLs over HTTP or FTP. When `On`, a vulnerability in any PHP code on the server could be used to fetch and execute remote malicious files. WordPress does not require this to be `On` — it uses the `WP_HTTP` API for remote requests rather than `fopen` wrappers.

**`cgi.fix_pathinfo = 0`**
Prevents a well-known Nginx + PHP-FPM path traversal vulnerability. When set to `1` (the default), PHP will attempt to find the nearest valid PHP file when given a path like `/uploads/malicious.jpg/index.php` — which can allow PHP code embedded in uploaded image files to execute. Setting it to `0` disables this entirely.

**`expose_php = Off`**
Stops PHP from adding an `X-Powered-By: PHP/8.3.x` header to every HTTP response. Advertising the exact PHP version publicly is unnecessary information for an attacker performing reconnaissance. This pairs with the `more_clear_headers 'X-Powered'` directive set in the Nginx `basic_settings.conf` from Chapter 21.

### Optimisation Directives

**`upload_max_filesize = 100M`**
The maximum size of a single file that can be uploaded through PHP. The default is `2M`, which is too small for WordPress theme and plugin uploads.

**`post_max_size = 125M`**
The maximum size of the entire POST body. This must always be **larger** than `upload_max_filesize` to account for the overhead of the multipart form data. Setting it to `125M` with a `100M` file limit provides adequate headroom.

**`max_input_vars = 3000`**
The maximum number of input variables PHP will process per request. The default of `1000` can cause problems with WordPress sites that have many menu items, complex ACF field groups, or large option pages — data silently gets truncated at the limit.

**`memory_limit = 256M`**
The maximum amount of memory a single PHP process is allowed to consume. `256M` is appropriate for a WordPress server handling typical plugin workloads (page builders, WooCommerce, etc.).

> **Important:** `memory_limit` must also be set in `wp-config.php` for each WordPress site using `define('WP_MEMORY_LIMIT', '256M');`. The PHP-level limit is a hard ceiling, but WordPress applies its own internal limit that defaults lower.

**`max_execution_time` and `max_input_time`** (left commented out)
These are left as comments because the defaults (`30s` and `60s` respectively) are suitable for most sites. They can be uncommented and adjusted if specific plugins require longer execution windows.

---

## 27.7 Part 3 — Reloading PHP-FPM

After saving the override file, reload PHP-FPM to apply the new configuration. The `fpmr` alias set up in Chapter 23 makes this a single short command:

```bash
fpmr
```

Which expands to `sudo systemctl reload php8.3-fpm`.

Verify PHP-FPM is running cleanly after the reload:

```bash
sudo systemctl status php8.3-fpm
```

---

## 27.8 Part 4 — Raising the PHP-FPM Open File Limit

PHP-FPM has its own open file limit, separate from the Nginx and MariaDB limits raised in earlier chapters. The limit is controlled directly inside `php-fpm.conf` rather than through a systemd drop-in, because PHP-FPM has native `rlimit` directives in its own configuration.

### Step 1 — Find the PHP-FPM Process ID and Check the Current Limit

```bash
ps aux | grep php-fpm
```

Note the PID from the master process row, then read its current limits:

```bash
cat /proc/<PID>/limits
```

Look at the **Max open files** row. The default value is typically `1024` or `32768`.

### Step 2 — Back Up and Edit `php-fpm.conf`

```bash
cd /etc/php/8.3/fpm/
sudo cp php-fpm.conf php-fpm.conf.bak
sudo nano /etc/php/8.3/fpm/php-fpm.conf
```

Search for `rlimit_files` using `Ctrl+W` in nano. The directive is present but commented out. Uncomment it and set the value:

```ini
rlimit_files = 32768
rlimit_core = unlimited
```

`rlimit_files` sets the maximum number of open file descriptors for PHP-FPM worker processes. `rlimit_core` controls core dump size — setting it to `unlimited` is useful for debugging if a worker process crashes.

### Step 3 — Reload Nginx and Restart PHP-FPM

```bash
ngr
fpmr
```

Which expands to `sudo systemctl reload nginx` and `sudo systemctl restart php8.3-fpm`.

### Step 4 — Verify the New Limit is Active

Find the new PHP-FPM master process PID after the restart:

```bash
ps aux | grep php-fpm
cat /proc/<NEW_PID>/limits
```

Confirm **Max open files** now shows `32768` for both soft and hard limits.

---

## 27.9 Summary

| Setting | File | Value |
|---------|------|-------|
| `allow_url_fopen` | `server_override.ini` | `Off` |
| `cgi.fix_pathinfo` | `server_override.ini` | `0` |
| `expose_php` | `server_override.ini` | `Off` |
| `upload_max_filesize` | `server_override.ini` | `100M` |
| `post_max_size` | `server_override.ini` | `125M` |
| `max_input_vars` | `server_override.ini` | `3000` |
| `memory_limit` | `server_override.ini` | `256M` |
| `rlimit_files` | `php-fpm.conf` | `32768` |
| `rlimit_core` | `php-fpm.conf` | `unlimited` |

---

| | | |
|:---|:---:|---:|
| [← Chapter 26 — MySQLTuner & MariaDB Open File Limits](./26-mysqltuner-mariadb-open-file-limits.md) | [↑ Top](#chapter-27--harden--optimise-php) | [Chapter 28 — Optimise PHP: OPcache & Open File Limit →](./28-optimize-php-opcache-open-file-limit.md) |
