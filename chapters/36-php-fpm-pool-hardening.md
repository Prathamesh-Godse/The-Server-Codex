[← Back to Index](../index.md)

---

# Chapter 36 — PHP-FPM Pool Hardening: Ownership, Permissions & Per-Pool PHP Configuration

*Transferring WordPress file ownership to the pool user, applying hardened filesystem permissions, verifying via a temporary `phpinfo()` page, and configuring per-pool PHP directives including `allow_url_fopen` and a strict `disable_functions` list.*

> **Scope:** Ubuntu 24.04 · Nginx · PHP 8.3 FPM · WordPress

---

## Contents

- [36.1 Overview](#361-overview)
- [36.2 Step 1 — Verify Current State Before Changes](#362-step-1--verify-current-state-before-changes)
- [36.3 Step 2 — Transfer Ownership to the Pool User](#363-step-2--transfer-ownership-to-the-pool-user)
- [36.4 Step 3 — Set Filesystem Permissions](#364-step-3--set-filesystem-permissions)
  - [3.1 — Directory Permissions](#31--directory-permissions)
  - [3.2 — File Permissions](#32--file-permissions)
- [36.5 Step 4 — Create a Temporary phpinfo() Test File](#365-step-4--create-a-temporary-phpinfo-test-file)
- [36.6 Step 5 — Understanding the PHP Configuration Hierarchy](#366-step-5--understanding-the-php-configuration-hierarchy)
- [36.7 Step 6 — Pool-Level PHP Configuration](#367-step-6--pool-level-php-configuration)
  - [6.1 — Current Baseline Directives](#61--current-baseline-directives)
  - [6.2 — Enable allow_url_fopen for WordPress](#62--enable-allow_url_fopen-for-wordpress)
  - [6.3 — Disable Dangerous PHP Functions](#63--disable-dangerous-php-functions)
- [36.8 Step 7 — Full Pool Config PHP Directives (Final State)](#368-step-7--full-pool-config-php-directives-final-state)
- [36.9 Step 8 — Delete the phpinfo Test File](#369-step-8--delete-the-phpinfo-test-file)
- [36.10 Final State Verification](#3610-final-state-verification)
- [36.11 Summary of Changes](#3611-summary-of-changes)

---

## 36.1 Overview

At this stage, WordPress is installed and the PHP-FPM pool (`expertwp.help.conf`) has already been created and is running as its own dedicated system user (`expertwp`). The site was initially installed with files owned by `www-data`. The goals here are:

1. Transfer file ownership from `www-data` → the pool user (`expertwp`)
2. Apply correct directory and file permissions
3. Verify the setup via a temporary `phpinfo()` page
4. Tune PHP settings at the pool level: memory limit, `allow_url_fopen`, and a hardened `disable_functions` list
5. Clean up test artefacts

---

## 36.2 Step 1 — Verify Current State Before Changes

Navigate to the site root and confirm the current ownership before making any changes:

```bash
cd /var/www/expertwp.help
ls -l
ls -l public_html/
```

At this point all files and directories inside `public_html/` are owned by `www-data:www-data`. This needs to change to the pool user so that PHP-FPM (running as `expertwp`) has proper read/write access without relying on `www-data` globally.

---

## 36.3 Step 2 — Transfer Ownership to the Pool User

Recursively change ownership of the entire document root to the pool user and group:

```bash
sudo chown -R expertwp:expertwp public_html/
```

Verify the change:

```bash
ls -l
```

The output should now show `expertwp expertwp` as owner and group on the `public_html/` directory itself.

---

## 36.4 Step 3 — Set Filesystem Permissions

### 3.1 — Directory Permissions

Set all directories to `770` (owner and group have full read/write/execute; others have none):

```bash
sudo find /var/www/expertwp.help/public_html/ -type d -exec chmod 770 {} \;
```

After running this, the `public_html/` directory permissions become `drwxrwx---`. If you try `ls -l public_html/` as a user who is not part of the `expertwp` group, you will get **Permission denied** — this is expected and confirms the permissions are working correctly.

### 3.2 — File Permissions

Set all files to `660` (owner and group can read/write; others have no access):

```bash
sudo find /var/www/expertwp.help/public_html/ -type f -exec chmod 660 {} \;
```

Verify the result:

```bash
ls -l public_html/
```

All WordPress core files should now show `-rw-rw----` with `expertwp:expertwp` ownership.

---

## 36.5 Step 4 — Create a Temporary `phpinfo()` Test File

To confirm that PHP-FPM is serving requests as the correct pool user and that the pool config is applied, create a temporary `phpinfo()` file inside the document root:

```bash
cd /var/www/expertwp.help/public_html/
sudo nano 37kc.php
```

File contents:

```php
<?php
phpinfo();
?>
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`). Open the file in a browser:

```
http://expertwp.help/37kc.php
```

The phpinfo output confirms:

- **PHP Version:** 8.3.9
- **Server API:** FPM/FastCGI — confirms requests are going through PHP-FPM, not mod_php
- **Configuration File Path:** `/etc/php/8.3/fpm`
- **Environment → USER:** `expertwp` — confirms PHP is running as the correct pool user
- **memory_limit:** 256M — coming from the global `server_override.ini`

---

## 36.6 Step 5 — Understanding the PHP Configuration Hierarchy

The PHP configuration on this server has two levels worth understanding clearly before editing.

**`/etc/php/8.3/fpm/conf.d/server_override.ini`** — Global PHP settings applied to all pools on the server:

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
```

Directives set with `php_admin_value` or `php_admin_flag` inside a pool config **override** these global values for that specific pool and cannot be overridden again by the application.

Inspect the global config:

```bash
sudo cat /etc/php/8.3/fpm/conf.d/server_override.ini
```

---

## 36.7 Step 6 — Pool-Level PHP Configuration

Navigate to the PHP-FPM pool configuration directory:

```bash
cd /etc/php/8.3/fpm/pool.d/
sudo nano expertwp.help.conf
```

### 6.1 — Current Baseline Directives

The bottom of the pool config file currently has these PHP directives:

```ini
;php_admin_value[sendmail_path] = /usr/sbin/sendmail -t -i -f www@my.domain.com
php_flag[display_errors] = off
php_admin_value[error_log] = /var/log/fpm-php.expertwp.log
php_admin_flag[log_errors] = on
;php_admin_value[memory_limit] = 32M
```

The `memory_limit` line is commented out, which means the 256M from `server_override.ini` applies. To cap this pool's memory independently, uncomment and set it:

```ini
php_admin_value[memory_limit] = 32M
```

Save, exit, and reload PHP-FPM:

```bash
sudo systemctl reload php8.3-fpm
```

Refreshing the phpinfo page confirms `memory_limit` now shows `32M` for this pool's Local Value while the Master Value remains 256M. This demonstrates how `php_admin_value` at the pool level takes precedence over the global ini.

To revert back to the global default, comment the line out again and reload (`fpmr`).

> **`fpmr` alias:** Throughout this setup, `fpmr` is used as a shell alias for `sudo systemctl reload php8.3-fpm`.

### 6.2 — Enable `allow_url_fopen` for WordPress

The global `server_override.ini` disables `allow_url_fopen` server-wide. WordPress requires this to be enabled for plugin/theme updates, HTTP API calls, and file operations. Enable it at the pool level only — this scopes it to this site and does not affect other pools:

```bash
sudo nano /etc/php/8.3/fpm/pool.d/expertwp.help.conf
```

Add this line at the end of the PHP directives section:

```ini
php_admin_flag[allow_url_fopen] = on
```

Save and reload:

```bash
sudo systemctl reload php8.3-fpm
```

Reload the phpinfo page and search for `allow_url_fopen`. The **Local Value** should now show `On` while the **Master Value** remains `Off` — confirming the per-pool override is working correctly.

### 6.3 — Disable Dangerous PHP Functions

WordPress does not need access to process control, shell execution, or POSIX system calls. Disabling these at the pool level significantly reduces the blast radius of any plugin vulnerability or code injection attempt.

Open the pool config:

```bash
sudo nano /etc/php/8.3/fpm/pool.d/expertwp.help.conf
```

Add the following block below the `allow_url_fopen` line:

```ini
; ENABLED FUNCTIONS
; disk_free_space

; DISABLED FUNCTIONS
php_admin_value[disable_functions] = shell_exec, opcache_get_configuration, opcache_get_status, disk_total_space, diskfreespace, dl, exec, passthru, pclose, pcntl_alarm, pcntl_exec, pcntl_fork, pcntl_get_last_error, pcntl_getpriority, pcntl_setpriority, pcntl_signal, pcntl_signal_dispatch, pcntl_sigprocmask, pcntl_sigtimedwait, pcntl_sigwaitinfo, pcntl_strerror, pcntl_waitpid, pcntl_wait, pcntl_wexitstatus, pcntl_wifcontinued, pcntl_wifexited, pcntl_wifsignaled, pcntl_wifstopped, pcntl_wstopsig, pcntl_wtermsig, popen, posix_getpwuid, posix_kill, posix_mkfifo, posix_setpgid, posix_setsid, posix_setuid, posix_uname, proc_close, proc_get_status, proc_nice, proc_open, proc_terminate, show_source, system
```

> **Important:** The `disable_functions` directive must be on a **single line** in the actual file.

Save and reload:

```bash
sudo systemctl reload php8.3-fpm
```

---

## 36.8 Step 7 — Full Pool Config PHP Directives (Final State)

After all edits, the bottom of `/etc/php/8.3/fpm/pool.d/expertwp.help.conf` should look like this:

```ini
;php_admin_value[sendmail_path] = /usr/sbin/sendmail -t -i -f www@my.domain.com
php_flag[display_errors] = off
php_admin_value[error_log] = /var/log/fpm-php.expertwp.log
php_admin_flag[log_errors] = on
;php_admin_value[memory_limit] = 32M

php_admin_flag[allow_url_fopen] = on

; ENABLED FUNCTIONS
; disk_free_space

; DISABLED FUNCTIONS
php_admin_value[disable_functions] = shell_exec, opcache_get_configuration, opcache_get_status, disk_total_space, diskfreespace, dl, exec, passthru, pclose, pcntl_alarm, pcntl_exec, pcntl_fork, pcntl_get_last_error, pcntl_getpriority, pcntl_setpriority, pcntl_signal, pcntl_signal_dispatch, pcntl_sigprocmask, pcntl_sigtimedwait, pcntl_sigwaitinfo, pcntl_strerror, pcntl_waitpid, pcntl_wait, pcntl_wexitstatus, pcntl_wifcontinued, pcntl_wifexited, pcntl_wifsignaled, pcntl_wifstopped, pcntl_wstopsig, pcntl_wtermsig, popen, posix_getpwuid, posix_kill, posix_mkfifo, posix_setpgid, posix_setsid, posix_setuid, posix_uname, proc_close, proc_get_status, proc_nice, proc_open, proc_terminate, show_source, system
```

---

## 36.9 Step 8 — Delete the phpinfo Test File

The `phpinfo()` page exposes detailed server configuration and must be removed before the site goes into production:

```bash
cd /var/www/expertwp.help/public_html/
sudo rm 37kc.php
ls
```

The working directory should contain only the standard WordPress files.

---

## 36.10 Final State Verification

```bash
ls -l /var/www/expertwp.help/public_html/
```

Expected output characteristics:

| Item | Permissions | Owner |
|------|------------|-------|
| WordPress PHP files | `-rw-rw----` (660) | `expertwp:expertwp` |
| `wp-admin/` directory | `drwxrwx---` (770) | `expertwp:expertwp` |
| `wp-content/` directory | `drwxrwx---` (770) | `expertwp:expertwp` |
| `wp-includes/` directory | `drwxrwx---` (770) | `expertwp:expertwp` |

---

## 36.11 Summary of Changes

| Component | What Changed |
|-----------|-------------|
| File ownership | Transferred from `www-data:www-data` → `expertwp:expertwp` |
| Directory permissions | Set to `770` — pool user/group full access, others none |
| File permissions | Set to `660` — pool user/group read/write, others none |
| `php_flag[display_errors]` | `off` — no error output to browser |
| `php_admin_value[error_log]` | Logs to `/var/log/fpm-php.expertwp.log` |
| `php_admin_flag[log_errors]` | `on` — errors are logged |
| `php_admin_value[memory_limit]` | Commented out — inherits 256M from `server_override.ini` |
| `php_admin_flag[allow_url_fopen]` | `on` — WordPress requires this; overrides global `Off` |
| `php_admin_value[disable_functions]` | Large list of shell/process/POSIX functions blocked |

---

| | | |
|:---|:---:|---:|
| [← Chapter 35 — PHP-FPM Pool Isolation](./35-php-fpm-pool-isolation.md) | [↑ Top](#chapter-36--php-fpm-pool-hardening-ownership-permissions--per-pool-php-configuration) | [Chapter 37 — WordPress Hardened File Permissions →](./37-wordpress-hardened-permissions.md) |
