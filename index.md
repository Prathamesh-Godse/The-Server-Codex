# Table of Contents

> **Linux & WordPress Server Setup Guide** — Full chapter index.  
> Start at the [README](./README.md) for an overview of this guide.

---

## Part I — Linux Fundamentals

Core Linux knowledge required before touching any server configuration.

### [Chapter 1 — Project Overview](./chapters/01-nginx-wordpress-server.md)
Stack, goals, infrastructure, and configuration philosophy.

- Stack: Nginx, MariaDB, PHP-FPM, WordPress, Certbot, Cloudflare
- Infrastructure: VPS requirements, domain registration, DNS via Cloudflare
- Configuration philosophy: write once, complete, from scratch

---

### [Chapter 2 — Linux Essentials](./chapters/02-linux-essentials.md)
Core Linux concepts needed to work on a remote server.

- Linux distributions and Ubuntu's versioning model
- The shell, Bash, SSH, terminal emulators by platform
- Reading the shell prompt; `ls` flags, long listing, permission string anatomy

---

### [Chapter 3 — Linux File System](./chapters/03-linux-file-system.md)
The Linux directory tree and how to navigate it.

- File and directory naming rules
- The `/` tree: `/etc`, `/var`, `/home`, `/root` and key subdirectories
- `pwd` and `cd`; absolute vs relative paths

---

### [Chapter 4 — Users, Groups, Ownership & Permissions](./chapters/04-users-groups-permissions.md)
The Linux access control model.

- Root, system, and normal user types; `sudo`
- Groups, ownership, permission types, reading `ls -l`, octal notation
- `chmod`, `chown`, the `777` warning

---

### [Chapter 5 — Essential Linux Skills](./chapters/05-essential-linux-skills.md)
The operational toolkit used throughout the server setup.

- `nano`, `apt`, `systemctl`, SSH keys, bash scripts, cron jobs

---

## Part II — Server Build & Hardening

Provisioning decisions, first login, and the complete hardening process.

### [Chapter 6 — Local Machine Setup & Server OS Choice](./chapters/06-local-setup-and-server-os.md)
What you need on your local machine and why Ubuntu 24.04 LTS is the correct server OS.

---

### [Chapter 7 — Server Provisioning](./chapters/07-server-provisioning.md)
Specifications, hosting provider selection, and creating a server instance.

- Dev vs production specs; Ubuntu Pro; the never-upgrade-in-place rule; Vultr setup

---

### [Chapter 8 — First Login via SSH & Hardening Introduction](./chapters/08-first-login-and-hardening-intro.md)
Connecting as root for the first time, fingerprints, and what hardening involves.

---

### [Chapter 9 — Server Hardening](./chapters/09-server-hardening.md)
Full first-login hardening — root password, non-root user, sudo, disabling root SSH login.

---

### [Chapter 10 — Hardening Continued: sudo, Updates & SSH Key Authentication](./chapters/10-hardening-sudo-updates-ssh-keys.md)
sudo usage, kernel updates, SSH key generation, disabling password authentication.

---

## Part III — Security Layers & OS Optimisation

Firewalling, intrusion prevention, and kernel-level hardening.

### [Chapter 11 — SSH Config File & Server Updates](./chapters/11-ssh-config-and-server-updates.md)
SSH alias config file and the four-step post-provisioning update sequence.

---

### [Chapter 12 — Firewall: UFW & Cloud Firewall](./chapters/12-firewall-ufw-and-cloud.md)
UFW at the OS level and the Vultr cloud firewall at the network perimeter.

---

### [Chapter 13 — Fail2Ban: Installation, Configuration & Management](./chapters/13-fail2ban.md)
Fail2Ban from the official GitHub release; 7-day SSH ban policy; monitoring and unbanning.

---

### [Chapter 14 — Hardening & Optimising the Server Distribution](./chapters/14-harden-and-optimize-server-distribution.md)
Timezone, swap file creation, and `/etc/fstab` configuration.

---

### [Chapter 15 — Hardening & Optimising the Server Distribution — Part 2](./chapters/15-harden-and-optimize-server-distribution-2.md)
`sysctl` kernel tuning, shared memory hardening, IPv6 disabling, network layer directives.

