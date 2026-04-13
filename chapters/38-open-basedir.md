[← Back to Index](../index.md)

---

# Chapter 38 — Restricting PHP File Access with `open_basedir`

*Sandboxing PHP's filesystem access to the site's own directories using `open_basedir`, with a site-specific `tmp/` directory replacing the shared system `/tmp`.*

---

## Contents

- [38.1 Overview](#381-overview)
- [38.2 Step 1 — Create a Site-Specific tmp/ Directory](#382-step-1--create-a-site-specific-tmp-directory)
- [38.3 Step 2 — Configure open_basedir, upload_tmp_dir, and sys_temp_dir](#383-step-2--configure-open_basedir-upload_tmp_dir-and-sys_temp_dir)
- [38.4 Step 3 — Reload PHP-FPM](#384-step-3--reload-php-fpm)
- [38.5 Summary](#385-summary)

---

## 38.1 Overview

`open_basedir` is a PHP configuration directive that restricts the files PHP can access to a specific directory tree or a list of defined directories. It is configured per site inside the PHP-FPM pool file, which means each site gets its own isolated file access boundary.

**Why this matters:**

- Prevents PHP scripts from reading, writing, or including files outside the site's allowed directories — even if a vulnerability in a plugin or theme is exploited.
- Complements the pool user isolation already in place: even if an attacker gains code execution, they cannot traverse to other sites or sensitive OS paths.
- The performance overhead from the additional filesystem boundary checks is minimal and can be further offset by OPcache.

> **Note:** WordPress requires access to two locations: the document root (for all site files) and a writable temporary directory (for file uploads, plugin installs, etc.). Using the system `/tmp` directory is dangerous because it is shared across all users and processes on the server. The correct approach is to create a **site-specific `tmp/` directory** inside the site's web root and point both `upload_tmp_dir` and `sys_temp_dir` there, then include only that path in `open_basedir`.

---

## 38.2 Step 1 — Create a Site-Specific `tmp/` Directory

Navigate to the site's web root and create the `tmp/` directory. Set its ownership to the site's pool user and restrict permissions so only the pool user can read and write to it:

```bash
cd /var/www/expertwp.help/
sudo mkdir tmp/
sudo chown expertwp:expertwp tmp/
sudo chmod 770 tmp/
ls -l
```

Expected output:

```
drwxrwx--- 5 expertwp expertwp 4096 Jul 23 20:02 public_html
drwxrwx--- 2 expertwp expertwp 4096 Jul 23 21:14 tmp
```

Both `public_html/` and `tmp/` are owned by the pool user `expertwp` with no access granted to others.

---

## 38.3 Step 2 — Configure `open_basedir`, `upload_tmp_dir`, and `sys_temp_dir` in the Pool File

Open the site's PHP-FPM pool configuration file:

```bash
cd /etc/php/8.3/fpm/pool.d/
sudo nano expertwp.help.conf
```

Add the following three directives at the bottom of the pool file, within the PHP admin values section:

```ini
php_admin_value[upload_tmp_dir] = /var/www/expertwp.help/tmp/
php_admin_value[sys_temp_dir]   = /var/www/expertwp.help/tmp/
php_admin_value[open_basedir]   = /var/www/expertwp.help/public_html/:/var/www/expertwp.help/tmp/
```

**What each directive does:**

| Directive | Purpose |
|-----------|---------|
| `upload_tmp_dir` | Overrides where PHP stores files during multipart upload before they are moved by the application. Points to the site-local `tmp/` instead of `/tmp`. |
| `sys_temp_dir` | Overrides the system-wide temporary directory used by PHP functions like `tempnam()` and `sys_get_temp_dir()`. Points away from the global `/tmp`. |
| `open_basedir` | Defines the complete list of directories PHP is permitted to access. Paths are separated by `:`. PHP can only touch files inside `public_html/` and the site-local `tmp/` — nothing else on the filesystem. |

The final state of the relevant section in the pool config:

```ini
php_flag[display_errors]              = off
php_admin_value[error_log]            = /var/log/fpm-php.expertwp.log
php_admin_flag[log_errors]            = on

; ENABLED FUNCTIONS
; disk_free_space

; DISABLED FUNCTIONS
php_admin_value[disable_functions]    = shell_exec, opcache_get_configuration, ...

php_admin_value[upload_tmp_dir]       = /var/www/expertwp.help/tmp/
php_admin_value[sys_temp_dir]         = /var/www/expertwp.help/tmp/
php_admin_value[open_basedir]         = /var/www/expertwp.help/public_html/:/var/www/expertwp.help/tmp/
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 38.4 Step 3 — Reload PHP-FPM

```bash
sudo systemctl reload php8.3-fpm
```

Or using the alias:

```bash
fpmr
```

---

## 38.5 Summary

At this point, PHP-FPM for this site is fully sandboxed at the filesystem level:

- PHP can only access files inside `public_html/` and the site-local `tmp/`.
- All file uploads go through the site-local `tmp/` — never the shared system `/tmp`.
- Even if a malicious script attempts to open or include a file outside these paths (e.g. `/etc/passwd`, another site's `wp-config.php`), PHP will refuse with a permission error.
- This is a **per-pool** setting, so each site on the server has its own `open_basedir` boundary completely independent of others.

> **Automation note:** When scripting this, parameterise the site name so the same block of commands works for any site. The three pool directives and the `tmp/` setup are the two atomic steps to automate.

---

| | | |
|:---|:---:|---:|
| [← Chapter 37 — WordPress Hardened File Permissions](./37-wordpress-hardened-permissions.md) | [↑ Top](#chapter-38--restricting-php-file-access-with-open_basedir) | [Chapter 39 — SSL/TLS Certificates →](./39-ssl-certificates.md) |
