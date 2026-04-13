[← Back to Index](../index.md)

---

# Chapter 34 — WordPress Hardening: Introduction & Log File Inspection

*An overview of the full WordPress hardening process and the habit of checking logs before and after every configuration change — including what attack traffic looks like in the access log.*

---

## Contents

- [34.1 Overview — Hardening Areas](#341-overview--hardening-areas)
- [34.2 Log File Inspection](#342-log-file-inspection)
  - [Log File Locations](#log-file-locations)
  - [Navigating the Nginx Log Directory](#navigating-the-nginx-log-directory)
  - [Reading Log Files](#reading-log-files)
  - [What to Look For in the Error Log](#what-to-look-for-in-the-error-log)
  - [What to Look For in the Access Log](#what-to-look-for-in-the-access-log)
- [34.3 What Comes Next](#343-what-comes-next)

---

## 34.1 Overview — Hardening Areas

Hardening a WordPress installation is the most critical phase of deploying a production web server. A default WordPress install exposes several well-known attack surfaces — the admin login endpoint, the XML-RPC interface, writable directories, overly permissive PHP functions, and more.

> **Note:** Some hardening steps — particularly around PHP function restrictions and filesystem permissions — can interfere with plugin functionality. Always check the logs after applying changes and test site behaviour before considering the configuration final.

The following areas are addressed across this and subsequent chapters:

- Isolating sites using dedicated PHP-FPM pools
- Filesystem ownership and permissions (standard and hardened)
- `open_basedir` restriction via PHP pool configuration
- Disabling dangerous PHP functions
- SSL/TLS certificate provisioning and configuration
- HTTP security response headers
- Nginx security directives (WordPress-specific WAF ruleset)
- DDoS mitigation and rate limiting (`wp-login.php`, `xmlrpc.php`)
- Hotlinking protection
- Disabling file edits from within WordPress (`DISALLOW_FILE_MODS`)
- Database privilege hardening
- WP REST API exposure control

---

## 34.2 Log File Inspection

Before making any changes to the server, the first habit to build is **checking the logs**. Logs are the primary diagnostic tool — both for catching problems introduced by hardening steps and for identifying attacks already in progress.

### Log File Locations

| Service | Log Directory |
|---------|--------------|
| Nginx | `/var/log/nginx/` |
| PHP-FPM | `/var/log/` |

### Navigating the Nginx Log Directory

```bash
cd /var/log/nginx
ls
```

The Nginx log directory contains per-site access and error logs. When Nginx is configured with separate `access_log` and `error_log` directives per server block (as in this setup), each site gets its own named log files. Rotated logs are compressed with `.gz` and suffixed with an incrementing number.

A typical log listing:

```
access.log                        error.log
access.log.1                      error.log.1
access.log.2.gz                   error.log.2.gz
access.log.expertwp.help.log      error.log.expertwp.help.log
access.log.expertwp.help.log.1    error.log.expertwp.help.log.1
access.log.expertwp.help.log.2.gz error.log.expertwp.help.log.2.gz
```

The site-specific logs follow the naming pattern defined in the Nginx server block configuration:

```nginx
access_log /var/log/nginx/access.log.example.com.log;
error_log  /var/log/nginx/error.log.example.com.log;
```

Log files are owned by `www-data` with group `adm`, and carry `-rw-r-----` permissions — this is why `sudo` is required to read them.

### Reading Log Files

To print a log file to the terminal in one shot:

```bash
sudo cat error.log.expertwp.help.log
```

Use this for small or older log files. The output is not paginated — it prints everything at once.

To read a large log file with scroll support:

```bash
sudo less access.log.expertwp.help.log
```

`less` allows scrolling up/down with the arrow keys and searching with `/`. Press `q` to quit. This is the preferred command for large active log files.

### What to Look For in the Error Log

A clean error log should be sparse. The most common benign entry is a missing `favicon.ico`:

```
2024/07/23 05:07:41 [error] 72453#72453: *9158 open() "/var/www/expertwp.help/public_html/favicon.ico" failed (2: No such file or directory), client: 128.1.43.38, server: expertwp.help, request: "GET /favicon.ico HTTP/1.1"
```

This can be silenced entirely by adding the following to the Nginx server block:

```nginx
location = /favicon.ico {
    access_log off;
    log_not_found off;
}
```

This directive is included in the Nginx security directives file documented later in this section.

### What to Look For in the Access Log

The access log is where attack traffic becomes visible. The example below — captured from a live server — shows a brute-force attack against `xmlrpc.php`:

```
146.70.192.182 - - [23/Jul/2024:17:42:16 +0200] "POST /xmlrpc.php HTTP/1.1" 200 413 "-" "Mozilla/5.0 ..."
146.70.192.182 - - [23/Jul/2024:17:42:16 +0200] "POST /xmlrpc.php HTTP/1.1" 200 413 "-" "Mozilla/5.0 ..."
146.70.192.182 - - [23/Jul/2024:17:42:17 +0200] "POST /xmlrpc.php HTTP/1.1" 200 413 "-" "Mozilla/5.0 ..."
```

A single IP making dozens of `POST` requests to `xmlrpc.php` within seconds is a classic credential-stuffing or brute-force attack. The server returned HTTP `200` to every one of these — meaning WordPress processed them. This is exactly the behaviour that rate limiting and the Nginx WAF ruleset will shut down.

The access log entry format is:

```
[client_ip] - [user] [timestamp] "[method] [uri] [protocol]" [status] [bytes] "[referer]" "[user_agent]"
```

---

## 34.3 What Comes Next

The following chapters build the full hardening stack on top of this foundation, in order:

1. **PHP-FPM Pool Isolation** (Chapter 35) — each site runs as its own system user with its own socket
2. **Ownership and Permissions** — filesystem hardening for the WordPress document root
3. **`open_basedir`** — restricting PHP file access to the site's own directories
4. **Disabled PHP Functions** — removing shell execution and other dangerous functions from the pool
5. **SSL/TLS Configuration** — Let's Encrypt certificates, TLS 1.2/1.3, HSTS, HTTP/3 + QUIC
6. **HTTP Response Headers** — `X-Frame-Options`, `X-Content-Type-Options`, `Permissions-Policy`, etc.
7. **Nginx Security Directives** — blocking access to sensitive WordPress paths and files
8. **Rate Limiting** — protecting `wp-login.php` and `xmlrpc.php` from brute-force traffic
9. **Database Privilege Hardening** — restricting the WordPress DB user to only required privileges

---

| | | |
|:---|:---:|---:|
| [← Chapter 33 — WordPress Config, Deployment & Setup](./33-wordpress-config-deployment-setup.md) | [↑ Top](#chapter-34--wordpress-hardening-introduction--log-file-inspection) | [Chapter 35 — PHP-FPM Pool Isolation →](./35-php-fpm-pool-isolation.md) |