---

### [Chapter 16 — Congestion Control, File Access Times & Open File Limits](./chapters/16-congestion-control-file-access-open-file-limits.md)
BBR verification, `noatime`, PAM open file limits raised to 120,000.

---

## Part IV — Web Stack Installation & Configuration

Installing and initially configuring the application layer.

### [Chapter 17 — DNS: Pointing a Domain to the Server](./chapters/17-dns-cloudflare-domain-setup.md)
Cloudflare nameserver delegation, A record, CNAME record, DNS-only proxy status.

---

### [Chapter 18 — Installing the Hosting Stack: Nginx, MariaDB & PHP 8.3](./chapters/18-installing-nginx-mariadb-php.md)
LEMP stack installation — Ondrej PPA, Brotli/cache-purge modules, PHP 8.3 with 15 extensions.

---

### [Chapter 19 — Server Mail Without SMTP Plugins](./chapters/19-server-mail-msmtp.md)
msmtp as a `sendmail` drop-in; Gmail SMTP relay; PHP `mail()` configuration and testing.

---

### [Chapter 20 — Nginx Configuration Files](./chapters/20-nginx-configuration-files.md)
Directives, contexts, location modifier priority, `try_files`, WordPress permalink mechanics.

---

## Part V — Nginx & MariaDB Hardening and Tuning

Deep configuration of Nginx and MariaDB.

### [Chapter 21 — Hardening & Optimising Nginx](./chapters/21-nginx-harden-and-optimize.md)
`main`/`events` context tuning; the `includes/` directory; all six include files populated.

---

### [Chapter 22 — Completing the Nginx Configuration: Wiring the Include Files](./chapters/22-nginx-conf-wiring-and-reload.md)
Live session: populating files, rewriting the http context, two real config errors and fixes.

---

### [Chapter 23 — Nginx Open File Limits & Bash Aliases](./chapters/23-nginx-open-file-limits-and-bash-aliases.md)
`worker_rlimit_nofile` raised to 45000; verified via `/proc/<PID>/limits`; bash aliases.

---

### [Chapter 24 — Harden MariaDB](./chapters/24-harden-mariadb.md)
`mysql_secure_installation`; all six prompts; Unix socket authentication explained.

---

### [Chapter 25 — Optimise MariaDB](./chapters/25-optimize-mariadb.md)
Performance Schema; `skip-name-resolve`; binary log retention; InnoDB buffer pool 800M / log 200M.

---

## Part VI — PHP Hardening, Tuning & Site Preparation

Hardening and optimising PHP-FPM, raising service file limits, and building the web root.

### [Chapter 26 — MySQLTuner & MariaDB: Raising the Open File Limit](./chapters/26-mysqltuner-mariadb-open-file-limits.md)
MySQLTuner periodic health check; systemd drop-in `LimitNOFILE=40000` for MariaDB.

---

### [Chapter 27 — Harden & Optimise PHP](./chapters/27-harden-and-optimize-php.md)
PHP config hierarchy; `server_override.ini`; `allow_url_fopen`, `cgi.fix_pathinfo`, `expose_php`; upload limits; `rlimit_files=32768`.

---

### [Chapter 28 — Optimise PHP: OPcache & PHP-FPM Open File Limit](./chapters/28-optimize-php-opcache-open-file-limit.md)
Completing `server_override.ini`; OPcache theory; `validate_timestamps` dev vs production; `rlimit_files` verified on master and worker processes.

---

### [Chapter 29 — Web Root File and Directory Structure](./chapters/29-web-root-directory-structure.md)
`/var/www/` layout; `mkdir -p`; `rmdir` vs `rm -rf`; `mv` for renaming; site document root created.

---

### [Chapter 30 — Nginx Server Blocks, Browser Caching & FastCGI Optimisation](./chapters/30-nginx-server-blocks-browser-caching-fastcgi.md)
Server block theory; `sites-available`/`sites-enabled` pattern; live reload-vs-restart demonstration; `browser_caching.conf`; `fastcgi_optimize.conf`; full server block created and enabled.

---

