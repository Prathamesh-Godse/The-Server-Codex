[← Back to Index](../index.md)

---

# Chapter 37 — WordPress Hardened File Permissions: Standard vs Hardened Mode

*The two permission models for a WordPress site under a dedicated PHP-FPM pool — standard (writable) for development and updates, hardened (read-only core) for production. Exact commands to apply and switch between both.*

> **Scope:** Ubuntu 24.04 · Nginx · PHP 8.3 FPM · WordPress

---

## Contents

- [37.1 Overview](#371-overview)
- [37.2 Trade-offs](#372-trade-offs)
- [37.3 Starting State (Standard Permissions)](#373-starting-state-standard-permissions)
- [37.4 Applying Hardened Permissions](#374-applying-hardened-permissions)
  - [Step 1 — Lock Down All Core Directories to 550](#step-1--lock-down-all-core-directories-to-550)
  - [Step 2 — Lock Down All Core Files to 440](#step-2--lock-down-all-core-files-to-440)
  - [Step 3 — Restore wp-content Files to 660](#step-3--restore-wp-content-files-to-660)
  - [Step 4 — Restore wp-content Directories to 770](#step-4--restore-wp-content-directories-to-770)
  - [Step 5 — Verify the Final Hardened State](#step-5--verify-the-final-hardened-state)
- [37.5 Reverting to Standard Permissions](#375-reverting-to-standard-permissions)
- [37.6 Summary of Commands](#376-summary-of-commands)

---

## 37.1 Overview

With the PHP-FPM pool set up and WordPress ownership transferred to the pool user (`expertwp`), there are two distinct permission strategies depending on the site's lifecycle stage:

| Mode | Dirs | Files | wp-content | Use When |
|------|------|-------|-----------|---------|
| **Standard** | `770` | `660` | `770` / `660` | Active development, update phase |
| **Hardened** | `550` | `440` | `770` / `660` | Production, stable site |

In both modes, file ownership stays `expertwp:expertwp` and `wp-config.php` ends up at `440`.

The key idea in the hardened model is removing write access from the WordPress **core** — everything except `wp-content/` — while keeping `wp-content/` fully writable. PHP-FPM can still read and execute all core files; it just can't modify them. `wp-content/` must stay writable because WordPress writes uploads, installs plugins and themes, and caches content there.

---

## 37.2 Trade-offs

**Advantages of hardened permissions:**

- WordPress core files cannot be overwritten by a compromised plugin or a PHP injection exploit
- File injection attacks are unable to drop malicious files into core directories
- Provides a strong secondary defence layer beneath Nginx's URL-level rules

**Disadvantages of hardened permissions:**

- Automatic WordPress core updates fail — the updater cannot overwrite read-only files
- Plugin and theme updates that write to locations outside `wp-content/` will break
- Manual updates require temporarily switching back to standard permissions, updating, then re-hardening
- Maintenance mode writes to the root — this can also fail if the directory is not writable

The hardened model is most appropriate for a stable production site where updates are managed manually or via a deployment pipeline. For sites still being built out or frequently updated, keep standard permissions and harden once the site is stable.

---

## 37.3 Starting State (Standard Permissions)

The site at the start of this stage has the standard permissions applied from Chapter 36:

```bash
cd /var/www/expertwp.help
ls -l
ls -l public_html/
```

Expected output:

- `public_html/` directory: `drwxrwx---` (`770`) — `expertwp:expertwp`
- All WordPress files: `-rw-rw----` (`660`) — `expertwp:expertwp`
- All WordPress directories: `drwxrwx---` (`770`) — `expertwp:expertwp`

---

## 37.4 Applying Hardened Permissions

### Step 1 — Lock Down All Core Directories to 550

Remove write permission from every directory under `public_html/`. The `550` mode (`r-xr-x---`) allows the owner and group to read and traverse directories but not write to them:

```bash
sudo find /var/www/expertwp.help/public_html/ -type d -exec chmod 550 {} \;
```

Verify:

```bash
ls -l public_html/
```

Directories now show `dr-xr-x---`. Files are still `660` at this point.

### Step 2 — Lock Down All Core Files to 440

Remove write permission from every file under `public_html/`. The `440` mode (`r--r-----`) makes all files read-only for owner and group:

```bash
sudo find /var/www/expertwp.help/public_html/ -type f -exec chmod 440 {} \;
```

Verify:

```bash
ls -l public_html/
```

All WordPress PHP files now show `-r--r-----`. The entire WordPress core is now read-only.

### Step 3 — Restore `wp-content` Files to 660

`wp-content/` must be writable — WordPress uploads media here, and plugin/theme files are read and written from this location. Restore file permissions for everything inside `wp-content/`:

```bash
sudo find /var/www/expertwp.help/public_html/wp-content/ -type f -exec chmod 660 {} \;
```

### Step 4 — Restore `wp-content` Directories to 770

Restore directory permissions inside `wp-content/` so WordPress can create subdirectories for uploads, cache, and plugin data:

```bash
sudo find /var/www/expertwp.help/public_html/wp-content/ -type d -exec chmod 770 {} \;
```

### Step 5 — Verify the Final Hardened State

```bash
ls -l public_html/
```

Expected hardened state:

| Path | Permissions | Notes |
|------|------------|-------|
| `wp-admin/` | `dr-xr-x---` (550) | Read/execute only — no writes |
| `wp-includes/` | `dr-xr-x---` (550) | Read/execute only — no writes |
| `wp-content/` | `dr-xr-x---` (550) | The top-level dir itself is 550 |
| `wp-content/uploads/` | `drwxrwx---` (770) | Writable — media uploads |
| `wp-content/plugins/` | `drwxrwx---` (770) | Writable — plugin files |
| `wp-content/themes/` | `drwxrwx---` (770) | Writable — theme files |
| Core `.php` files | `-r--r-----` (440) | Read-only |
| `wp-content/` files | `-rw-rw----` (660) | Read/write |
| `wp-config.php` | `-r--r-----` (440) | Read-only |

---

## 37.5 Reverting to Standard Permissions

When updates or maintenance are needed, revert to standard permissions, perform the update, then re-apply the hardened mode:

```bash
# Reset all dirs to 770
sudo find /var/www/expertwp.help/public_html/ -type d -exec chmod 770 {} \;

# Reset all files to 660
sudo find /var/www/expertwp.help/public_html/ -type f -exec chmod 660 {} \;
```

Verify:

```bash
ls -l public_html/
```

All directories back to `drwxrwx---` (770), all files back to `-rw-rw----` (660). WordPress updates and plugin installs will work normally in this state. After completing updates, re-apply the hardened mode by repeating the four steps in section 37.4.

---

## 37.6 Summary of Commands

### Standard Mode (Development / Update Phase)

```bash
sudo find /var/www/expertwp.help/public_html/ -type d -exec chmod 770 {} \;
sudo find /var/www/expertwp.help/public_html/ -type f -exec chmod 660 {} \;
```

### Hardened Mode (Production)

```bash
# Lock core
sudo find /var/www/expertwp.help/public_html/ -type d -exec chmod 550 {} \;
sudo find /var/www/expertwp.help/public_html/ -type f -exec chmod 440 {} \;

# Keep wp-content writable
sudo find /var/www/expertwp.help/public_html/wp-content/ -type f -exec chmod 660 {} \;
sudo find /var/www/expertwp.help/public_html/wp-content/ -type d -exec chmod 770 {} \;
```

### `wp-config.php` (Always)

Regardless of which mode is active, `wp-config.php` should remain read-only. It contains database credentials and secret keys and should never be writable by PHP processes:

```bash
sudo chmod 440 /var/www/expertwp.help/public_html/wp-config.php
```

### Scriptable Functions

```bash
SITE_ROOT="/var/www/expertwp.help/public_html"
WP_CONTENT="${SITE_ROOT}/wp-content"

harden_permissions() {
    sudo find "$SITE_ROOT" -type d -exec chmod 550 {} \;
    sudo find "$SITE_ROOT" -type f -exec chmod 440 {} \;
    sudo find "$WP_CONTENT" -type f -exec chmod 660 {} \;
    sudo find "$WP_CONTENT" -type d -exec chmod 770 {} \;
    sudo chmod 440 "${SITE_ROOT}/wp-config.php"
}

standard_permissions() {
    sudo find "$SITE_ROOT" -type d -exec chmod 770 {} \;
    sudo find "$SITE_ROOT" -type f -exec chmod 660 {} \;
}
```

These functions are non-interactive and safe to run idempotently. Always run `harden_permissions` as the final step before a site goes live and `standard_permissions` before any WordPress core or plugin update.

---

| | | |
|:---|:---:|---:|
| [← Chapter 36 — PHP-FPM Pool Hardening](./36-php-fpm-pool-hardening.md) | [↑ Top](#chapter-37--wordpress-hardened-file-permissions-standard-vs-hardened-mode) | [Chapter 38 — Restricting PHP File Access with open_basedir →](./38-open-basedir.md) |
