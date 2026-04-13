# Linux & WordPress Server Setup Guide

A complete, hands-on documentation of building a hardened, performance-optimised WordPress hosting environment from scratch on **Ubuntu 24.04 LTS**.

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
- Fully hardened HTTP headers and file permissions

---

## How to Read This Guide

Start with the **[Table of Contents](./index.md)** for the full chapter index.

Jump-in points by experience level:

- New to Linux → **Chapter 1**
- Comfortable with Linux, starting fresh → **Chapter 6**
- Server provisioned and hardened → **Chapter 11**
- OS optimised, ready to install the stack → **Chapter 17**
- Stack installed, ready to configure Nginx/MariaDB → **Chapter 21**
- Nginx/MariaDB tuned, ready to configure PHP → **Chapter 26**
- PHP configured, ready to install WordPress → **Chapter 31**
- WordPress installed, ready to harden and enable HTTPS → **Chapter 36**
- SSL live, ready for security headers and firewall rules → **Chapter 41**
- Nginx hardening complete, final WordPress hardening → **Chapter 46**

---

## Contents at a Glance

| Part | Chapters | Topics |
|------|----------|--------|
| I — Linux Fundamentals | 1–5 | Shell, filesystem, permissions, `apt`, `systemctl`, SSH, cron |
| II — Server Build & Hardening | 6–10 | Provisioning, first login, root disabled, SSH key auth |
| III — Security Layers & OS Optimisation | 11–16 | Firewall, Fail2Ban, sysctl, IPv6, swap, PAM limits |
| IV — Web Stack Installation | 17–20 | DNS, LEMP install, msmtp, Nginx config fundamentals |
| V — Nginx & MariaDB Tuning | 21–25 | includes/ directory, InnoDB tuning, mariadb secure |
| VI — PHP Hardening & Site Prep | 26–30 | server_override.ini, OPcache, web root, server block |
| VII — WordPress Installation | 31–35 | DB setup, wp-config.php, browser install, pool isolation |
| VIII — Permissions & SSL | 36–40 | Pool ownership, 550/440 hardened mode, open_basedir, Certbot |
| IX — Security Headers, Rate Limiting & WAF | 41–45 | HTTP headers, Nginx firewall ruleset, rate limiting, NinjaFirewall |
| X — Final WordPress Application Hardening | 46–49 | Hotlinking, DISALLOW_FILE_MODS, DB privileges, REST API |

---

## Status

- [x] Part I — Linux Fundamentals (Ch 1–5)
- [x] Part II — Server Build & Hardening (Ch 6–10)
- [x] Part III — Security Layers & OS Optimisation (Ch 11–16)
- [x] Part IV — Web Stack Installation & Configuration (Ch 17–20)
- [x] Part V — Nginx & MariaDB Hardening and Tuning (Ch 21–25)
- [x] Part VI — PHP Hardening, Tuning & Site Preparation (Ch 26–30)
- [x] Part VII — WordPress Installation & Hardening (Ch 31–35)
- [x] Part VIII — PHP Pool Hardening, Permissions & SSL (Ch 36–40)
- [x] Part IX — Nginx Security Headers, Rate Limiting & WAF (Ch 41–45)
- [x] Part X — Final WordPress Application Hardening (Ch 46–49)
- [ ] Part XI — OPcache, FastCGI Caching & Cloudflare Proxy

---

## License

Personal reference documentation. Not licensed for redistribution.
