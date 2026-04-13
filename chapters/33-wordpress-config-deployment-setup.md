[← Back to Index](../index.md)

---

# Chapter 33 — WordPress: `wp-config.php`, Deployment & Browser Setup

*Completing the WordPress installation — editing `wp-config.php` in full detail, deploying files with `rsync`, setting ownership, running the browser installation wizard, and performing first-login housekeeping.*

---

## Contents

- [33.1 Step 1 — Rename the Sample Config and Fetch Salts](#331-step-1--rename-the-sample-config-and-fetch-salts)
- [33.2 Step 2 — Edit wp-config.php in Nano](#332-step-2--edit-wp-configphp-in-nano)
  - [2a — Database Settings](#2a--database-settings)
  - [2b — Authentication Keys and Salts](#2b--authentication-keys-and-salts)
  - [2c — Database Table Prefix](#2c--database-table-prefix)
  - [2d — Operational Constants](#2d--operational-constants)
- [33.3 Step 3 — Deploy WordPress Files to the Document Root](#333-step-3--deploy-wordpress-files-to-the-document-root)
- [33.4 Step 4 — Set File Ownership](#334-step-4--set-file-ownership)
- [33.5 Step 5 — Browser-Based WordPress Installation](#335-step-5--browser-based-wordpress-installation)
  - [5a — Language Selection](#5a--language-selection)
  - [5b — Site Information Form](#5b--site-information-form)
  - [5c — Success Screen](#5c--success-screen)
  - [5d — Confirmation Email](#5d--confirmation-email)
- [33.6 Step 6 — First-Login Housekeeping](#336-step-6--first-login-housekeeping)
  - [6a — Log In to the Dashboard](#6a--log-in-to-the-dashboard)
  - [6b — General Settings — Verify Site URLs](#6b--general-settings--verify-site-urls)
  - [6c — Permalinks — Set Post Name Structure](#6c--permalinks--set-post-name-structure)
  - [6d — Delete Default Plugins](#6d--delete-default-plugins)
  - [6e — Delete Unused Themes](#6e--delete-unused-themes)
  - [6f — Update Admin Profile Display Name](#6f--update-admin-profile-display-name)
- [33.7 Step 7 — Install Maintenance Mode Plugin](#337-step-7--install-maintenance-mode-plugin)
- [33.8 Summary](#338-summary)

---

## 33.1 Step 1 — Rename the Sample Config and Fetch Salts

Inside the extracted `~/wordpress/` directory, the sample config is renamed to the live config file:

```bash
mv wp-config-sample.php wp-config.php
```

Before opening the file, the WordPress secret-key API is queried to generate a fresh set of authentication salts. The `-s` flag suppresses `curl`'s download progress output:

```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

The output is 8 PHP `define()` statements with random values. Copy the entire block to a local reference file alongside the credentials, ready to paste into `wp-config.php`.

---

## 33.2 Step 2 — Edit `wp-config.php` in Nano

```bash
nano wp-config.php
```

The file has four main sections that need editing.

### 2a — Database Settings

Replace the placeholder values with the credentials generated in Chapter 32:

```php
// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'expertwp_help' );

/** Database username */
define( 'DB_USER', 'OhxVoPWnKo' );

/** Database password */
define( 'DB_PASSWORD', 'N0Szm58iVP' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
```

`DB_HOST` is left as `localhost` — the database server and web server are on the same machine, communicating via a local socket rather than a network connection.

### 2b — Authentication Keys and Salts

Scrolling down, the keys section contains 8 placeholder `define()` lines that all read `'put your unique phrase here'`. Delete the entire block and replace it with the output from the `curl` salt API call.

These keys are used by WordPress to cryptographically sign and verify cookies. Without unique values, cookie forgery becomes trivial. Changing them at any future point will invalidate all currently logged-in sessions, forcing everyone to re-authenticate.

### 2c — Database Table Prefix

Below the salts section is the table prefix variable:

```php
$table_prefix = 'wp_';
```

Change it to a short random string to make database table names unpredictable. The default `wp_` prefix is well-known, and automated SQL injection attacks often target it directly:

```php
$table_prefix = 'bF6_';
```

> **Convention:** The prefix must end with an underscore (`_`), contain only letters, numbers, and underscores, and be reasonably short to avoid truncation issues.

### 2d — Operational Constants

The file contains a comment: `/* Add any custom values between this line and the "stop editing" line. */`

Add the four operational constants in this zone, before the `require_once ABSPATH . 'wp-settings.php';` line at the very bottom:

```php
/** Allow Direct Updating Without FTP */
define('FS_METHOD', 'direct');

/** Disable Editing of Themes and Plugins Using the Built In Editor */
define('DISALLOW_FILE_EDIT', 'true');

/** Turn Off Automatic Core Updates */
define('WP_AUTO_UPDATE_CORE', false);
define('AUTOMATIC_UPDATER_DISABLED', 'true');
```

> **Placement matters:** These constants must be placed **before** the `require_once ABSPATH . 'wp-settings.php';` line at the very bottom of the file. Anything placed after the `require_once` will never execute.

Save and exit nano (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 33.3 Step 3 — Deploy WordPress Files to the Document Root

Navigate back to the home directory and run `rsync` to copy the WordPress files:

```bash
cd
sudo rsync -artv wordpress/ /var/www/expertwp.help/public_html/
```

After the transfer completes, clean up the home directory:

```bash
sudo rm -rf latest.tar.gz wordpress/
ls
```

Output after cleanup: only the `MySQLTuner` directory remains.

---

## 33.4 Step 4 — Set File Ownership

Navigate to the site directory and transfer ownership of `public_html/` to `www-data`:

```bash
cd /var/www/expertwp.help/
ls -l
```

Before:

```
drwxr-xr-x 5 andrew andrew 4096 Jul 22 19:43 public_html
```

```bash
sudo chown -R www-data:www-data public_html/
ls -l
```

After:

```
drwxr-xr-x 5 www-data www-data 4096 Jul 22 19:43 public_html
```

Every file and directory now shows `www-data www-data` as owner and group. This is what enables WordPress to write files directly — creating uploads, generating cache, updating itself — without falling back to FTP.

---

## 33.5 Step 5 — Browser-Based WordPress Installation

### Important Note — Canonical URL Decision

Before opening a browser, decide which URL form the site will use: `http://expertwp.help` or `http://www.expertwp.help`. **This is critical** — whichever URL is used to complete the installation wizard becomes the permanent WordPress "Site Address" stored in the database. Changing it later requires either a database edit or a search-and-replace with WP-CLI. The bare domain (`expertwp.help`) was chosen.

### 5a — Language Selection

Navigate to `http://expertwp.help` in the browser. WordPress detects the empty database and redirects automatically to the installation wizard at `/wp-admin/install.php`. The first screen presents a language selector — **English (United States)** is selected and **Continue** is clicked.

### 5b — Site Information Form

| Field | Value entered |
|-------|--------------|
| Site Title | `Expert WordPress Help` |
| Username | `E5xoPx1zgx6cdg1kF5JErvFsJlhpHs` (from generated credentials) |
| Password | `ZeBNjMBtWCmRqZhaTgOVkdMl4ptnqb` (from generated credentials) |
| Your Email | `vpsxpert@gmail.com` |
| Search Engine Visibility | Left unchecked (allow indexing) |

The username and password are copied from the local credential reference file to avoid typos. **Install WordPress** is clicked.

### 5c — Success Screen

WordPress confirms installation with a "Success!" screen. A **Log In** link is presented, which redirects to `/wp-login.php`.

### 5d — Confirmation Email

WordPress automatically sends an installation confirmation email to the admin address. This confirms the mail server configured in Chapter 19 is functioning correctly.

---

## 33.6 Step 6 — First-Login Housekeeping

### 6a — Log In to the Dashboard

Navigate to `http://expertwp.help/wp-login.php`, enter the admin username and password, and log in. The WordPress admin dashboard loads at `/wp-admin/`.

### 6b — General Settings — Verify Site URLs

Navigate to **Settings → General** (`/wp-admin/options-general.php`).

Confirm that both **WordPress Address (URL)** and **Site Address (URL)** show `http://expertwp.help` — the bare domain without `www`. No changes are needed.

### 6c — Permalinks — Set Post Name Structure

Navigate to **Settings → Permalinks** (`/wp-admin/options-permalink.php`).

The default permalink structure is **Plain** (`/?p=123`), which is not human-readable and is poor for SEO. Change it to **Post name**:

```
http://expertwp.help/sample-post/
```

Click **Save Changes**. WordPress writes the rewrite rules; the Nginx `try_files` directive already configured in the server block handles URL routing without any additional Nginx changes needed.

### 6d — Delete Default Plugins

Navigate to **Plugins → Installed Plugins** (`/wp-admin/plugins.php`).

Two default plugins ship with WordPress — **Akismet Anti-Spam** and **Hello Dolly**. Neither is needed:

- Select all plugins using the checkbox at the top
- From the **Bulk actions** dropdown, select **Delete**
- Click **Apply** and confirm

Both plugins are permanently removed.

### 6e — Delete Unused Themes

Navigate to **Appearance → Themes** (`/wp-admin/themes.php`).

WordPress 6.6 ships with three bundled themes: **Twenty Twenty-Four** (active), **Twenty Twenty-Three**, and **Twenty Twenty-Two**. The two inactive themes are deleted to reduce the attack surface — each theme is a PHP codebase that can contain vulnerabilities.

For each inactive theme:
1. Hover over the theme thumbnail
2. Click **Theme Details** to open the detail panel
3. Click the **Delete** button in the bottom-right corner
4. Confirm the deletion

After both are removed, only Twenty Twenty-Four remains.

### 6f — Update Admin Profile Display Name

Navigate to **Users → All Users**, then click **Edit** under the admin account.

The admin username cannot be changed — WordPress does not allow username modification after creation. However, the **Nickname** and **Display name publicly as** fields can be updated:

- **Nickname:** Changed from the random username to `Andrew`
- **Display name publicly as:** Updated to show `Andrew`

Click **Update Profile**. The admin bar now shows "Howdy, Andrew" instead of the random username string, while the login username remains the unguessable random string.

---

## 33.7 Step 7 — Install Maintenance Mode Plugin

While the site is still being set up and before SSL is configured, a maintenance mode plugin is installed to show a holding page to visitors.

Navigate to **Plugins → Add New Plugin** and search for `maintenance mode`.

The **Maintenance** plugin by WebFactory Ltd is selected (900,000+ active installations). Click **Install Now**, then **Activate**.

After activation the admin bar shows **Maintenance is On** in orange. Public visitors to `http://expertwp.help` will see the maintenance holding page. Logged-in admin users continue to see the live site normally.

---

## 33.8 Summary

| Task | Status |
|------|--------|
| `wp-config.php` database credentials | Filled in |
| `wp-config.php` authentication salts | Replaced with API-generated values |
| `wp-config.php` table prefix | Randomised to `bF6_` |
| `wp-config.php` operational constants | Set (`FS_METHOD`, `DISALLOW_FILE_EDIT`, auto-updates off) |
| WordPress files deployed to `public_html/` | Done via `rsync` |
| Source files cleaned up from `~` | Done (`rm -rf`) |
| File ownership | Set to `www-data:www-data` recursively |
| Browser installation wizard | Completed — site title, admin credentials, email set |
| Confirmation email received | Confirmed — mail server working |
| Permalinks | Set to Post name structure |
| Default plugins (Akismet, Hello Dolly) | Deleted |
| Unused themes (TT2, TT3) | Deleted |
| Admin display name | Set to `Andrew` (login username unchanged) |
| Maintenance mode | Active — holding page live for public visitors |

The next step is configuring SSL with Let's Encrypt to upgrade the site from HTTP to HTTPS.

---

| | | |
|:---|:---:|---:|
| [← Chapter 32 — MariaDB Setup & WordPress Prep](./32-mariadb-wordpress-installation-prep.md) | [↑ Top](#chapter-33--wordpress-wp-configphp-deployment--browser-setup) | [Chapter 34 — WordPress Hardening Introduction →](./34-wordpress-hardening-introduction.md) |
