# Table of Contents

> **Linux & WordPress Server Setup Guide** — Full chapter index.  
> Start at the [README](./README.md) for an overview of this guide.

---

## Part I — Linux Fundamentals

### [Chapter 1 — Project Overview](./chapters/01-nginx-wordpress-server.md)
Stack, goals, infrastructure, and configuration philosophy.

### [Chapter 2 — Linux Essentials](./chapters/02-linux-essentials.md)
Distributions, shell, SSH, terminal emulators, prompt anatomy, `ls`.

### [Chapter 3 — Linux File System](./chapters/03-linux-file-system.md)
Directory hierarchy, naming rules, `pwd`, `cd`, absolute vs relative paths.

### [Chapter 4 — Users, Groups, Ownership & Permissions](./chapters/04-users-groups-permissions.md)
User types, `sudo`, groups, ownership, permission types, `chmod`, `chown`, the `777` warning.

### [Chapter 5 — Essential Linux Skills](./chapters/05-essential-linux-skills.md)
`nano`, `apt`, `systemctl`, SSH keys, bash scripts, cron jobs.

---

## Part II — Server Build & Hardening

### [Chapter 6 — Local Machine Setup & Server OS Choice](./chapters/06-local-setup-and-server-os.md)
Local vs remote, why Ubuntu 24.04 LTS, local tooling requirements.

### [Chapter 7 — Server Provisioning](./chapters/07-server-provisioning.md)
Dev vs production specs; Ubuntu Pro; never-upgrade-in-place rule; Vultr instance creation.

### [Chapter 8 — First Login via SSH & Hardening Introduction](./chapters/08-first-login-and-hardening-intro.md)
Root SSH login, fingerprint acceptance, `known_hosts`, what hardening involves and why.

### [Chapter 9 — Server Hardening](./chapters/09-server-hardening.md)
Root password, non-root user, `visudo`, disabling `PermitRootLogin` in both SSH config files.

### [Chapter 10 — Hardening Continued: sudo, Updates & SSH Key Authentication](./chapters/10-hardening-sudo-updates-ssh-keys.md)
sudo usage, kernel updates, SSH key generation, disabling password authentication.

---

## Part III — Security Layers & OS Optimisation

### [Chapter 11 — SSH Config File & Server Updates](./chapters/11-ssh-config-and-server-updates.md)
SSH alias config file; four-step post-provisioning update sequence.

### [Chapter 12 — Firewall: UFW & Cloud Firewall](./chapters/12-firewall-ufw-and-cloud.md)
UFW at the OS level and the Vultr cloud firewall at the network perimeter.

### [Chapter 13 — Fail2Ban: Installation, Configuration & Management](./chapters/13-fail2ban.md)
Fail2Ban from the official GitHub release; 7-day SSH ban policy; monitoring and unbanning.

### [Chapter 14 — Hardening & Optimising the Server Distribution](./chapters/14-harden-and-optimize-server-distribution.md)
Timezone, swap file creation, `/etc/fstab` configuration.

### [Chapter 15 — Hardening & Optimising the Server Distribution — Part 2](./chapters/15-harden-and-optimize-server-distribution-2.md)
`sysctl` kernel tuning, shared memory hardening, IPv6 disabling, network layer directives.

### [Chapter 16 — Congestion Control, File Access Times & Open File Limits](./chapters/16-congestion-control-file-access-open-file-limits.md)
BBR verification, `noatime`, PAM open file limits raised to 120,000.

---

## Part IV — Web Stack Installation & Configuration

### [Chapter 17 — DNS: Pointing a Domain to the Server](./chapters/17-dns-cloudflare-domain-setup.md)
Cloudflare nameserver delegation, A record, CNAME record, DNS-only proxy status.

### [Chapter 18 — Installing the Hosting Stack: Nginx, MariaDB & PHP 8.3](./chapters/18-installing-nginx-mariadb-php.md)
LEMP stack — Ondrej PPA, Brotli/cache-purge modules, PHP 8.3 with 15 extensions.

### [Chapter 19 — Server Mail Without SMTP Plugins](./chapters/19-server-mail-msmtp.md)
msmtp as a `sendmail` drop-in; Gmail SMTP relay; PHP `mail()` configuration and testing.

### [Chapter 20 — Nginx Configuration Files](./chapters/20-nginx-configuration-files.md)
Directives, contexts, location modifier priority, `try_files`, WordPress permalink mechanics.

---

## Part V — Nginx & MariaDB Hardening and Tuning

### [Chapter 21 — Hardening & Optimising Nginx](./chapters/21-nginx-harden-and-optimize.md)
`main`/`events` context tuning; the `includes/` directory; all six include files populated.

### [Chapter 22 — Completing the Nginx Configuration: Wiring the Include Files](./chapters/22-nginx-conf-wiring-and-reload.md)
Live session: populating files, rewriting the http context, two real config errors and fixes.

