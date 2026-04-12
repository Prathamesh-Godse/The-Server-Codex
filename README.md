# Production WordPress Server on Ubuntu 24.04

A fully documented, step-by-step guide to building a hardened, performance-optimised WordPress hosting environment on Ubuntu 24.04 LTS using Nginx, MariaDB, and PHP-FPM. Designed to host multiple isolated WordPress sites on a single VPS with free SSL, server-side caching, and Cloudflare DNS.

---

## Goals

| Target | Result |
|--------|--------|
| Google PageSpeed | **97–100%** |
| SSL Rating (SSL Labs) | **A+** |
| HTTP Header Rating | **A / A+** |
| Multi-site isolation | Per-site PHP-FPM pools |
| SSL certificates | Wildcard via Let's Encrypt / Certbot |
| DDoS mitigation | Cloudflare edge protection |

---

## Stack

| Component | Role |
|-----------|------|
| **Ubuntu 24.04 LTS** | Operating system |
| **Nginx** | Web server and reverse proxy |
| **MariaDB** | Database server |
| **PHP 8.3 + PHP-FPM** | Server-side scripting with per-site pools |
| **WordPress** | CMS |
| **Certbot + Let's Encrypt** | Free SSL/TLS certificates with auto-renewal |
| **Cloudflare** | DNS management, CDN, and DDoS mitigation |
| **Fail2Ban** | SSH brute-force protection |
| **UFW** | Host-level firewall |

---

## Documentation Index

### Part 1 — Linux & Infrastructure Foundations
- [01 — Project Overview](01-nginx-wordpress-server.md)
- [02 — Linux Essentials](02-linux-essentials.md)
- [03 — Linux File System](03-linux-file-system.md)
- [04 — Users, Groups & Permissions](04-users-groups-permissions.md)
- [05 — Essential Linux Skills](05-essential-linux-skills.md)

### Part 2 — Server Provisioning & Initial Setup
- [06 — Local Setup & Server OS](06-local-setup-and-server-os.md)
- [07 — Server Provisioning](07-server-provisioning.md)
- [08 — First Login & Hardening Intro](08-first-login-and-hardening-intro.md)

### Part 3 — Server Hardening
- [09 — Server Hardening Overview](09-server-hardening.md)
- [10 — Sudo, Updates & SSH Keys](10-hardening-sudo-updates-ssh-keys.md)
- [11 — SSH Config & Server Updates](11_ssh_config_and_server_updates.md)
- [12 — Firewall (UFW) & Cloud Firewalls](12_firewall_ufw_and_cloud.md)
- [13 — Fail2Ban](13_fail2ban.md)
- [14 — Harden & Optimise Server Distribution](14_harden_and_optimize_server_distribution.md)
- [15 — Harden & Optimise Server Distribution (Part 2)](15_harden_and_optimize_server_distribution_2.md)
- [16 — Congestion Control, File Access & Open File Limits](16_congestion_control_file_access_open_file_limits.md)

### Part 4 — DNS & Domain Setup
- [17 — DNS, Cloudflare & Domain Setup](17_dns_cloudflare_domain_setup.md)

### Part 5 — Installing the LEMP Stack
- [18 — Installing Nginx, MariaDB & PHP](18_installing_nginx_mariadb_php.md)
- [19 — Server Mail (msmtp)](19_server_mail_msmtp.md)

### Part 6 — Nginx Configuration
- [20 — Nginx Configuration Files](20_nginx_configuration_files.md)
- [21 — Harden & Optimise Nginx](21_nginx_harden_and_optimize.md)
- [22 — Nginx Config Wiring & Reload](22_nginx_conf_wiring_and_reload.md)
- [23 — Nginx Open File Limits & Bash Aliases](23-section-11-nginx-open-file-limits-and-bash-aliases.md)

### Part 7 — MariaDB Configuration
- [24 — Harden MariaDB](section-24-harden-mariadb.md)
- [25 — Optimise MariaDB](section-25-optimize-mariadb.md)
- [26 — MySQLTuner & Open File Limits](section-26-mysqltuner-mariadb-open-file-limits.md)

### Part 8 — PHP Configuration
- [27 — Harden & Optimise PHP](section-27-harden-and-optimize-php.md)
- [28 — OPcache & Open File Limits](section-28-optimize-php-opcache-open-file-limit.md)

### Part 9 — Web Root & Server Blocks
- [29 — Web Root Directory Structure](section-29-web-root-directory-structure.md)
- [30 — Nginx Server Blocks, Browser Caching & FastCGI](section-30-nginx-server-blocks-browser-caching-fastcgi.md)
- [31 — Nginx Server Block: FastCGI & Browser Caching (Detail)](31_nginx_server_block_fastcgi_browser_caching.md)

### Part 10 — WordPress Installation
- [32 — MariaDB & WordPress Installation Prep](32_mariadb_wordpress_installation_prep.md)
- [33 — WordPress Config & Deployment Setup](33_wordpress_config_deployment_setup.md)

### Part 11 — WordPress Hardening
- [34 — WordPress Hardening Introduction](34_wordpress_hardening_introduction.md)
- [35 — PHP-FPM Pool Isolation](35_php_fpm_pool_isolation.md)
- [36 — PHP-FPM Pool Hardening](36_php_fpm_pool_hardening.md)
- [37 — WordPress Hardened Permissions](37_wordpress_hardened_permissions.md)
- [38 — open_basedir](section-38-open-basedir.md)

### Part 12 — SSL & HTTPS
- [39 — SSL Certificates (Certbot & Let's Encrypt)](section-39-ssl-certificates.md)
- [41 — HTTPS Verification & Certbot Renewal](41-https-verification-certbot-renewal.md)

### Part 13 — Security Headers & Nginx Hardening
- [42 — HTTP Response Headers](42_http_response_headers.md)
- [43 — Nginx WordPress Security Directives](43_nginx_wordpress_security_directives.md)
- [44 — Nginx DDoS Protection](44_nginx_ddos_protection.md)
- [45 — Nginx Rate Limiting](45_nginx_rate_limiting.md)
- [46 — Web Application Firewall](46_web_application_firewall.md)
- [47 — Hotlinking Protection](47_hotlinking_protection.md)
- [48 — Disallow File Modifications](48_disallow_file_mods.md)
- [49 — Database Hardening](49_database_hardening.md)
- [50 — REST API Hardening](50_rest_api_hardening.md)

### Part 14 — WordPress Optimisation
- [51 — WordPress Optimisation Overview](51-wordpress-optimization-overview.md)
- [52 — Post Revisions Policy](52-post-revisions-policy.md)
- [53 — Setting Max Memory Limit](53-setting-max-memory-limit.md)

---

## Prerequisites

- A publicly accessible VPS (Ubuntu 24.04 LTS) — local VMs cannot be used as SSL issuance requires a public IP
- A registered domain with DNS managed through Cloudflare
- Basic familiarity with: SSH, Linux CLI, `nano`, `systemctl`, and `apt`

---

## Configuration Principles

- Each config file is written once, completely from scratch — no incremental patching.
- Each component (Nginx, PHP, MariaDB) is configured as a cohesive unit.
- All changes are followed by a service reload/restart and a status check.
- Per-site isolation is enforced via dedicated PHP-FPM pools.

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Contributing

Contributions, corrections, and improvements are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a pull request.
