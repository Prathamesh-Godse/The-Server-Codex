# Linux & WordPress Server Setup Guide

A complete, hands-on documentation of building a hardened, performance-optimised WordPress hosting environment from scratch on **Ubuntu 24.04 LTS**. Written as a personal reference and structured as a book — each chapter covers one layer of the stack, from Linux fundamentals through to a fully operational, multi-site WordPress server.

---

## About This Guide

This guide is written for anyone who wants to understand **not just what to run, but why** — every command, configuration directive, and permission decision is explained in context. It assumes no prior Linux server experience, but does not hold back on the technical detail.

| | |
|---|---|
| **OS** | Ubuntu 24.04 LTS (Noble Numbat) |
| **Web Server** | Nginx |
| **Database** | MariaDB |
| **Language Runtime** | PHP with PHP-FPM pools |
| **CMS** | WordPress |
| **SSL** | Let's Encrypt via Certbot |
| **DNS / CDN** | Cloudflare |

---

## Goals

- Google PageSpeed score of **97–100%**
- **A+** SSL rating and **A(+)** HTTP security header rating
- Multiple WordPress sites, each isolated via **PHP-FPM pools**
- **Wildcard SSL certificates** via Let's Encrypt
- Server-side caching at the Nginx level
- Fully hardened HTTP headers and file permissions

---

## How to Read This Guide

Start with the **[Table of Contents](./index.md)** for a full overview and direct links to every chapter.

Chapters are designed to be read in order — later chapters build on concepts introduced earlier. Jump-in points by experience level:

- New to Linux → **Chapter 1**
- Comfortable with Linux, starting fresh → **Chapter 6**
- Server provisioned and hardened → **Chapter 11**
- OS optimised, ready to install the stack → **Chapter 17**
- Stack installed, ready to configure Nginx/MariaDB → **Chapter 21**
- Nginx/MariaDB tuned, ready to configure PHP and site structure → **Chapter 26**
- PHP configured, ready to install WordPress → **Chapter 31**

---

## Contents at a Glance

### Part I — Linux Fundamentals

| Ch | Title | Topics |
|----|-------|--------|
| 1 | [Project Overview](./chapters/01-nginx-wordpress-server.md) | Stack, goals, infrastructure, config principles |
| 2 | [Linux Essentials](./chapters/02-linux-essentials.md) | Distributions, shell, SSH, terminal, `ls` |
| 3 | [Linux File System](./chapters/03-linux-file-system.md) | Directory hierarchy, naming, `pwd`, `cd`, paths |
| 4 | [Users, Groups & Permissions](./chapters/04-users-groups-permissions.md) | User types, `sudo`, ownership, `chmod`, `chown` |
| 5 | [Essential Linux Skills](./chapters/05-essential-linux-skills.md) | `nano`, `apt`, `systemctl`, SSH keys, bash, cron |

### Part II — Server Build & Hardening

| Ch | Title | Topics |
|----|-------|--------|
| 6 | [Local Machine Setup](./chapters/06-local-setup-and-server-os.md) | Local vs remote, why Ubuntu 24.04, local tooling |
| 7 | [Server Provisioning](./chapters/07-server-provisioning.md) | Specs, Ubuntu Pro, migration strategy, Vultr |
| 8 | [First Login & Hardening Intro](./chapters/08-first-login-and-hardening-intro.md) | Root SSH, fingerprints, known_hosts |
| 9 | [Server Hardening](./chapters/09-server-hardening.md) | Password, non-root user, sudo, root SSH disabled |
| 10 | [sudo, Updates & SSH Key Auth](./chapters/10-hardening-sudo-updates-ssh-keys.md) | sudo, kernel updates, key auth, password disabled |

### Part III — Security Layers & OS Optimisation

| Ch | Title | Topics |
|----|-------|--------|
| 11 | [SSH Config & Server Updates](./chapters/11-ssh-config-and-server-updates.md) | SSH alias, apt update cycle |
| 12 | [Firewall: UFW & Cloud](./chapters/12-firewall-ufw-and-cloud.md) | UFW, default deny, Vultr cloud firewall |
| 13 | [Fail2Ban](./chapters/13-fail2ban.md) | Installation, jail.local, SSH jail, monitoring |
| 14 | [Harden & Optimise OS — Part 1](./chapters/14-harden-and-optimize-server-distribution.md) | Timezone, swap file, fstab |
| 15 | [Harden & Optimise OS — Part 2](./chapters/15-harden-and-optimize-server-distribution-2.md) | sysctl, shared memory, IPv6, network tuning |
| 16 | [Congestion Control & File Limits](./chapters/16-congestion-control-file-access-open-file-limits.md) | BBR, noatime, PAM limits |