### [Chapter 23 — Nginx Open File Limits & Bash Aliases](./chapters/23-nginx-open-file-limits-and-bash-aliases.md)
`worker_rlimit_nofile` raised to 45000; verified via `/proc/<PID>/limits`; bash aliases.

### [Chapter 24 — Harden MariaDB](./chapters/24-harden-mariadb.md)
`mysql_secure_installation` — all six prompts with reasons; Unix socket authentication explained.

### [Chapter 25 — Optimise MariaDB](./chapters/25-optimize-mariadb.md)
Performance Schema; `skip-name-resolve`; binary log retention; InnoDB buffer pool 800M / log 200M.

---

## Part VI — PHP Hardening, Tuning & Site Preparation

### [Chapter 26 — MySQLTuner & MariaDB: Raising the Open File Limit](./chapters/26-mysqltuner-mariadb-open-file-limits.md)
MySQLTuner periodic health check; systemd drop-in `LimitNOFILE=40000` for MariaDB.

### [Chapter 27 — Harden & Optimise PHP](./chapters/27-harden-and-optimize-php.md)
PHP config hierarchy; `server_override.ini`; `allow_url_fopen`, `cgi.fix_pathinfo`, `expose_php`; upload limits; `rlimit_files=32768`.

### [Chapter 28 — Optimise PHP: OPcache & PHP-FPM Open File Limit](./chapters/28-optimize-php-opcache-open-file-limit.md)
OPcache theory; `validate_timestamps` dev vs production; `rlimit_files` verified on master and workers.

### [Chapter 29 — Web Root File and Directory Structure](./chapters/29-web-root-directory-structure.md)
`/var/www/` layout; `mkdir -p`; `rmdir` vs `rm -rf`; `mv` for renaming; site document root created.

### [Chapter 30 — Nginx Server Blocks, Browser Caching & FastCGI Optimisation](./chapters/30-nginx-server-blocks-browser-caching-fastcgi.md)
Server block theory; `sites-available`/`sites-enabled` pattern; live reload-vs-restart demo; include files; full server block.

---

## Part VII — WordPress Installation & Hardening

### [Chapter 31 — Nginx Server Block: FastCGI Optimisation & Browser Caching](./chapters/31-nginx-server-block-fastcgi-browser-caching.md)
Full implementation session — include files, server block, symlink, path typo fix, 403 curl verification.

### [Chapter 32 — MariaDB Database Setup & WordPress Installation Preparation](./chapters/32-mariadb-wordpress-installation-prep.md)
SQL fundamentals; random credential generation; DB/user creation; WordPress download, extraction, and `rsync` deploy.

### [Chapter 33 — WordPress: wp-config.php, Deployment & Browser Setup](./chapters/33-wordpress-config-deployment-setup.md)
`wp-config.php` in full detail; browser installation wizard; first-login housekeeping.

### [Chapter 34 — WordPress Hardening: Introduction & Log File Inspection](./chapters/34-wordpress-hardening-introduction.md)
Hardening areas overview; Nginx log directory layout; live brute-force attack in the access log.

### [Chapter 35 — PHP-FPM Pool Isolation](./chapters/35-php-fpm-pool-isolation.md)
Dedicated pool user; socket; resource limits; error logging; Nginx `fastcgi_pass` updated to site-specific socket.

---

## Part VIII — PHP Pool Hardening, Permissions & SSL

### [Chapter 36 — PHP-FPM Pool Hardening: Ownership, Permissions & Per-Pool PHP Configuration](./chapters/36-php-fpm-pool-hardening.md)
Transferring ownership to the pool user; directory `770` / file `660`; `phpinfo()` test; `allow_url_fopen` per-pool override; `disable_functions` hardened list.

### [Chapter 37 — WordPress Hardened File Permissions: Standard vs Hardened Mode](./chapters/37-wordpress-hardened-permissions.md)
The two permission models — `770`/`660` for development and `550`/`440` with writable `wp-content/` for production.

### [Chapter 38 — Restricting PHP File Access with open_basedir](./chapters/38-open-basedir.md)
Site-specific `tmp/` directory; `open_basedir`, `upload_tmp_dir`, and `sys_temp_dir` per pool.

### [Chapter 39 — SSL/TLS Certificates: Certbot, Let's Encrypt & Nginx HTTPS Configuration](./chapters/39-ssl-certificates.md)
Webroot challenge certificate issuance, DH parameters, and two modular SSL include files.

### [Chapter 40 — HTTPS Verification, Fixing Mixed Content & Automating SSL Renewal](./chapters/40-https-verification-certbot-renewal.md)
`curl -I` verification; SSL Labs A+; HTTP/3 via http3check.net; WordPress URL update; root crontab renewal automation.

---

## Part IX — Nginx Security Headers, Rate Limiting & WAF

### [Chapter 41 — HTTP Response Headers](./chapters/41-http-response-headers.md)
Security-focused HTTP response headers via a modular include file — applied to both dynamic responses and static assets.

