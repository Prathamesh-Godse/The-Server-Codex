# Index — Production WordPress Server Documentation

A quick-reference index of every section in this guide, grouped by topic. Each entry links to the corresponding documentation file.

---

## Foundations

| # | Section | Key Topics |
|---|---------|------------|
| 01 | [Project Overview](01-nginx-wordpress-server.md) | Stack overview, goals, infrastructure, config principles |
| 02 | [Linux Essentials](02-linux-essentials.md) | Commands, file system, man pages |
| 03 | [Linux File System](03-linux-file-system.md) | FHS, `/etc`, `/var`, `/usr`, paths |
| 04 | [Users, Groups & Permissions](04-users-groups-permissions.md) | `chmod`, `chown`, `sudo`, user management |
| 05 | [Essential Linux Skills](05-essential-linux-skills.md) | `nano`, `apt`, `systemctl`, SSH, cron, bash scripts |

---

## Server Provisioning

| # | Section | Key Topics |
|---|---------|------------|
| 06 | [Local Setup & Server OS](06-local-setup-and-server-os.md) | Choosing a VPS, Ubuntu 24.04 LTS |
| 07 | [Server Provisioning](07-server-provisioning.md) | VPS setup, initial SSH access |
| 08 | [First Login & Hardening Intro](08-first-login-and-hardening-intro.md) | Root login, first steps |

---

## Server Hardening

| # | Section | Key Topics |
|---|---------|------------|
| 09 | [Server Hardening Overview](09-server-hardening.md) | Threat model, hardening plan |
| 10 | [Sudo, Updates & SSH Keys](10-hardening-sudo-updates-ssh-keys.md) | Non-root user, SSH key auth |
| 11 | [SSH Config & Server Updates](11_ssh_config_and_server_updates.md) | `sshd_config`, disabling root login, unattended upgrades |
| 12 | [Firewall — UFW & Cloud](12_firewall_ufw_and_cloud.md) | UFW rules, cloud firewall, port management |
| 13 | [Fail2Ban](13_fail2ban.md) | SSH jail, ban time, monitoring |
| 14 | [Harden & Optimise Distribution (1)](14_harden_and_optimize_server_distribution.md) | Kernel parameters, sysctl |
| 15 | [Harden & Optimise Distribution (2)](15_harden_and_optimize_server_distribution_2.md) | Additional sysctl, shared memory |
| 16 | [Congestion Control & File Limits](16_congestion_control_file_access_open_file_limits.md) | BBR, `fs.file-max`, `ulimit` |

---

## DNS & Domain

| # | Section | Key Topics |
|---|---------|------------|
| 17 | [DNS, Cloudflare & Domain Setup](17_dns_cloudflare_domain_setup.md) | Nameservers, A records, Cloudflare proxy |

---

## LEMP Stack Installation

| # | Section | Key Topics |
|---|---------|------------|
| 18 | [Installing Nginx, MariaDB & PHP](18_installing_nginx_mariadb_php.md) | `apt install`, initial service checks |
| 19 | [Server Mail — msmtp](19_server_mail_msmtp.md) | msmtp config, test email, cron mail relay |

---

## Nginx Configuration

| # | Section | Key Topics |
|---|---------|------------|
| 20 | [Nginx Configuration Files](20_nginx_configuration_files.md) | `nginx.conf`, includes directory structure |
| 21 | [Harden & Optimise Nginx](21_nginx_harden_and_optimize.md) | Worker processes, buffers, timeouts, gzip, brotli |
| 22 | [Config Wiring & Reload](22_nginx_conf_wiring_and_reload.md) | Including snippets, `nginx -t`, reload workflow |
| 23 | [Open File Limits & Bash Aliases](23-section-11-nginx-open-file-limits-and-bash-aliases.md) | `worker_rlimit_nofile`, aliases for `ngt` / `ngr` |

---

## MariaDB Configuration

| # | Section | Key Topics |
|---|---------|------------|
| 24 | [Harden MariaDB](section-24-harden-mariadb.md) | `mysql_secure_installation`, bind address, socket auth |
| 25 | [Optimise MariaDB](section-25-optimize-mariadb.md) | InnoDB buffer pool, query cache, slow query log |
| 26 | [MySQLTuner & Open File Limits](section-26-mysqltuner-mariadb-open-file-limits.md) | MySQLTuner, `open_files_limit` |

---

## PHP Configuration

