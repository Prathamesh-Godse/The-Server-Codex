[← Back to Index](../index.md)

---

# Chapter 47 — Hardening WordPress: Disabling File Modifications via `DISALLOW_FILE_MODS`

*Adding the `DISALLOW_FILE_MODS` constant to `wp-config.php` to prevent WordPress from installing, updating, or modifying any files from the admin dashboard — the most impactful single-line WordPress hardening measure.*

---

## Contents

- [47.1 Overview](#471-overview)
- [47.2 What DISALLOW_FILE_MODS Does](#472-what-disallow_file_mods-does)
- [47.3 Implementation](#473-implementation)
  - [Step 1 — Navigate to the WordPress Site Root](#step-1--navigate-to-the-wordpress-site-root)
  - [Step 2 — Open wp-config.php](#step-2--open-wp-configphp)
  - [Step 3 — Add the DISALLOW_FILE_MODS Constant](#step-3--add-the-disallow_file_mods-constant)
  - [Step 4 — Reload PHP-FPM](#step-4--reload-php-fpm)
- [47.4 Constant Reference](#474-constant-reference)
- [47.5 Important Considerations](#475-important-considerations)

---

## 47.1 Overview

With PHP pool isolation, hardened file permissions, and Nginx security directives already in place, this step locks down the WordPress application layer itself. The `DISALLOW_FILE_MODS` constant prevents WordPress from writing to the filesystem at all — no plugin installs, no theme updates, no file edits — directly from the WordPress admin dashboard. This is one of the most effective single-line hardening measures available in `wp-config.php`.

---

## 47.2 What `DISALLOW_FILE_MODS` Does

Adding this constant to `wp-config.php` has the following effects:

- Disables WordPress's ability to modify files on the filesystem
- Removes the ability to install or update plugins and themes from the WordPress dashboard
- Prevents unauthorised or compromised code from modifying core WordPress files
- Has a significant impact on WordPress functionality — plugin and theme management must be done manually via the server

> **Note:** This constant should only be set once the site is fully configured and production-ready. Any plugin or theme updates will need to be performed manually via SSH.

---

## 47.3 Implementation

### Step 1 — Navigate to the WordPress Site Root

```bash
cd /var/www/expertwp.help/public_html
```

Verify the WordPress installation is present:

```bash
ls -la
```

The output should show standard WordPress core files and directories (`wp-admin/`, `wp-content/`, `wp-includes/`, `wp-config.php`, etc.), all owned by the site's pool user (`expertwp:expertwp`).

### Step 2 — Open `wp-config.php`

```bash
sudo nano wp-config.php
```

### Step 3 — Add the `DISALLOW_FILE_MODS` Constant

Locate the custom configuration block — the section between the database settings and the `/* That's all, stop editing! */` line. This block already contains the constants added in Chapter 33:

```php
/** Allow Direct Updating Without FTP */
define('FS_METHOD', 'direct');

/** Disable Editing of Themes and Plugins Using the Built In Editor */
define('DISALLOW_FILE_EDIT', 'true');

/** TURN OFF AUTOMATIC UPDATES */
define('WP_AUTO_UPDATE_CORE', 'false');
define('AUTOMATIC_UPDATER_DISABLED', 'true');
```

Append the `DISALLOW_FILE_MODS` constant immediately after:

```php
/** DISALLOW FILE MODS */
define('DISALLOW_FILE_MODS', 'true');
```

The complete custom block in `wp-config.php`:

```php
/** Allow Direct Updating Without FTP */
define('FS_METHOD', 'direct');

/** Disable Editing of Themes and Plugins Using the Built In Editor */
define('DISALLOW_FILE_EDIT', 'true');

/** TURN OFF AUTOMATIC UPDATES */
define('WP_AUTO_UPDATE_CORE', 'false');
define('AUTOMATIC_UPDATER_DISABLED', 'true');

/** DISALLOW FILE MODS */
define('DISALLOW_FILE_MODS', 'true');
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 4 — Reload PHP-FPM

```bash
sudo systemctl reload php8.3-fpm
```

Or using the alias: `fpmr`

---

## 47.4 Constant Reference

| Constant | Value | Effect |
|----------|-------|--------|
| `FS_METHOD` | `'direct'` | Allows direct filesystem writes without FTP credentials |
| `DISALLOW_FILE_EDIT` | `'true'` | Removes the theme/plugin editor from the WordPress admin UI |
| `WP_AUTO_UPDATE_CORE` | `'false'` | Disables automatic WordPress core updates |
| `AUTOMATIC_UPDATER_DISABLED` | `'true'` | Disables the entire automatic updater subsystem |
| `DISALLOW_FILE_MODS` | `'true'` | Prevents WordPress from installing, updating, or modifying any files |

---

## 47.5 Important Considerations

- `DISALLOW_FILE_MODS` supersedes `DISALLOW_FILE_EDIT` — when file mods are disabled, the file editor is also implicitly disabled.
- Plugin and theme updates must now be performed manually: download the update, extract it via SSH, and place it in the appropriate directory under `wp-content/`.
- WordPress core updates also need to be applied manually or via WP-CLI when this constant is active.
- This constant is intentionally disruptive to WordPress's built-in update mechanism — that is the point. Combined with rate limiting on `wp-login.php` and blocked PHP execution in `wp-content/` directories, it significantly reduces the attack surface of the WordPress installation.

---

| | | |
|:---|:---:|---:|
| [← Chapter 46 — Hotlinking Protection](./46-hotlinking-protection.md) | [↑ Top](#chapter-47--hardening-wordpress-disabling-file-modifications-via-disallow_file_mods) | [Chapter 48 — Database Privilege Hardening →](./48-database-hardening.md) |