- `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Referrer-Policy`, `Permissions-Policy`
- Why `add_header` does not inherit from parent `location` contexts — why `browser_caching.conf` must also include `http_headers.conf`
- `etag on`, `if_modified_since exact`, `Pragma "public"` added to caching blocks

---

### [Chapter 42 — Nginx WordPress Security Directives](./chapters/42-nginx-wordpress-security-directives.md)
A ~100-line WordPress-specific Nginx firewall ruleset in a single reusable include file.

- Blocking `wp-config.php`, `readme.html`, `wp-admin/install.php`, `wp-includes/` PHP files
- Denying PHP execution in `uploads/`, `plugins/`, and `themes/` with case-insensitive regex
- The `set $susquery` variable pattern for aggregating multiple `if` conditions
- SQL injection, file injection, common exploit, spam, and bad user agent blocking
- Final full server block structure after all includes from Parts VIII and IX

---

### [Chapter 43 — Nginx DDoS Protection: Considerations for WordPress](./chapters/43-nginx-ddos-protection.md)
Why Nginx-level DDoS mitigation doesn't work for WordPress and where protection should actually be applied — Cloudflare or hosting provider infrastructure level.

---

### [Chapter 44 — Nginx Rate Limiting for WordPress](./chapters/44-nginx-rate-limiting.md)
Targeted rate limiting for `wp-login.php` and `xmlrpc.php` — HTTP `444` on limit exceeded; `limit_req_zone` in `http` block; `burst=20 nodelay` per-site include file.

---

### [Chapter 45 — Web Application Firewall: NinjaFirewall](./chapters/45-web-application-firewall.md)
Installing NinjaFirewall in Full WAF mode to intercept all PHP requests before WordPress loads. `.user.ini` bootstrap; sandbox mode; Full WAF configuration choices.

---

## Part X — Final WordPress Application Hardening

The last layer of the WordPress hardening process — application-level measures that lock down bandwidth theft, file modification attacks, database privileges, and unauthenticated API access.

### [Chapter 46 — Protecting Site Assets with Hotlinking Protection](./chapters/46-hotlinking-protection.md)
Preventing external sites from embedding your images and consuming your server bandwidth.

- **Cloudflare approach:** Scrape Shield → Hotlink Protection toggle; covered file types: gif, ico, jpg, jpeg, png; `hotlink-ok/` directory for exceptions
- **Nginx approach:** `valid_referers none blocked server_names ~\.google\.`; `if ($invalid_referer) { return 403; }`
- Why the Nginx approach fails when Cloudflare proxy is active
- Side-by-side comparison: setup complexity, proxy compatibility, granular control, origin performance impact

---

### [Chapter 47 — Hardening WordPress: Disabling File Modifications via DISALLOW_FILE_MODS](./chapters/47-disallow-file-mods.md)
The most impactful single-line WordPress hardening measure — preventing all filesystem writes from the admin dashboard.

- `DISALLOW_FILE_MODS = 'true'` added to `wp-config.php` custom block
- How it relates to `DISALLOW_FILE_EDIT`, `WP_AUTO_UPDATE_CORE`, and `AUTOMATIC_UPDATER_DISABLED`
- Trade-off: plugin and theme updates must now be done manually via SSH or WP-CLI
- Complete `wp-config.php` custom constants block in final hardened state

---

### [Chapter 48 — Hardening WordPress: Restricting MariaDB Database User Privileges](./chapters/48-database-hardening.md)
Tightening the WordPress database user from `ALL PRIVILEGES` to the minimum four: `SELECT`, `INSERT`, `UPDATE`, `DELETE`.

- Full MariaDB privilege reference table with descriptions
- `REVOKE ALL PRIVILEGES` → `GRANT SELECT, INSERT, UPDATE, DELETE` → `SHOW GRANTS` → `FLUSH PRIVILEGES`
- WooCommerce warning: do not apply this hardening to WooCommerce installations
- How to grant additional privileges for plugins that require `CREATE`, `ALTER`, or `INDEX` at activation time

---

### [Chapter 49 — Hardening WordPress: Restricting the REST API](./chapters/49-rest-api-hardening.md)
Blocking unauthenticated REST API access — closing the user enumeration endpoint and plugin vulnerability attack surface.

- Why the REST API cannot be completely disabled without breaking core WordPress
- The Bricks Builder RCE vulnerability as a real-world example of unauthenticated REST API exploitation
- `Disable REST API` plugin by Dave McHale — `401 rest_cannot_access` response for unauthenticated users
- Per-endpoint and per-role exception configuration in the plugin settings
- Compatibility notes: WooCommerce, page builders (Bricks, Elementor, Divi), wp-cron

---

## Upcoming Chapters

> These chapters are planned and will be added as each component is configured.

| Chapter | Title |
|---------|-------|
| 50 | OPcache Per-Pool Configuration |
| 51 | Server-Side Caching with FastCGI Cache |
| 52 | Cloudflare Proxy Activation |
| 53 | Adding a Second WordPress Site |

---

[← Back to README](./README.md)