### Part IV — Web Stack Installation & Configuration

| Ch | Title | Topics |
|----|-------|--------|
| 17 | [DNS & Cloudflare Setup](./chapters/17-dns-cloudflare-domain-setup.md) | Nameservers, A record, CNAME |
| 18 | [Installing Nginx, MariaDB & PHP](./chapters/18-installing-nginx-mariadb-php.md) | Ondrej PPA, LEMP stack, PHP extensions |
| 19 | [Server Mail with msmtp](./chapters/19-server-mail-msmtp.md) | msmtp, Gmail SMTP, PHP mail() |
| 20 | [Nginx Configuration Files](./chapters/20-nginx-configuration-files.md) | Directives, contexts, location modifiers, try_files |

### Part V — Nginx & MariaDB Hardening and Tuning

| Ch | Title | Topics |
|----|-------|--------|
| 21 | [Hardening & Optimising Nginx](./chapters/21-nginx-harden-and-optimize.md) | main/events context, includes/ directory, six conf files |
| 22 | [Wiring Nginx Include Files](./chapters/22-nginx-conf-wiring-and-reload.md) | Populating files, http context rewrite, error fixes |
| 23 | [Nginx Open File Limits & Bash Aliases](./chapters/23-nginx-open-file-limits-and-bash-aliases.md) | worker_rlimit_nofile 45000, /proc/PID/limits |
| 24 | [Harden MariaDB](./chapters/24-harden-mariadb.md) | mysql_secure_installation, Unix socket auth |
| 25 | [Optimise MariaDB](./chapters/25-optimize-mariadb.md) | Performance Schema, InnoDB tuning, log retention |

### Part VI — PHP Hardening, Tuning & Site Preparation

| Ch | Title | Topics |
|----|-------|--------|
| 26 | [MySQLTuner & MariaDB Open File Limits](./chapters/26-mysqltuner-mariadb-open-file-limits.md) | MySQLTuner, systemd drop-in LimitNOFILE=40000 |
| 27 | [Harden & Optimise PHP](./chapters/27-harden-and-optimize-php.md) | server_override.ini, hardening + optimisation directives |
| 28 | [Optimise PHP: OPcache & Open File Limit](./chapters/28-optimize-php-opcache-open-file-limit.md) | OPcache theory, rlimit_files=32768, worker verification |
| 29 | [Web Root Directory Structure](./chapters/29-web-root-directory-structure.md) | /var/www layout, mkdir -p, rmdir, rm -rf, mv |
| 30 | [Nginx Server Blocks, Browser Caching & FastCGI](./chapters/30-nginx-server-blocks-browser-caching-fastcgi.md) | Virtual host, include files, symlink, live error demo |

### Part VII — WordPress Installation & Hardening

| Ch | Title | Topics |
|----|-------|--------|
| 31 | [Nginx Server Block: FastCGI & Browser Caching](./chapters/31-nginx-server-block-fastcgi-browser-caching.md) | Full implementation session, path typo fix, 403 verification |
| 32 | [MariaDB Setup & WordPress Prep](./chapters/32-mariadb-wordpress-installation-prep.md) | SQL fundamentals, random credentials, DB/user creation, rsync deploy |
| 33 | [WordPress: wp-config.php, Deployment & Setup](./chapters/33-wordpress-config-deployment-setup.md) | wp-config.php in full, browser wizard, first-login housekeeping |
| 34 | [WordPress Hardening Introduction](./chapters/34-wordpress-hardening-introduction.md) | Hardening areas overview, log inspection, attack traffic in access log |
| 35 | [PHP-FPM Pool Isolation](./chapters/35-php-fpm-pool-isolation.md) | Dedicated pool user, socket, resource limits, Nginx socket update |

---

## Status

- [x] Part I — Linux Fundamentals (Ch 1–5)
- [x] Part II — Server Build & Hardening (Ch 6–10)
- [x] Part III — Security Layers & OS Optimisation (Ch 11–16)
- [x] Part IV — Web Stack Installation & Configuration (Ch 17–20)
- [x] Part V — Nginx & MariaDB Hardening and Tuning (Ch 21–25)
- [x] Part VI — PHP Hardening, Tuning & Site Preparation (Ch 26–30)
- [x] Part VII — WordPress Installation & Hardening (Ch 31–35)
- [ ] Part VIII — SSL, Security Headers & Cloudflare Proxy
- [ ] Part IX — Performance Tuning & Caching

---

## License

Personal reference documentation. Not licensed for redistribution.