## Part VII — WordPress Installation & Hardening

Installing WordPress, wiring it to the database, and beginning the security hardening process.

### [Chapter 31 — Nginx Server Block: FastCGI Optimisation & Browser Caching](./chapters/31-nginx-server-block-fastcgi-browser-caching.md)
Complete implementation session — creating include files, building the server block, enabling with a symlink, path typo troubleshooting, 403 verification.

- `readlink -f` to confirm include file absolute paths
- Full `browser_caching.conf` and `fastcgi_optimize.conf` with directive explanations
- `ln -s` symlink creation; `curl -I` confirming the 403 from an empty document root

---

### [Chapter 32 — MariaDB Database Setup & WordPress Installation Preparation](./chapters/32-mariadb-wordpress-installation-prep.md)
MariaDB shell fundamentals and full WordPress pre-installation workflow.

- SQL syntax rules; `create database`, `grant`, `flush privileges`, `show grants`, `drop database`
- `/dev/urandom` pipeline for generating random DB and admin credentials
- `tar xf` WordPress extraction; `rsync -artv` deployment; trailing slash significance
- `wp-config.php` DB credentials, salts via API, `FS_METHOD`, `DISALLOW_FILE_EDIT`, auto-update disabled
- `chown -R www-data:www-data` file ownership transfer

---

### [Chapter 33 — WordPress: wp-config.php, Deployment & Browser Setup](./chapters/33-wordpress-config-deployment-setup.md)
Detailed `wp-config.php` editing, WordPress deployment, and the complete installation wizard walkthrough.

- `curl -s https://api.wordpress.org/secret-key/1.1/salt/` salt generation
- Randomised `$table_prefix` (e.g. `bF6_`) — why the default `wp_` is a target
- Canonical URL decision before running the installer — why it matters for the database
- Browser installation wizard: language, site info form, success screen, confirmation email
- First-login housekeeping: General Settings, Permalinks (Post name), deleting Akismet/Hello Dolly/unused themes
- Admin display name update (login username remains the random string)
- Maintenance mode plugin installation

---

### [Chapter 34 — WordPress Hardening: Introduction & Log File Inspection](./chapters/34-wordpress-hardening-introduction.md)
An overview of the full WordPress hardening process and the habit of checking logs before every change.

- Hardening areas: PHP-FPM pools, permissions, `open_basedir`, dangerous functions, SSL, headers, WAF rules, rate limiting, DB privileges, REST API
- `/var/log/nginx/` log directory layout; site-specific vs global logs; log rotation
- `sudo cat` vs `sudo less` — when to use each
- Error log: benign `favicon.ico` 404 and how to suppress it
- Access log: live brute-force attack on `xmlrpc.php` — reading the entry format and spotting the pattern

---

### [Chapter 35 — PHP-FPM Pool Isolation](./chapters/35-php-fpm-pool-isolation.md)
Creating a dedicated PHP-FPM pool per WordPress site — the structural security foundation for everything else.

- Why pool isolation prevents lateral movement between sites on the same server
- `useradd` for the pool user; `usermod` group assignments for `www-data` and `$USER`
- Copying `www.conf` → `expertwp.help.conf`; editing pool name, user/group, socket path
- `rlimit_files = 15000`, `rlimit_core = 100` — per-pool resource limits
- `php_flag[display_errors] = off`, per-pool error log path, `log_errors = on`
- Creating the log file with `touch`, correct `chown` and `chmod 660`
- Updating `fastcgi_pass` in the Nginx server block to the site-specific socket
- Verifying the socket exists after `systemctl reload php8.3-fpm`

---

## Upcoming Chapters

> These chapters are planned and will be added as each component is configured.

| Chapter | Title |
|---------|-------|
| 36 | WordPress Filesystem Ownership & Permissions Hardening |
| 37 | PHP open_basedir & Dangerous Function Restrictions |
| 38 | SSL Certificates with Let's Encrypt & Certbot |
| 39 | Nginx Security Directives & Rate Limiting |
| 40 | HTTP Security Response Headers |
| 41 | Cloudflare Proxy Activation & Performance Tuning |

---

[← Back to README](./README.md)
