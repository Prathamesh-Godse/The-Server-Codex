[← Back to Index](../index.md)

---

# Chapter 24 — Harden MariaDB

*Running `mysql_secure_installation` to remove insecure defaults from a fresh MariaDB install, and understanding how Unix socket authentication works and why `sudo mysql` is the correct way to connect as root.*

---

## Contents

- [24.1 Overview](#241-overview)
- [24.2 Running mysql_secure_installation](#242-running-mysql_secure_installation)
  - [Current Root Password](#current-root-password)
  - [Switch to Unix Socket Authentication](#switch-to-unix-socket-authentication)
  - [Change the Root Password](#change-the-root-password)
  - [Remove Anonymous Users](#remove-anonymous-users)
  - [Disallow Root Login Remotely](#disallow-root-login-remotely)
  - [Remove the Test Database](#remove-the-test-database)
  - [Reload Privilege Tables](#reload-privilege-tables)
- [24.3 Unix Socket Authentication and Logging Into MariaDB](#243-unix-socket-authentication-and-logging-into-mariadb)
  - [Why sudo is Required](#why-sudo-is-required)
- [24.4 Summary](#244-summary)

---

## 24.1 Overview

A fresh MariaDB installation ships with several insecure defaults — anonymous user accounts, a publicly accessible `test` database, and the ability to log in as root from remote hosts. Before putting the database into production use, these need to be removed. MariaDB ships with a built-in security script, `mysql_secure_installation`, that handles all of this interactively.

---

## 24.2 Running `mysql_secure_installation`

Run the hardening script with sudo:

```bash
sudo mysql_secure_installation
```

The script walks through a series of prompts. Below is a breakdown of each prompt and the response used.

---

### Current Root Password

```
Enter current password for root (enter for none):
```

Press **Enter** — no password has been set yet on a fresh install. The script confirms:

```
OK, successfully used password, moving on...
```

---

### Switch to Unix Socket Authentication

```
Switch to unix_socket authentication [Y/n] n
```

Answer **n** — the root account is already protected via Unix socket authentication (explained in section 24.3), so switching again is unnecessary.

---

### Change the Root Password

```
Change the root password? [Y/n] n
```

Answer **n** — root login is protected by Unix socket authentication, so a password here would not add meaningful security and is not needed.

---

### Remove Anonymous Users

```
Remove anonymous users? [Y/n] y
... Success!
```

Answer **y**. Anonymous users allow anyone to connect to MariaDB without credentials. These exist only to make a fresh install easier to test and must be removed before production use.

---

### Disallow Root Login Remotely

```
Disallow root login remotely? [Y/n] y
... Success!
```

Answer **y**. Root should only ever connect from `localhost`. Disabling remote root login prevents an attacker from attempting to brute-force the root account over the network.

---

### Remove the Test Database

```
Remove test database and access to it? [Y/n] y
 - Dropping test database...
... Success!
 - Removing privileges on test database...
... Success!
```

Answer **y**. The `test` database is accessible to all users including anonymous ones by default. It serves no purpose in production.

---

### Reload Privilege Tables

```
Reload privilege tables now? [Y/n] y
... Success!
```

Answer **y**. This ensures all the changes made by the script take effect immediately without needing to restart MariaDB.

The script finishes with:

```
All done! If you've completed all the above steps, your MariaDB
installation should now be secure.
```

---

## 24.3 Unix Socket Authentication and Logging Into MariaDB

After hardening, attempting to log in without sudo fails:

```bash
mysql
# ERROR 1698 (28000): Access denied for user 'andrew'@'localhost'

mysql -u root
# ERROR 1698 (28000): Access denied for user 'root'@'localhost'

mysql -u root -p
# ERROR 1698 (28000): Access denied for user 'root'@'localhost'
```

All three fail. The correct way to connect as root is:

```bash
sudo mysql
```

This succeeds immediately and drops into the MariaDB monitor:

```
Welcome to the MariaDB monitor. Commands end with ; or \g.
Your MariaDB connection id is 48
Server version: 10.11.8-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04
```

### Why `sudo` is Required

MariaDB on Ubuntu uses **Unix socket authentication** for the root account by default. Instead of checking a password, it verifies the OS-level user making the connection. Because the MariaDB root account is mapped to the system's `root` user, only a process running as root (via `sudo`) can authenticate. This is actually more secure than password authentication — there is no password to brute-force or leak.

This is why:
- `mysql -u root -p` fails even with the correct password — the auth plugin ignores the password entirely and checks the OS user identity instead.
- `sudo mysql` works without any password prompt — `sudo` elevates the process to run as root, which satisfies the Unix socket auth check.

To exit the MariaDB shell:

```sql
exit
```

---

## 24.4 Summary

| Prompt | Response | Reason |
|--------|----------|--------|
| Switch to Unix socket authentication | `n` | Already active |
| Change the root password | `n` | Unix socket auth makes it unnecessary |
| Remove anonymous users | `y` | Required for production |
| Disallow root login remotely | `y` | Root should only connect locally |
| Remove test database | `y` | No use in production |
| Reload privilege tables | `y` | Apply changes immediately |

| Task | Command |
|------|---------|
| Run the security hardening script | `sudo mysql_secure_installation` |
| Log into MariaDB as root | `sudo mysql` |
| Exit MariaDB shell | `exit` |

---

| | | |
|:---|:---:|---:|
| [← Chapter 23 — Nginx Open File Limits & Bash Aliases](./23-nginx-open-file-limits-and-bash-aliases.md) | [↑ Top](#chapter-24--harden-mariadb) | [Chapter 25 — Optimise MariaDB →](./25-optimize-mariadb.md) |
