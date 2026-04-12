[← Back to Index](../index.md)

---

# Chapter 4 — Users, Groups, Ownership & Permissions

*The Linux access control model. Incorrect permissions are one of the most common causes of both security vulnerabilities and broken WordPress sites.*

---

## Contents

- [4.1 User Types](#41-user-types)
- [4.2 sudo](#42-sudo)
- [4.3 Groups](#43-groups)
- [4.4 Ownership](#44-ownership)
- [4.5 Permissions](#45-permissions)
- [4.6 Reading Permissions from ls -l](#46-reading-permissions-from-ls--l)
- [4.7 Numeric (Octal) Permission Values](#47-numeric-octal-permission-values)
- [4.8 Common Permission Values](#48-common-permission-values)
- [4.9 Changing Permissions — chmod](#49-changing-permissions--chmod)
- [4.10 Changing Ownership — chown](#410-changing-ownership--chown)
- [4.11 The 777 Warning](#411-the-777-warning)

---

## 4.1 User Types

Linux has three categories of users:

| Type | Description |
|------|-------------|
| **Root user** | Superuser with unrestricted access to the entire system. UID 0. Can read, write, and execute any file regardless of permissions. |
| **System users** | Created automatically by services during installation (e.g. `www-data` for Nginx, `mysql` for MariaDB). These are not login accounts — they exist so services can run with limited, isolated privileges. |
| **Normal users** | Human accounts used for day-to-day administration. On most VPS providers the default user is called `ubuntu`. These accounts have limited privileges and use `sudo` to perform administrative tasks. |

Throughout this project, all work is done as a normal user with `sudo` — never as root directly.

---

## 4.2 `sudo`

`sudo` (superuser do) allows a normal user to execute a single command with root-level privileges without switching to the root account. It is prepended to any command that requires elevated access.

```bash
# Run a privileged command as a normal user
sudo apt update

# Edit a system-owned file
sudo nano /etc/nginx/nginx.conf

# Restart a system service
sudo systemctl restart nginx
```

When `sudo` is used, the system logs the command and who ran it — making it more auditable and safer than running everything as root.

---

## 4.3 Groups

Groups are collections of users that share a set of permissions. Every user belongs to at least one group (their primary group, which typically has the same name as the username). A user can also belong to additional supplementary groups.

Groups are used throughout this project to control which processes and users can read or write to specific directories — particularly the web root at `/var/www/`.

Key groups in this project:

| Group | Purpose |
|-------|---------|
| `www-data` | The group under which Nginx runs. WordPress files are assigned to this group so Nginx can serve them. |
| `sudo` | Members of this group can use `sudo`. The default `ubuntu` user is in this group. |

---

## 4.4 Ownership

Every file and directory on Linux has two owners assigned to it:

- **Owner (user)** — the individual user who created or owns the file
- **Group owner** — the group that has group-level access to the file
- **Other** — any user on the system that is neither the owner nor in the group

Ownership determines which set of permissions applies when a user tries to access a file. If you are the file owner, the owner permissions apply. If you are in the group, group permissions apply. Otherwise, the "other" permissions apply.

---

## 4.5 Permissions

Every file and directory has three types of permissions, each applying independently to the owner, group, and other:

| Permission | Letter | Number | On a File | On a Directory |
|------------|--------|--------|-----------|----------------|
| Read | `r` | `4` | View file contents | List directory contents (`ls`) |
| Write | `w` | `2` | Modify or delete the file | Create, delete, rename files inside the directory |
| Execute | `x` | `1` | Run as a script or program | Enter the directory (`cd`) and traverse into it |
| None | `-` | `0` | No access | No access |

The execute permission on a directory is often overlooked but is essential — without it you cannot `cd` into a directory even if you can list it.

---

## 4.6 Reading Permissions from `ls -l`

The `ls -l` output from `/etc/nginx` shows the permission string for each file and directory:

```
andrew@hostname:/etc/nginx$ ls -l

d rwx r-x r-x  2  root  root  4096  Nov  9  2018  conf.d
- rw-  r-- r--  1  root  root  5759  Mar 14 20:35  nginx.conf
```

Breaking down the first entry `d rwxr-xr-x`:

```
d  rwx  r-x  r-x
│   │    │    │
│   │    │    └── Other:      read + execute  (r-x = 5)
│   │    └─────── Group:      read + execute  (r-x = 5)
│   └──────────── Owner:      read + write + execute  (rwx = 7)
└──────────────── d = directory
```

The full `ls -l` column breakdown:

```
[type+perms]  [links]  [owner]  [group]  [size]  [date]  [name]
drwxr-xr-x      2       root     root     4096   Nov 9    conf.d
-rw-r--r--      1       root     root     5759   Mar 14   nginx.conf
```

The permission string is always 10 characters: 1 type character followed by 3 sets of `rwx` (owner, group, other).

---

## 4.7 Numeric (Octal) Permission Values

Permissions can also be expressed as a 3-digit number, which is what `chmod` uses. Each digit represents one of the three ownership levels (owner, group, other), and is the sum of the permission values that apply:

| Permission | Value |
|------------|-------|
| read (r) | 4 |
| write (w) | 2 |
| execute (x) | 1 |
| none (-) | 0 |

**Example: `rw-rw-r--` → 664**

```
rw-   rw-   r--
 │     │     │
 │     │     └── other:  r-- = 4+0+0 = 4
 │     └───────── group:  rw- = 4+2+0 = 6
 └─────────────── owner:  rw- = 4+2+0 = 6

Result: 664
```

**Example: `rwxrwxr-x` → 775**

```
rwx   rwx   r-x
 │     │     │
 │     │     └── other:  r-x = 4+0+1 = 5
 │     └───────── group:  rwx = 4+2+1 = 7
 └─────────────── owner:  rwx = 4+2+1 = 7

Result: 775
```

---

## 4.8 Common Permission Values

| Value | String | Typical use |
|-------|--------|-------------|
| `644` | `rw-r--r--` | Standard files — owner can edit, everyone else can read. Used for config files and WordPress PHP files. |
| `664` | `rw-rw-r--` | Files where both owner and group need write access. |
| `755` | `rwxr-xr-x` | Standard directories — owner has full access, others can read and traverse. Used for web root directories. |
| `775` | `rwxrwxr-x` | Directories where both owner and group need write access. |
| `400` | `r--------` | Read-only for owner only. Used for private keys. |
| `600` | `rw-------` | Read/write for owner only. Used for sensitive config files. |

---

## 4.9 Changing Permissions — `chmod`

`chmod` changes the permission bits on a file or directory. It accepts both numeric and symbolic notation.

```bash
# Set permissions using numeric notation
chmod 644 /var/www/example.com/public_html/wp-config.php
chmod 755 /var/www/example.com/public_html/

# Apply recursively to all files and directories inside a path
chmod -R 755 /var/www/example.com/public_html/

# Symbolic notation — add execute for owner
chmod u+x script.sh

# Symbolic notation — remove write from group and other
chmod go-w /etc/nginx/nginx.conf
```

In this project, numeric notation is used exclusively since it is unambiguous and easier to script.

---

## 4.10 Changing Ownership — `chown`

`chown` changes the user owner and/or group owner of a file or directory.

```bash
# Change owner only
chown ubuntu /var/www/example.com/public_html/

# Change owner and group
chown ubuntu:www-data /var/www/example.com/public_html/

# Change group only (note the leading colon)
chown :www-data /var/www/example.com/public_html/

# Apply recursively
chown -R ubuntu:www-data /var/www/example.com/public_html/
```

Throughout this project, WordPress site directories are typically owned by the normal user (`ubuntu`) with the group set to `www-data` so Nginx can read the files without having write access.

---

## 4.11 The 777 Warning

Never set any file or directory to `777` permissions on a production server.

`777` grants read, write, and execute to **every user on the system**, including any compromised process or script. This completely bypasses all access control and means any malicious code running on the server — even under a different user account — can modify, delete, or execute anything with those permissions.

The correct approach is to set the minimum permissions required for the service to function:

- WordPress files: `644` (owner read/write, everyone else read-only)
- WordPress directories: `755` (owner full, everyone else read + traverse)
- `wp-config.php`: `600` or `640` (restrict to owner or owner+group only)

---

| | | |
|:---|:---:|---:|
| [← Chapter 3 — Linux File System](./03-linux-file-system.md) | [↑ Top](#chapter-4--users-groups-ownership--permissions) | [Chapter 5 — Essential Linux Skills →](./05-essential-linux-skills.md) |
