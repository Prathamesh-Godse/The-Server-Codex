[← Back to Index](../index.md)

---

# Chapter 32 — MariaDB Database Setup & WordPress Installation Preparation

*Building familiarity with the MariaDB shell — creating databases, users, and privileges — then preparing a production WordPress installation: generating credentials, creating the database, downloading WordPress, and configuring `wp-config.php`.*

---

## Contents

- [32.1 Pre-Installation Checklist](#321-pre-installation-checklist)
- [32.2 Part 1 — MariaDB Fundamentals](#322-part-1--mariadb-fundamentals)
  - [Connecting to MariaDB](#connecting-to-mariadb)
  - [Exploring the Default Databases](#exploring-the-default-databases)
  - [Creating a Database](#creating-a-database)
  - [Creating a User and Granting Privileges](#creating-a-user-and-granting-privileges)
  - [Flushing Privileges](#flushing-privileges)
  - [Verifying Grants](#verifying-grants)
  - [Listing All Users](#listing-all-users)
  - [Switching Databases and Inspecting Tables](#switching-databases-and-inspecting-tables)
  - [Cleaning Up — Dropping the Test Database and User](#cleaning-up--dropping-the-test-database-and-user)
- [32.3 Part 2 — WordPress Site Installation Workflow](#323-part-2--wordpress-site-installation-workflow)
  - [Step 1 — Verify the Document Root Structure](#step-1--verify-the-document-root-structure)
  - [Step 2 — Generate Random Database Credentials](#step-2--generate-random-database-credentials)
  - [Step 3 — Create the Production Database and User](#step-3--create-the-production-database-and-user)
  - [Step 4 — Generate WordPress Admin Credentials](#step-4--generate-wordpress-admin-credentials)
  - [Step 5 — Download and Extract WordPress](#step-5--download-and-extract-wordpress)
  - [Step 6 — Configure wp-config.php](#step-6--configure-wp-configphp)
  - [Step 7 — Deploy WordPress Files to the Document Root](#step-7--deploy-wordpress-files-to-the-document-root)
  - [Step 8 — Set File Ownership](#step-8--set-file-ownership)
- [32.4 Summary](#324-summary)
- [32.5 Reference — Useful MariaDB Commands](#325-reference--useful-mariadb-commands)

---

## 32.1 Pre-Installation Checklist

Before beginning WordPress installation, the following must be in place:

- Server configured to send mail
- DNS A and CNAME records pointing to the server IP
- Non-secure (HTTP) Nginx server block created and active

DNS was verified with `nslookup` before proceeding:

```bash
nslookup expertwp.help
nslookup www.expertwp.help
```

Both the bare domain and `www` resolved to the same server IP (`95.179.129.251`), confirming the A record and CNAME are propagated. Both were set to DNS-only (no proxy) in Cloudflare.

---

## 32.2 Part 1 — MariaDB Fundamentals

MariaDB is a community-developed fork of MySQL. It is open source, supports connection pools of 200,000+ simultaneous connections, replicates faster than MySQL, and is the database of choice for this stack. The `mysql` client command works identically with MariaDB.

A few syntax rules to keep in mind before working in the MariaDB shell:

- Every SQL statement ends with a semicolon (`;`). Without it, MariaDB waits for more input.
- SQL keywords are case-insensitive (`CREATE DATABASE` and `create database` are identical).
- Database and user names should use underscores (`_`) as separators, never dots (`.`). A dot has special meaning in SQL and will break queries if used in an identifier.

### Connecting to MariaDB

MariaDB is accessed from the shell using the `mysql` client with `sudo`. On Ubuntu, the `root` MariaDB account authenticates via the system's `sudo` mechanism rather than a password:

```bash
sudo mysql
```

### Exploring the Default Databases

```sql
show databases;
```

Output:

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set
```

These four databases are MariaDB internals and should never be modified or deleted. `mysql` holds user accounts and privileges. `information_schema` is a read-only metadata layer. `performance_schema` and `sys` are diagnostic views.

### Creating a Database

```sql
create database test123;
```

Database names follow the same underscore convention as user names — no dots, no spaces.

### Creating a User and Granting Privileges

The `GRANT` statement both creates a user (if they don't already exist) and assigns privileges in a single operation:

```sql
grant all privileges on test123.* to 'andrew'@'localhost' identified by 'password456';
```

Breaking this down:

- `grant all privileges on test123.*` — Grants every privilege on all tables within the `test123` database.
- `to 'andrew'@'localhost'` — The user `andrew` who can only connect from `localhost`. The `@'localhost'` restriction is deliberate — it means this user cannot connect remotely, which is correct for a WordPress database user that php-fpm contacts over a local socket.
- `identified by 'password456'` — Sets the account password.

### Flushing Privileges

After any privilege modification, the privilege tables need to be reloaded into memory:

```sql
flush privileges;
```

### Verifying Grants

```sql
show grants for 'andrew'@'localhost';
```

Output:

```
+--------------------------------------------------------------------------------------------+
| Grants for andrew@localhost                                                                |
+--------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'andrew'@'localhost' IDENTIFIED BY PASSWORD '*614D8D38...'          |
| GRANT ALL PRIVILEGES ON `test123`.* TO 'andrew'@'localhost'                               |
+--------------------------------------------------------------------------------------------+
```

The first row is the base `USAGE` grant (the user exists and can log in). The second confirms full access to `test123`. The password is shown as a hashed value, never in plaintext.

### Listing All Users

```sql
select host, user from mysql.user;
```

Output:

```
+-----------+-------------+
| Host      | User        |
+-----------+-------------+
| localhost | andrew      |
| localhost | mariadb.sys |
| localhost | mysql       |
| localhost | root        |
+-----------+-------------+
```

All users are bound to `localhost`, which is correct — none should be accessible remotely.

### Switching Databases and Inspecting Tables

```sql
use sys;
show tables;
describe processlist;
```

`use db_name;` changes the active database. `show tables;` lists all tables in it. `describe table_name;` outputs the column structure — field names, data types, and constraints.

### Cleaning Up — Dropping the Test Database and User

After the exploration, the test resources were removed to leave the system clean before creating the production database:

```sql
drop database test123;
show databases;
```

```sql
drop user 'andrew'@'localhost';
select host, user from mysql.user;
```

`drop database` permanently removes the database and all its tables. `drop user` removes the account entirely. Exit the shell:

```sql
exit
```

---

## 32.3 Part 2 — WordPress Site Installation Workflow

The site installation follows this sequence:

1. Verify the document root structure
2. Generate a random database username and password
3. Create the production database and user in MariaDB
4. Generate WordPress admin credentials
5. Download WordPress and extract files
6. Configure `wp-config.php`
7. Copy files to the document root
8. Set file ownership

### Step 1 — Verify the Document Root Structure

```bash
tree /var/www
```

Output:

```
/var/www
├── expertwp.help
│   └── public_html
└── html
    └── index.nginx-debian.html
```

The `public_html` directory is empty, ready to receive WordPress files.

### Step 2 — Generate Random Database Credentials

Credentials are generated from the system's cryptographically secure random source:

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 10 | head -n 2
```

**What this pipeline does:**

| Component | Purpose |
|-----------|---------|
| `cat /dev/urandom` | Streams raw random bytes continuously |
| `tr -dc 'a-zA-Z0-9'` | Deletes (`-d`) all characters except (`-c` complement) alphanumerics |
| `fold -w 10` | Wraps the output into 10-character lines |
| `head -n 2` | Takes only the first two lines — one for the username, one for the password |

Example output:

```
OhxVoPWnKo
N0Szm58iVP
```

These are noted immediately — the first becomes the DB username and the second the DB password. Record them in a local text file for reference during the `wp-config.php` configuration step.

### Step 3 — Create the Production Database and User

```bash
sudo mysql
```

```sql
create database expertwp_help;
grant all privileges on expertwp_help.* to 'OhxVoPWnKo'@'localhost' identified by 'N0Szm58iVP';
flush privileges;
exit
```

> **Note:** The database name uses an underscore (`expertwp_help`), not a dot. The user is scoped to `localhost` only — WordPress connects to the database from the same server over a local socket, not over the network.

### Step 4 — Generate WordPress Admin Credentials

A second `urandom` run generates the WordPress admin username and password. A 30-character string is used here for stronger security on the admin account:

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 30 | head -n 2
```

Example output:

```
E5xoPx1zgx6cdg1kF5JErvFsJlhpHs
ZeBNjMBtWCmRqZhaTgOVkdMl4ptnqb
```

> **Why random admin credentials?** Using random, high-entropy usernames for the WordPress admin account makes credential-stuffing and brute-force attacks significantly harder. The default `admin` username is targeted by automated bots constantly.

### Step 5 — Download and Extract WordPress

```bash
cd
wget https://wordpress.org/latest.tar.gz
ls -l
tar xf latest.tar.gz
ls -l
```

WordPress maintains a permanent URL for the latest version, so the `wget` command never needs to be updated. After extraction, a `wordpress/` directory appears in the home directory.

Enter the extracted directory and inspect its contents:

```bash
cd wordpress/
ls
```

The sample config file `wp-config-sample.php` needs to be renamed before WordPress can be deployed:

```bash
mv wp-config-sample.php wp-config.php
```

### Step 6 — Configure `wp-config.php`

```bash
sudo nano wp-config.php
```

#### Database Credentials

Locate and fill in the three database constants:

```php
define( 'DB_NAME', 'expertwp_help' );
define( 'DB_USER', 'OhxVoPWnKo' );
define( 'DB_PASSWORD', 'N0Szm58iVP' );
define( 'DB_HOST', 'localhost' );
```

#### Security Keys and Salts

WordPress uses a set of secret keys and salts to secure cookies and stored passwords. Query the WordPress API for a fresh set:

```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

Copy the entire output (8 PHP `define()` statements) and paste it into `wp-config.php`, replacing the existing placeholder lines for the keys section.

#### Additional Security and Behaviour Constants

Add the following constants after the database constants but before the `require_once` at the bottom:

```php
/** Allow Direct Updating Without FTP */
define('FS_METHOD', 'direct');

/** Disable Editing of Themes and Plugins Using the Built In Editor */
define('DISALLOW_FILE_EDIT', 'true');

/** Turn Off Automatic Core Updates */
define('WP_AUTO_UPDATE_CORE', false);
define('AUTOMATIC_UPDATER_DISABLED', 'true');
```

| Constant | Value | Purpose |
|----------|-------|---------|
| `FS_METHOD` | `'direct'` | Tells WordPress to write files directly to the filesystem instead of prompting for FTP credentials. Correct when the web server user owns the files. |
| `DISALLOW_FILE_EDIT` | `'true'` | Removes the built-in Theme and Plugin editor from the WordPress admin — if the admin panel is ever compromised, an attacker cannot use it to inject malicious PHP. |
| `WP_AUTO_UPDATE_CORE` | `false` | Disables automatic WordPress core updates. Updates are handled manually to prevent unexpected breaking changes. |
| `AUTOMATIC_UPDATER_DISABLED` | `'true'` | A broader flag that disables the automatic updater subsystem entirely, covering plugins and themes as well as core. |

Save and close the file.

### Step 7 — Deploy WordPress Files to the Document Root

`rsync` is used rather than `cp` because it preserves permissions, handles symlinks correctly, and gives verbose output:

```bash
cd
sudo rsync -artv wordpress/ /var/www/expertwp.help/public_html/
```

**`rsync` flags used:**

| Flag | Meaning |
|------|---------|
| `-a` | Archive mode — preserves permissions, timestamps, symlinks, and recursively copies directories |
| `-r` | Recursive (implied by `-a`, but explicit here for clarity) |
| `-t` | Preserve modification timestamps |
| `-v` | Verbose — prints each file as it is transferred |

> **Important:** The trailing slash on `wordpress/` is significant. With `rsync`, `wordpress/` means "copy the *contents* of the wordpress directory", not the directory itself. Without the trailing slash, `rsync` would create `/var/www/expertwp.help/public_html/wordpress/`, which is wrong.

After transfer, clean up the home directory:

```bash
sudo rm -rf latest.tar.gz wordpress/
ls
```

### Step 8 — Set File Ownership

After the files are copied, ownership is transferred to `www-data`, the user that Nginx and php-fpm run as:

```bash
cd /var/www/expertwp.help/
sudo chown -R www-data:www-data public_html/
ls -l
```

All files and directories in `public_html/` should now show `www-data www-data` as owner and group.

---

## 32.4 Summary

| Component | Status |
|-----------|--------|
| DNS (A + CNAME records) | Verified — both resolving to server IP |
| Nginx server block | Active and reloaded |
| MariaDB database `expertwp_help` | Created |
| MariaDB user `OhxVoPWnKo` | Created with full privileges on `expertwp_help` |
| WordPress files | Downloaded, extracted, deployed to `public_html/` |
| `wp-config.php` | Configured with DB credentials, salts, FS method, and auto-update disabled |
| File ownership | Set to `www-data:www-data` recursively |

Navigating to `http://expertwp.help` will now launch the WordPress installation wizard.

---

## 32.5 Reference — Useful MariaDB Commands

```sql
-- List all databases
show databases;

-- List all users and their allowed hosts
select host, user from mysql.user;

-- Switch to a specific database
use db_name;

-- List tables in the active database
show tables;

-- Describe a table's column structure
describe table_name;

-- Show privileges for a user
show grants for 'db_user'@'localhost';

-- Remove a database entirely (irreversible)
drop database db_name;

-- Remove a user account
drop user 'db_user'@'localhost';
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 31 — Nginx Server Block: FastCGI & Browser Caching](./31-nginx-server-block-fastcgi-browser-caching.md) | [↑ Top](#chapter-32--mariadb-database-setup--wordpress-installation-preparation) | [Chapter 33 — WordPress Config, Deployment & Setup →](./33-wordpress-config-deployment-setup.md) |
