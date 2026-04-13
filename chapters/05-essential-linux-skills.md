[← Back to Index](../index.md)

---

# Chapter 5 — Essential Linux Skills for Server Administration

*The operational toolkit — the specific skills used repeatedly throughout the server setup, from editing config files to scheduling automated tasks.*

---

## Contents

- [5.1 The nano Text Editor](#51-the-nano-text-editor)
- [5.2 Configuration Files](#52-configuration-files)
- [5.3 Packages, Repositories, and the Package Manager](#53-packages-repositories-and-the-package-manager)
- [5.4 The Init System: systemd and systemctl](#54-the-init-system-systemd-and-systemctl)
- [5.5 SSH and Server Fingerprints](#55-ssh-and-server-fingerprints)
- [5.6 SSH Key Authentication](#56-ssh-key-authentication)
- [5.7 Bash Scripts](#57-bash-scripts)
- [5.8 Cron Jobs](#58-cron-jobs)
- [5.9 Summary](#59-summary)

---

## 5.1 The `nano` Text Editor

`nano` is the terminal-based text editor used throughout this project to edit configuration files directly on the server. It is minimal, straightforward, and available on virtually every Linux system by default.

### Opening nano

To open nano with a blank buffer:

```bash
nano
```

To open an existing file for editing directly:

```bash
nano /path/to/file/filename
```

Alternatively, navigate to the directory first and then open by filename:

```bash
cd /path/to/file/
nano filename
```

If the file does not exist, nano creates it on save.

### Essential nano Keyboard Shortcuts

| Command        | Shortcut        | Description                                                                 |
|----------------|-----------------|-----------------------------------------------------------------------------|
| Exit           | `Ctrl + X`      | Close nano. You will be prompted to save any unsaved changes before exiting. |
| Save           | `Ctrl + O`      | Write (save) the file without closing nano.                                  |
| Search         | `Ctrl + W`      | Search for a string in the file.                                             |
| Find Next      | `Alt + W`       | Jump to the next occurrence of the search term.                              |
| Search/Replace | `Ctrl + \`      | Search for a string and replace it with another.                             |
| Undo           | `Alt + U`       | Undo the last change.                                                        |
| Cancel         | `Ctrl + C`      | Cancel the current operation or key selection.                               |

> **Tip:** The shortcuts are displayed at the bottom of the nano window at all times. `^` represents `Ctrl` and `M-` represents `Alt`.

---

## 5.2 Configuration Files

Configuration files (commonly called "conf files") are plain text files that contain **directives** — settings that control the behaviour of a specific service or application. Every major service on the server (Nginx, PHP, MariaDB) has at least one configuration file.

### How Configuration File Comments Work

Lines beginning with `#` or `;` are comments and are ignored by the service. They are used for documentation and to temporarily disable a directive without deleting it.

```ini
# InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.
default_storage_engine = InnoDB

# Commenting out a directive disables it without removing it:
#innodb_log_file_size = 50M

innodb_buffer_pool_size = 256M
innodb_log_buffer_size  = 8M
```

```ini
; How often (in seconds) to check file timestamps for changes to the shared
; memory storage allocation.
opcache.revalidate_freq=0
```

### Best Practices When Editing Configuration Files

Always make a backup before editing. A broken config can take a service completely offline:

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

Reload or restart the service after changes for them to take effect:

```bash
sudo systemctl reload nginx
# or
sudo systemctl restart nginx
```

> `reload` applies changes without dropping active connections. `restart` fully stops and starts the service — use this when reload is not sufficient.

---

## 5.3 Packages, Repositories, and the Package Manager

### What is a Package?

A **package** is a software bundle that contains everything needed to install and run a specific application or component — executables, libraries, default configuration files, and other resources. Installing a package via the package manager automatically handles all of this.

### What is a Repository?

A **repository** (repo) is a centralised, curated location that stores software packages and their metadata for a specific Linux distribution. Ubuntu ships configured with its official repos. There are also third-party repos that can be added for software not in the official set.

### The `apt` Package Manager

Ubuntu's built-in package manager is `apt` (Advanced Package Tool). It handles install, update, upgrade, and removal of packages, along with their dependencies, automatically.

```bash
# Sync local package index with remote repositories — always run before installing
sudo apt update

# Upgrade all installed packages to their newest available versions
sudo apt upgrade

# Remove orphaned dependencies left after package removal
sudo apt autoremove

# Download and install a package with all required dependencies
sudo apt install package_name

# Uninstall a package — leaves configuration files behind
sudo apt remove package_name

# Uninstall a package and remove its configuration files
sudo apt purge package_name

# Search the package index by name or keyword
sudo apt-cache search package_name
```

> **How apt works internally:** When you run `apt install`, it queries the local package index, identifies all required dependencies, downloads the package and its dependencies from the configured repo, and installs everything in the correct order.

---

## 5.4 The Init System: `systemd` and `systemctl`

### The Init System

The **init system** is the first process that starts when the server boots (PID 1). It manages the initialisation and startup of the entire system. Ubuntu uses **systemd** as its init system.

### systemd

`systemd` is the system and service manager responsible for managing all services (daemons) running on the server — including Nginx, PHP-FPM, and MariaDB in this project.

### systemctl

`systemctl` is the command-line utility used to interact with systemd.

```bash
# Check the current status of a service
sudo systemctl status service_name

# Start a service
sudo systemctl start service_name

# Stop a service
sudo systemctl stop service_name

# Restart a service (full stop + start)
sudo systemctl restart service_name

# Reload a service's configuration without a full restart
sudo systemctl reload service_name

# Enable a service to start automatically at boot
sudo systemctl enable service_name

# Disable a service from starting at boot
sudo systemctl disable service_name
```

> Throughout this project, these commands are used frequently with `nginx`, `php8.x-fpm`, and `mariadb`.

---

## 5.5 SSH and Server Fingerprints

### What is a Server Fingerprint?

A **server fingerprint** is a cryptographic hash of the server's public host key. It is used to verify the identity of a server — to confirm you are connecting to the intended server, and not an impersonator.

### First Connection Behaviour

The first time you SSH into any server, you are prompted to accept or deny the connection:

```
The authenticity of host '203.0.113.10 (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:abc123...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Once accepted, the fingerprint is stored in `~/.ssh/known_hosts`. On all subsequent connections, SSH compares the server's current fingerprint against the stored one. A mismatch raises a warning — this could indicate a server OS reinstallation, a man-in-the-middle (MITM) attack, or IP spoofing.

### Clearing a Stale Fingerprint

If you reinstall the server OS and need to reconnect, remove the old fingerprint:

```bash
nano ~/.ssh/known_hosts
```

Find the line for the server's IP or hostname and delete it, then save. The next connection will prompt for the new fingerprint.

---

## 5.6 SSH Key Authentication

### Why Use SSH Key Authentication?

SSH key authentication replaces username/password login with a cryptographic key pair. It provides:

- **Enhanced security** — no password that can be guessed or brute-forced
- **No password entry required** — login is automatic once the key pair is set up
- **Key-based identity verification** — only the holder of the private key can authenticate

> For the initial setup of the server, username and password login is used. SSH key authentication is configured afterwards.

### How SSH Key Authentication Works

A key pair is generated **locally on your PC or Mac** — never on the server.

- The **private key** stays on your local machine (never shared)
- The **public key** is uploaded to the server

**Authentication flow:**

1. Client initiates an SSH connection to the server
2. The server sends a random challenge message back to the client
3. The client signs the challenge using its **private key** and sends the signature back
4. The server verifies the signature using the stored **public key** — if it matches, access is granted

### Generating an SSH Key Pair

Run this on your **local machine** (PC or Mac), not on the server:

```bash
ssh-keygen -t ed25519 -C "your_comment_or_email"
```

This generates two files in `~/.ssh/`:
- `id_ed25519` — private key (never share this)
- `id_ed25519.pub` — public key (uploaded to the server)

### Copying the Public Key to the Server

```bash
ssh-copy-id username@server_ip
```

This appends your public key to `~/.ssh/authorized_keys` on the server.

---

## 5.7 Bash Scripts

A **bash script** is a plain text file containing a series of shell commands that are executed in sequence. Instead of typing the same commands manually each time, they are written once and run as a single unit — the foundation for automating the entire server setup.

### Structure of a Bash Script

Every bash script begins with a **shebang line** that tells the system which interpreter to use:

```bash
#!/bin/bash
```

Example script:

```bash
#!/bin/bash

# SET SELINUX CONTEXT RECURSIVELY
sudo chcon -R -t httpd_sys_rw_content_t /var/www/mysite.com/public_html/
```

### Making a Script Executable

```bash
chmod +x script_name.sh
```

### Running a Script

```bash
./script_name.sh
# or with an absolute path:
/home/user/scripts/script_name.sh
```

> All server automation in this project — package installation, Nginx configuration, WordPress setup — is structured so it can be extracted into a bash script for repeatable deployments.

---

## 5.8 Cron Jobs

**Cron** is the built-in task scheduler on Linux. It allows commands or scripts to be run automatically at specified times — without any manual intervention. In this project, cron is used for SSL certificate renewal and site backups.

### Editing the Crontab

```bash
crontab -l      # list existing cron jobs
crontab -e      # open the crontab editor

sudo crontab -l # list root's cron jobs
sudo crontab -e # edit root's crontab
```

### Cron Job Syntax

Each cron job is a single line:

```
[schedule]   [command]   [redirect]
```

Full example:

```
0 5 * * 1   tar -zcf /var/backups/home.tgz /home/   >/dev/null 2>&1
```

### The Schedule Field

```
* * * * *
│ │ │ │ └── Day of week  (0–6, where 0 = Sunday)
│ │ │ └──── Month        (1–12)
│ │ └────── Day of month (1–31)
│ └──────── Hour         (0–23)
└────────── Minute       (0–59)
```

Special syntax:

| Syntax | Meaning |
|--------|---------|
| `*` | Every unit |
| `*/15` | Every 15 units |
| `6,12` | At specific values |

### Schedule Examples

| Expression | Meaning |
|------------|---------|
| `00 2 * * *` | Every day at 2:00 AM |
| `30 9 * * 6` | Every Saturday at 9:30 AM |
| `00 9,15,21 7 * *` | At 9am, 3pm, 9pm on the 7th of every month |
| `15 1 5,25 * *` | At 1:15 AM on the 5th and 25th |
| `*/15 * * * *` | Every 15 minutes |

### The Redirect Field

By default, cron emails every job's output to the owning user. Redirecting to `/dev/null` silences it:

```
>/dev/null 2>&1
```

- `>/dev/null` — discards stdout
- `2>&1` — redirects stderr to the same destination

### Full Cron Job Example

```
0 5 * * 1   tar -zcf /var/backups/home.tgz /home/   >/dev/null 2>&1
```

Runs a backup every Monday at 5:00 AM, suppresses all output.

---

## 5.9 Summary

| Skill | Purpose in This Project |
|-------|------------------------|
| `nano` | Editing all server configuration files |
| Configuration files | Tuning Nginx, PHP, MariaDB behaviour |
| `apt` | Installing Nginx, MariaDB, PHP, WordPress CLI, Certbot |
| `systemctl` | Starting, stopping, enabling Nginx, PHP-FPM, MariaDB |
| SSH fingerprints | Secure and verified server access |
| SSH key auth | Passwordless, hardened server login |
| Bash scripts | Automating the full server setup as a repeatable script |
| Cron jobs | Scheduling SSL renewal and site backups automatically |

---

| | | |
|:---|:---:|---:|
| [← Chapter 4 — Users, Groups & Permissions](./04-users-groups-permissions.md) | [↑ Top](#chapter-5--essential-linux-skills-for-server-administration) | [↑ Back to Index](../index.md) |
