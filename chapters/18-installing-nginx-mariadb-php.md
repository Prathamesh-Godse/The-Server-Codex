[← Back to Index](../index.md)

---

# Chapter 18 — Installing the Hosting Stack: Nginx, MariaDB & PHP 8.3

*Installing the full LEMP stack — Nginx from the Ondrej PPA for HTTP/3 support, MariaDB from the official Ubuntu repositories, and PHP 8.3 with all extensions required by WordPress.*

---

## Contents

- [18.1 Repository Strategy: Official vs Unofficial](#181-repository-strategy-official-vs-unofficial)
- [18.2 The apt Package Manager](#182-the-apt-package-manager)
- [18.3 Update and Upgrade the System](#183-update-and-upgrade-the-system)
- [18.4 Install Nginx](#184-install-nginx)
  - [Note on the Ondrej Nginx PPA Name](#note-on-the-ondrej-nginx-ppa-name-april-2025)
  - [Add the Ondrej Nginx PPA](#add-the-ondrej-nginx-ppa)
  - [Install Nginx with Required Modules](#install-nginx-with-required-modules)
  - [Fix the Post-Install Failure Caused by Disabled IPv6](#fix-the-post-install-failure-caused-by-disabled-ipv6)
  - [Verify Nginx is Running and Enabled](#verify-nginx-is-running-and-enabled)
  - [Confirm Nginx Version and Test the Default Page](#confirm-nginx-version-and-test-the-default-page)
- [18.5 Install MariaDB](#185-install-mariadb)
- [18.6 Install PHP 8.3](#186-install-php-83)
  - [How the Stack Fits Together (PHP-FPM)](#how-the-stack-fits-together-php-fpm)
  - [Add the Ondrej PHP PPA](#add-the-ondrej-php-ppa)
  - [Install PHP 8.3 with All Required Extensions](#install-php-83-with-all-required-extensions)
  - [Verify PHP-FPM is Running and Enabled](#verify-php-fpm-is-running-and-enabled)
- [18.7 Summary](#187-summary)

---

## 18.1 Repository Strategy: Official vs Unofficial

The general rule for Ubuntu package management is to install from official repositories wherever possible. Official packages have near-zero compatibility issues with other system software and are thoroughly tested for security.

For this stack, two **trusted unofficial PPAs** from Ondrej Surý are used for Nginx and PHP. This is a deliberate trade-off: the official Ubuntu 24.04 repositories ship Nginx 1.24.x, while the Ondrej PPA provides Nginx 1.27.x which includes HTTP/3 support. For PHP, the Ondrej PPA allows multiple PHP versions to coexist on the same server, making future upgrades far easier. MariaDB is installed from the official Ubuntu repositories since the version available there is sufficient.

---

## 18.2 The `apt` Package Manager

All installs are performed with `apt` (Advanced Package Tool). Key points to keep in mind:

- Always run `sudo apt update` before installing packages to refresh the local package list.
- Use `apt-cache search <name>` to find available packages and `apt-cache show <name>` to inspect version and dependency information before installing.
- Using `apt` is an administrative task — always prefix with `sudo`.

```bash
# Search for available Nginx packages
sudo apt-cache search nginx

# Inspect a specific package
sudo apt-cache show njs
```

---

## 18.3 Update and Upgrade the System

Before installing anything, refresh the package index and apply any pending upgrades:

```bash
sudo apt update
sudo apt upgrade
```

> During the upgrade, if `openssh-server` is updated and a dialog appears asking what to do with the modified `sshd_config`, select **"keep the local version currently installed"**. The custom SSH configuration applied earlier must not be overwritten.

---

## 18.4 Install Nginx

### Note on the Ondrej Nginx PPA Name (April 2025)

The PPA name changed from `ppa:ondrej/nginx-mainline` to `ppa:ondrej/nginx`. If the old name was already added and `apt update` raises a label-change warning, answer `Y` to accept it, then migrate:

```bash
sudo add-apt-repository --remove ppa:ondrej/nginx-mainline
sudo add-apt-repository ppa:ondrej/nginx
sudo apt update
sudo apt upgrade
```

If the old PPA file is not removed automatically, manually delete it:

```bash
cd /etc/apt/sources.list.d/
ls
# Remove the file with "mainline" in its name
```

### Add the Ondrej Nginx PPA

```bash
sudo add-apt-repository ppa:ondrej/nginx-mainline
```

> Use `ppa:ondrej/nginx` if you are following this guide after April 2025.

### Install Nginx with Required Modules

The install command includes Nginx itself plus four additional modules needed for caching, custom headers, and Brotli compression:

```bash
sudo apt install nginx \
  libnginx-mod-http-cache-purge \
  libnginx-mod-http-headers-more-filter \
  libnginx-mod-http-brotli-filter \
  libnginx-mod-http-brotli-static
```

| Module | Purpose |
|---|---|
| `libnginx-mod-http-cache-purge` | Allows selective purging of cached content |
| `libnginx-mod-http-headers-more-filter` | Add, modify, or remove HTTP request/response headers |
| `libnginx-mod-http-brotli-filter` | On-the-fly Brotli compression for responses |
| `libnginx-mod-http-brotli-static` | Serves pre-compressed `.br` static files |

### Fix the Post-Install Failure Caused by Disabled IPv6

After installation, Nginx will immediately fail to start. This is expected — IPv6 was disabled in Chapter 15, but the default Nginx vhost includes a `listen [::]:80` directive that tries to open an IPv6 socket:

```bash
sudo systemctl status nginx
# Active: failed
# nginx: [emerg] socket() [::]:80 failed (97: Address family not supported by protocol)
```

Use the long-form status flag to see the full untruncated error:

```bash
sudo systemctl status nginx --no-pager -l
```

Open the default vhost configuration and comment out the IPv6 listen line:

```bash
sudo nano /etc/nginx/sites-available/default
```

Find this line:

```nginx
listen [::]:80 default_server;
```

Comment it out by adding a `#` at the start:

```nginx
# listen [::]:80 default_server;
```

Save and exit, then start Nginx:

```bash
sudo systemctl start nginx
```

### Verify Nginx is Running and Enabled

```bash
sudo systemctl status nginx --no-pager -l
sudo systemctl is-enabled nginx
```

The status output should show `Active: active (running)` with a green dot, and `is-enabled` should return `enabled`.

### Confirm Nginx Version and Test the Default Page

```bash
nginx -v
# nginx version: nginx/1.27.0
```

Send an HTTP HEAD request to verify the server is responding:

```bash
curl -I http://<server_ip>/
# HTTP/1.1 200 OK
# Server: nginx/1.27.0
```

The default web root is `/var/www/html/`, which at this point contains `index.nginx-debian.html` — the default "Welcome to nginx!" page. Navigating to the server's IP in a browser confirms Nginx is serving correctly.

---

## 18.5 Install MariaDB

MariaDB is installed from the official Ubuntu repositories — no PPA needed.

```bash
sudo apt update
sudo apt install mariadb-server
```

### Verify MariaDB is Running and Enabled at Boot

```bash
sudo systemctl status mariadb
sudo systemctl status mariadb --no-pager -l
sudo systemctl is-enabled mariadb
```

The status output should show `Active: active (running)` and the version installed is MariaDB 10.11.8. The `is-enabled` check should return `enabled`, confirming it will start automatically on reboot.

---

## 18.6 Install PHP 8.3

### How the Stack Fits Together (PHP-FPM)

PHP runs as a separate service called **PHP-FPM** (FastCGI Process Manager). When a client makes a request, Nginx receives it, serves any static content directly, and passes PHP requests through the FastCGI protocol to the PHP-FPM process, which queries MariaDB as needed and returns the result to Nginx.

```
Client → Nginx → FastCGI → PHP-FPM → MariaDB
              ↕
         Static content
```

### Add the Ondrej PHP PPA

PHP 8.3 is the default version for Ubuntu 24.04 and is available from both official repositories and the Ondrej PPA. The Ondrej PPA is used here because it makes multiple PHP versions co-installable, which is valuable for future migrations.

```bash
sudo add-apt-repository ppa:ondrej/php
```

### Install PHP 8.3 with All Required Extensions

The brace expansion syntax `php8.3-{...}` installs all listed extensions in a single command:

```bash
sudo apt install php8.3-{fpm,gd,mbstring,mysql,xml,xmlrpc,opcache,cli,zip,soap,intl,bcmath,curl,imagick,ssh2}
```

| Extension | Purpose |
|---|---|
| `fpm` | FastCGI Process Manager — the PHP process server |
| `gd` | Image processing (GD library) |
| `mbstring` | Multibyte string handling (required for WordPress) |
| `mysql` | MariaDB/MySQL database connectivity |
| `xml` | XML parsing support |
| `xmlrpc` | XML-RPC protocol support |
| `opcache` | Bytecode caching for significant performance gains |
| `cli` | PHP command-line interface |
| `zip` | ZIP archive handling |
| `soap` | SOAP web service support |
| `intl` | Internationalisation support |
| `bcmath` | Arbitrary precision mathematics |
| `curl` | HTTP client for external requests |
| `imagick` | ImageMagick bindings for advanced image processing |
| `ssh2` | SSH2 protocol support |

### Verify PHP-FPM is Running and Enabled

```bash
sudo systemctl status php8.3-fpm
sudo systemctl is-enabled php8.3-fpm
```

The status output should show `Active: active (running)` with the FPM master process and two worker processes in the `www` pool. The `is-enabled` check should return `enabled`.

```bash
php -v
```

---

## 18.7 Summary

All three components of the hosting stack are now installed and running:

| Service | Version | Status | Auto-start |
|---|---|---|---|
| Nginx | 1.27.0 | active (running) | enabled |
| MariaDB | 10.11.8 | active (running) | enabled |
| PHP-FPM | 8.3.x | active (running) | enabled |

The next steps involve configuring Nginx properly with a server block for the domain, securing MariaDB, and wiring PHP-FPM into the Nginx configuration.

---

| | | |
|:---|:---:|---:|
| [← Chapter 17 — DNS & Cloudflare Setup](./17-dns-cloudflare-domain-setup.md) | [↑ Top](#chapter-18--installing-the-hosting-stack-nginx-mariadb--php-83) | [Chapter 19 — Server Mail with msmtp →](./19-server-mail-msmtp.md) |
