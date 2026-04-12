# Changelog

All notable changes to this project are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [Unreleased]

Changes staged for the next release will be listed here.

---

## [1.0.0] — 2024-07-20

### Added — Foundations & Server Setup
- Section 01: Project overview — stack, goals, infrastructure, and configuration principles
- Section 02: Linux essentials — commands, file system navigation, man pages
- Section 03: Linux file system hierarchy — `/etc`, `/var`, `/usr`, `/home`
- Section 04: Users, groups, and permissions — `chmod`, `chown`, `sudo`, user management
- Section 05: Essential Linux skills — `nano`, `apt`, `systemctl`, SSH, cron, bash scripting

### Added — Server Provisioning
- Section 06: Local setup and server OS — VPS selection, Ubuntu 24.04 LTS
- Section 07: Server provisioning — VPS creation, initial SSH access
- Section 08: First login and hardening introduction

### Added — Server Hardening
- Section 09: Server hardening overview — threat model and hardening plan
- Section 10: Sudo, system updates, and SSH key authentication
- Section 11: SSH configuration and unattended upgrades
- Section 12: UFW firewall and cloud-level firewall rules
- Section 13: Fail2Ban — SSH jail with 7-day ban, 3-hour findtime, 3 retry threshold
- Section 14 & 15: Kernel hardening via `sysctl` — network stack, shared memory, core dumps
- Section 16: TCP BBR congestion control, file access limits, system-wide open file limits

### Added — DNS & Domain
- Section 17: Cloudflare DNS setup — nameserver delegation, A record, CNAME, DNSSEC

### Added — LEMP Stack
- Section 18: Nginx, MariaDB, and PHP 8.3 installation
- Section 19: Server mail relay via msmtp

### Added — Nginx Configuration
- Section 20: `nginx.conf` and modular includes directory structure
- Section 21: Nginx hardening and optimisation — worker processes, buffers, gzip, brotli, timeouts
- Section 22: Nginx config wiring, include chain, and reload workflow
- Section 23: Nginx open file limits (`worker_rlimit_nofile`) and bash aliases (`ngt`, `ngr`)

### Added — MariaDB Configuration
- Section 24: MariaDB hardening — `mysql_secure_installation`, bind address, socket auth
- Section 25: MariaDB optimisation — InnoDB buffer pool, slow query log, query cache
- Section 26: MySQLTuner analysis and MariaDB open file limits

### Added — PHP Configuration
- Section 27: PHP 8.3 hardening and optimisation — `conf.d` override file, upload limits, memory
- Section 28: OPcache tuning and PHP-FPM open file limits

### Added — Web Root & Server Blocks
- Section 29: Web root directory structure — `/var/www/domain/public_html` layout
- Section 30 & 31: Nginx server blocks — `browser_caching.conf`, `fastcgi_optimize.conf`, FastCGI wiring

### Added — WordPress Installation
- Section 32: MariaDB database and user creation for WordPress
- Section 33: WordPress deployment — `wp-config.php`, WP-CLI install

### Added — WordPress Hardening
- Section 34: WordPress hardening overview
- Section 35: PHP-FPM pool isolation — dedicated socket per site
- Section 36: PHP-FPM pool hardening — `php_admin_value`, chroot, process limits
- Section 37: WordPress hardened file permissions — `find` + `chmod`/`chown` scripts
- Section 38: `open_basedir` restriction to site root

### Added — SSL & HTTPS
- Section 39: Certbot and Let's Encrypt — webroot challenge, DH parameters, `ssl_all_sites.conf`
  - TLS 1.2 + 1.3 only
  - HSTS with long `max-age`
  - HTTP/3 (QUIC) support
  - OCSP stapling
- Section 41: HTTPS verification — SSL Labs A+ confirmation, HTTP/3 check, mixed content fix, automated renewal cron

### Added — Security Headers & Nginx Hardening
- Section 42: HTTP response headers — CSP, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`
- Section 43: Nginx WordPress security directives — block `xmlrpc.php`, `readme.html`, PHP execution in `uploads/`, `plugins/`, `themes/`
- Section 44: DDoS protection rationale — Cloudflare edge vs Nginx-level rate limiting
- Section 45: Nginx rate limiting — `limit_req_zone` for `wp-login.php` and `xmlrpc.php`
- Section 46: Web application firewall — bad bot blocking, malicious UA filtering
- Section 47: Hotlinking protection — `valid_referers` Nginx directive
- Section 48: Disallow file modifications — `DISALLOW_FILE_EDIT`, `DISALLOW_FILE_MODS`
- Section 49: Database hardening — custom table prefix, hardened `wp_options`
- Section 50: REST API hardening — restricting unauthenticated access

### Added — WordPress Optimisation
- Section 51: WordPress optimisation overview — caching strategy, performance targets
- Section 52: Post revisions policy — `WP_POST_REVISIONS` in `wp-config.php`
- Section 53: Memory limits — `WP_MEMORY_LIMIT`, `WP_MAX_MEMORY_LIMIT`

---

[Unreleased]: https://github.com/your-username/repo-name/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/your-username/repo-name/releases/tag/v1.0.0
