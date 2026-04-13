[← Back to Index](../index.md)

---

# Chapter 48 — Hardening WordPress: Restricting MariaDB Database User Privileges

*Tightening the WordPress database user from `ALL PRIVILEGES` down to the minimum four needed for day-to-day operation — `SELECT`, `INSERT`, `UPDATE`, `DELETE` — applying the principle of least privilege to the database layer.*

---

## Contents

- [48.1 Overview](#481-overview)
- [48.2 MariaDB Privilege Reference](#482-mariadb-privilege-reference)
- [48.3 Implementation](#483-implementation)
  - [Step 1 — Identify the Database Credentials](#step-1--identify-the-database-credentials)
  - [Step 2 — Log Into MariaDB as Root](#step-2--log-into-mariadb-as-root)
  - [Step 3 — Revoke All Existing Privileges](#step-3--revoke-all-existing-privileges)
  - [Step 4 — Grant Only the Required Privileges](#step-4--grant-only-the-required-privileges)
  - [Step 5 — Verify the Grants](#step-5--verify-the-grants)
  - [Step 6 — Flush Privileges and Exit](#step-6--flush-privileges-and-exit)
- [48.4 Handling Plugins That Require Additional Privileges](#484-handling-plugins-that-require-additional-privileges)
- [48.5 Summary of Commands](#485-summary-of-commands)

---

## 48.1 Overview

During the initial WordPress installation, the site database user was granted `ALL PRIVILEGES` on the site database. This was the safe default at the time — plugins have varying database requirements, and granting full privileges avoids unexpected breakage during setup. Now that the site is configured and running, this is tightened down to the minimum set of privileges required for day-to-day WordPress operation.

The principle here is the same as filesystem permissions: the database user should only be able to do what WordPress actually needs it to do. Anything beyond that is unnecessary attack surface.

> **Warning:** Back up the site and database before making any changes to database privileges. If a plugin requires a privilege that is revoked, the site will break until that privilege is restored.

> **Warning:** Do not apply privilege restrictions if the site runs WooCommerce. WooCommerce performs complex database operations — including creating and modifying tables during plugin updates and order processing — that require privileges beyond the standard four. Applying this hardening to a WooCommerce site will break site functionality.

---

## 48.2 MariaDB Privilege Reference

| Privilege | Description |
|-----------|-------------|
| `SELECT` | Read rows from tables |
| `INSERT` | Add new rows to tables |
| `UPDATE` | Modify existing rows |
| `DELETE` | Remove rows from tables |
| `CREATE` | Create new databases and tables |
| `DROP` | Delete databases and tables |
| `RELOAD` | Execute `FLUSH` and reload server settings |
| `SHUTDOWN` | Shut down the MySQL/MariaDB server |
| `PROCESS` | View and kill user threads |
| `FILE` | Read and write files on the server |
| `GRANT` | Grant or revoke privileges for other users |
| `REFERENCES` | Create foreign key constraints |
| `INDEX` | Create or remove indexes on tables |
| `ALTER` | Modify the structure of tables |
| `SHOW DATABASES` | List all databases on the server |
| `SUPER` | Perform administrative tasks such as setting global server variables |
| `CREATE TEMPORARY TABLES` | Create temporary tables |
| `LOCK TABLES` | Lock tables for explicit use in transactions |
| `EXECUTE` | Execute stored procedures and functions |
| `CREATE VIEW` | Create views |
| `SHOW VIEW` | View the `CREATE VIEW` statement for views |
| `CREATE ROUTINE` | Create stored procedures and functions |
| `ALTER ROUTINE` | Alter stored procedures and functions |
| `CREATE USER` | Create new MariaDB user accounts |
| `EVENT` | Create, alter, drop, and execute events |
| `TRIGGER` | Create and drop triggers |

For a standard WordPress site (no WooCommerce), only `SELECT`, `INSERT`, `UPDATE`, and `DELETE` are required for day-to-day operation.

---

## 48.3 Implementation

### Step 1 — Identify the Database Credentials

The database name and database user are defined in `wp-config.php`. Grep for the `DB_` constants to retrieve them without opening the full file:

```bash
grep DB_ /var/www/expertwp.help/public_html/wp-config.php
```

Output:

```
define( 'DB_NAME', 'site_db' );
define( 'DB_USER', 'site_user' );
define( 'DB_PASSWORD', 'site_password' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8' );
define( 'DB_COLLATE', '' );
```

Note the values of `DB_NAME` and `DB_USER` — these are the database and user that will be targeted in the privilege changes.

### Step 2 — Log Into MariaDB as Root

```bash
sudo mysql
```

This opens the MariaDB monitor as the `root` user. The prompt changes to `MariaDB [(none)]>`.

### Step 3 — Revoke All Existing Privileges

Strip the database user of all currently granted privileges on the site database:

```sql
REVOKE ALL PRIVILEGES ON site_db.* FROM 'site_user'@'localhost';
```

Replace `site_db` with the value of `DB_NAME` and `site_user` with the value of `DB_USER` from `wp-config.php`. This returns `Query OK, 0 rows affected` on success.

### Step 4 — Grant Only the Required Privileges

Grant the minimum set of privileges needed for WordPress to function:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON site_db.* TO 'site_user'@'localhost';
```

This covers all standard WordPress read/write operations: reading posts and options, writing new content, updating existing records, and deleting data.

### Step 5 — Verify the Grants

```sql
SHOW GRANTS FOR 'site_user'@'localhost';
```

Expected output:

```
+-------------------------------------------------------------------------------------------+
| Grants for site_user@localhost                                                            |
+-------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'site_user'@'localhost' IDENTIFIED BY PASSWORD '***'               |
| GRANT SELECT, INSERT, UPDATE, DELETE ON `site_db`.* TO 'site_user'@'localhost'           |
+-------------------------------------------------------------------------------------------+
```

There should be no `ALL PRIVILEGES` or any other privileges listed beyond the four granted in the previous step.

### Step 6 — Flush Privileges and Exit

```sql
FLUSH PRIVILEGES;
exit
```

`FLUSH PRIVILEGES` forces MariaDB to reload the grant tables from disk, ensuring the changes take effect immediately for all new connections.

---

## 48.4 Handling Plugins That Require Additional Privileges

Some plugins — especially those that create their own database tables on activation (e.g. membership plugins, e-commerce plugins, caching layers with database backends) — need additional privileges such as `CREATE`, `ALTER`, or `INDEX`. These are typically only needed during plugin activation or updates, not during normal operation.

If a plugin fails to activate or behaves unexpectedly after applying privilege restrictions, grant the additional privileges it requires:

```sql
GRANT CREATE, ALTER, INDEX ON site_db.* TO 'site_user'@'localhost';
FLUSH PRIVILEGES;
```

Then re-test on a development server before applying to production. Once the plugin is activated and its tables are in place, these additional privileges can optionally be revoked again if the plugin only requires them at activation time.

---

## 48.5 Summary of Commands

```bash
# Check database credentials
grep DB_ /var/www/example.com/public_html/wp-config.php

# Log into MariaDB
sudo mysql
```

```sql
-- Strip all existing privileges
REVOKE ALL PRIVILEGES ON site_db.* FROM 'site_user'@'localhost';

-- Grant only what WordPress needs
GRANT SELECT, INSERT, UPDATE, DELETE ON site_db.* TO 'site_user'@'localhost';

-- Verify the result
SHOW GRANTS FOR 'site_user'@'localhost';

-- Apply and exit
FLUSH PRIVILEGES;
exit
```

```sql
-- If a plugin needs schema-level access (e.g., during activation)
GRANT CREATE, ALTER, INDEX ON site_db.* TO 'site_user'@'localhost';
FLUSH PRIVILEGES;
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 47 — Disabling File Modifications](./47-disallow-file-mods.md) | [↑ Top](#chapter-48--hardening-wordpress-restricting-mariadb-database-user-privileges) | [Chapter 49 — REST API Hardening →](./49-rest-api-hardening.md) |