| # | Section | Key Topics |
|---|---------|------------|
| 27 | [Harden & Optimise PHP](section-27-harden-and-optimize-php.md) | PHP 8.3, `conf.d` overrides, upload limits, memory |
| 28 | [OPcache & Open File Limits](section-28-optimize-php-opcache-open-file-limit.md) | OPcache tuning, `rlimit_files` in FPM |

---

## Web Root & Server Blocks

| # | Section | Key Topics |
|---|---------|------------|
| 29 | [Web Root Directory Structure](section-29-web-root-directory-structure.md) | `/var/www` layout, `public_html` |
| 30 | [Server Blocks Overview](section-30-nginx-server-blocks-browser-caching-fastcgi.md) | `sites-available`, `sites-enabled`, symlinks |
| 31 | [FastCGI & Browser Caching](31_nginx_server_block_fastcgi_browser_caching.md) | `fastcgi_optimize.conf`, `browser_caching.conf`, server block wiring |

---

## WordPress Installation

| # | Section | Key Topics |
|---|---------|------------|
| 32 | [MariaDB & WordPress Prep](32_mariadb_wordpress_installation_prep.md) | DB creation, user grants, collation |
| 33 | [WordPress Config & Deployment](33_wordpress_config_deployment_setup.md) | `wp-config.php`, WP-CLI, initial install |

---

## WordPress Hardening

| # | Section | Key Topics |
|---|---------|------------|
| 34 | [Hardening Introduction](34_wordpress_hardening_introduction.md) | Threat model, hardening plan |
| 35 | [PHP-FPM Pool Isolation](35_php_fpm_pool_isolation.md) | Dedicated pool per site, socket path |
| 36 | [PHP-FPM Pool Hardening](36_php_fpm_pool_hardening.md) | `chroot`, `php_admin_value`, `open_basedir` in pool |
| 37 | [Hardened File Permissions](37_wordpress_hardened_permissions.md) | `find` + `chmod`/`chown` scripts, `www-data` ownership |
| 38 | [open_basedir](section-38-open-basedir.md) | Restricting PHP file access to the site root |

---

## SSL & HTTPS

| # | Section | Key Topics |
|---|---------|------------|
| 39 | [SSL Certificates](section-39-ssl-certificates.md) | Certbot webroot, DH params, `ssl_all_sites.conf`, HSTS, HTTP/3 |
| 41 | [HTTPS Verification & Renewal](41-https-verification-certbot-renewal.md) | `curl -I`, SSL Labs A+, HTTP/3 check, mixed content fix, cron renewal |

---

## Security Headers & Nginx Hardening

| # | Section | Key Topics |
|---|---------|------------|
| 42 | [HTTP Response Headers](42_http_response_headers.md) | CSP, X-Frame-Options, Referrer-Policy, Permissions-Policy |
| 43 | [Nginx WordPress Security Directives](43_nginx_wordpress_security_directives.md) | Block `xmlrpc.php`, `readme.html`, uploads PHP execution |
| 44 | [Nginx DDoS Considerations](44_nginx_ddos_protection.md) | Why Cloudflare is preferred over Nginx-level rate limiting for DDoS |
| 45 | [Nginx Rate Limiting](45_nginx_rate_limiting.md) | `limit_req_zone` for `wp-login.php` and `xmlrpc.php` |
| 46 | [Web Application Firewall](46_web_application_firewall.md) | WAF rules, bad bot blocking |
| 47 | [Hotlinking Protection](47_hotlinking_protection.md) | `valid_referers`, bandwidth protection |
| 48 | [Disallow File Modifications](48_disallow_file_mods.md) | `DISALLOW_FILE_EDIT`, `DISALLOW_FILE_MODS` in `wp-config.php` |
| 49 | [Database Hardening](49_database_hardening.md) | WordPress DB prefix, table hardening |
| 50 | [REST API Hardening](50_rest_api_hardening.md) | Restricting unauthenticated REST API access |

---

## WordPress Optimisation

| # | Section | Key Topics |
|---|---------|------------|
| 51 | [Optimisation Overview](51-wordpress-optimization-overview.md) | Caching strategy, performance goals |
| 52 | [Post Revisions Policy](52-post-revisions-policy.md) | `WP_POST_REVISIONS` in `wp-config.php` |
| 53 | [Max Memory Limit](53-setting-max-memory-limit.md) | `WP_MEMORY_LIMIT`, `WP_MAX_MEMORY_LIMIT` |

---

*Each section is self-contained. Commands are written for Ubuntu 24.04 LTS and can be run in sequence from top to bottom.*
