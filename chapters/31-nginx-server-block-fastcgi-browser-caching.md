[← Back to Index](../index.md)

---

# Chapter 31 — Nginx Server Block: FastCGI Optimisation & Browser Caching

*Configuring a production-ready Nginx server block for a WordPress site — creating the browser caching and FastCGI include files, building the full server block, enabling it with a symlink, and working through a real path typo error.*

---

## Contents

- [31.1 Request Flow](#311-request-flow)
- [31.2 Step 1 — Review the Existing Nginx Includes Directory](#312-step-1--review-the-existing-nginx-includes-directory)
- [31.3 Step 2 — Create the Browser Caching Include File](#313-step-2--create-the-browser-caching-include-file)
- [31.4 Step 3 — Create the FastCGI Optimisation Include File](#314-step-3--create-the-fastcgi-optimisation-include-file)
- [31.5 Step 4 — Verify Include File Paths](#315-step-4--verify-include-file-paths)
- [31.6 Step 5 — Create the Nginx Server Block](#316-step-5--create-the-nginx-server-block)
- [31.7 Step 6 — Enable the Site by Creating a Symlink](#317-step-6--enable-the-site-by-creating-a-symlink)
- [31.8 Step 7 — Test Nginx Configuration and Reload](#318-step-7--test-nginx-configuration-and-reload)
- [31.9 Troubleshooting — Nginx Syntax Test Failure](#319-troubleshooting--nginx-syntax-test-failure)
- [31.10 Step 8 — Verify the Server Block is Active](#3110-step-8--verify-the-server-block-is-active)
- [31.11 Summary](#3111-summary)

---

## 31.1 Request Flow

```
Client (port 80/443)
        ↕
      Nginx  ──────────────────→  Static Content (served directly)
        ↕
    FastCGI (protocol)
        ↕
    php-fpm
        ↕
    MariaDB
```

Nginx handles all incoming requests. Static assets (images, CSS, JS, fonts, etc.) are served directly by Nginx without touching PHP. Dynamic requests (`.php` files) are proxied to `php-fpm` over the FastCGI protocol, which in turn queries MariaDB as needed.

---

## 31.2 Step 1 — Review the Existing Nginx Includes Directory

Before creating new include files, review the existing contents of `/etc/nginx/includes/` to understand what configuration snippets are already in place:

```bash
cd /etc/nginx/includes/
ls
```

Output:

```
basic_settings.conf  brotli.conf  buffers.conf  file_handle_cache.conf  gzip.conf  timeouts.conf
```

These files were created in Chapters 21–22 and handle global Nginx tuning. The two new files — `browser_caching.conf` and `fastcgi_optimize.conf` — will be added here alongside them.

---

## 31.3 Step 2 — Create the Browser Caching Include File

Browser caching instructs clients and proxies to store static assets locally for a set period, reducing repeat requests to the server and improving load times significantly.

```bash
sudo nano browser_caching.conf
```

### File: `/etc/nginx/includes/browser_caching.conf`

```nginx
location ~* \.(webp|3gp|gif|jpg|jpeg|png|ico|wmv|avi|asf|asx|mpg|mpeg|mp4|pls|mp3|mid|wav|swf|flv|exe|zip|tar|rar|gz|tgz|bz2|uha|7z|doc|docx|xls|xlsx|pdf|iso)$ {
    expires 365d;
    add_header Cache-Control "public, no-transform";
    access_log off;
}

location ~* \.(js)$ {
    expires 30d;
    add_header Cache-Control "public, no-transform";
    access_log off;
}

location ~* \.(css)$ {
    expires 30d;
    add_header Cache-Control "public, no-transform";
    access_log off;
}

location ~* \.(eot|svg|ttf|woff|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, no-transform";
    access_log off;
}
```

**What each directive does:**

`location ~* \.(ext)$` — Case-insensitive regex match on file extensions.

`expires 365d` / `expires 30d` — Sets the `Expires` response header. Media and binary files are cached for a full year since they rarely change. JavaScript, CSS, and font files are cached for 30 days.

`add_header Cache-Control "public, no-transform"` — `public` allows shared caches (CDNs, proxies) to store the asset. `no-transform` prevents intermediaries from altering the content (e.g. re-encoding images).

`access_log off` — Suppresses access log entries for static asset requests, keeping log files focused on meaningful traffic.

> **Why separate blocks?** Each file type category has a different update frequency. Binary assets like images and PDFs are deployed once and don't change, justifying a 1-year cache. CSS and JS are updated more frequently during development, so a 30-day TTL provides a reasonable balance.

Save and exit nano (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 31.4 Step 3 — Create the FastCGI Optimisation Include File

The default Nginx FastCGI settings are conservative. For a WordPress site under real traffic, the buffer sizes and timeouts need to be tuned to prevent gateway timeouts and reduce disk I/O caused by insufficient buffer allocation.

```bash
sudo nano fastcgi_optimize.conf
```

### File: `/etc/nginx/includes/fastcgi_optimize.conf`

```nginx
fastcgi_connect_timeout 60;
fastcgi_send_timeout 180;
fastcgi_read_timeout 180;
fastcgi_buffer_size 512k;
fastcgi_buffers 512 16k;
fastcgi_busy_buffers_size 1m;
fastcgi_temp_file_write_size 4m;
fastcgi_max_temp_file_size 4m;
fastcgi_intercept_errors on;
```

| Directive | Value | Purpose |
|-----------|-------|---------|
| `fastcgi_connect_timeout` | `60` | Seconds to wait for a connection to php-fpm before giving up |
| `fastcgi_send_timeout` | `180` | Seconds to wait between successive write operations to php-fpm |
| `fastcgi_read_timeout` | `180` | Seconds to wait for php-fpm to send back a response — critical for slow PHP operations like large imports |
| `fastcgi_buffer_size` | `512k` | Size of the buffer holding the first part of the php-fpm response (headers) |
| `fastcgi_buffers` | `512 16k` | 512 buffers of 16KB each = 8MB total buffer pool for response body |
| `fastcgi_busy_buffers_size` | `1m` | Maximum buffer size that can be in use simultaneously while sending a response to the client |
| `fastcgi_temp_file_write_size` | `4m` | Maximum chunk size written to a temp file when the response overflows buffers |
| `fastcgi_max_temp_file_size` | `4m` | Maximum size of a temp file used when buffering a response |
| `fastcgi_intercept_errors` | `on` | Allows Nginx to intercept and serve custom error pages for php-fpm error codes |

> **Why larger buffers matter:** When php-fpm returns a large response and Nginx's buffer is too small, Nginx writes the overflow to a temp file on disk. This adds I/O latency and can slow down every request. Larger buffers keep responses in memory.

Save and exit nano.

---

## 31.5 Step 4 — Verify Include File Paths

Before referencing these files from a server block, confirm their absolute paths resolve correctly:

```bash
readlink -f browser_caching.conf
# /etc/nginx/includes/browser_caching.conf

readlink -f fastcgi_optimize.conf
# /etc/nginx/includes/fastcgi_optimize.conf
```

> **Note:** `readlink -f` resolves and prints the absolute canonical path of a file. Using absolute paths in `include` directives avoids any ambiguity about which file Nginx loads, regardless of the working directory.

Listing the directory confirms both new files are present:

```bash
ls /etc/nginx/includes/
```

```
basic_settings.conf  browser_caching.conf  brotli.conf  buffers.conf  fastcgi_optimize.conf  file_handle_cache.conf  gzip.conf  timeouts.conf
```

---

## 31.6 Step 5 — Create the Nginx Server Block

Navigate to the `sites-available` directory and open the site's config file for editing:

```bash
cd /etc/nginx/sites-available/
sudo nano expertwp.help.conf
```

### Final File: `/etc/nginx/sites-available/expertwp.help.conf`

```nginx
server {

    listen 80;

    server_name expertwp.help www.expertwp.help;

    root /var/www/expertwp.help/public_html;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        include /etc/nginx/includes/fastcgi_optimize.conf;
    }

    include /etc/nginx/includes/browser_caching.conf;

    access_log /var/log/nginx/access.log.expertwp.help.log combined buffer=256k flush=60m;
    error_log /var/log/nginx/error.log.expertwp.help.log;

}
```

**Directive-by-directive breakdown:**

`listen 80;` — Accept HTTP traffic on port 80. HTTPS (port 443) with SSL will be added in a later step when Let's Encrypt is configured.

`server_name expertwp.help www.expertwp.help;` — Nginx will match incoming requests for either the bare domain or the `www` subdomain to this server block.

`root /var/www/expertwp.help/public_html;` — Document root where WordPress files will live.

`index index.php;` — The default file Nginx looks for when a directory is requested. WordPress's entry point is `index.php`.

`location / { try_files ... }` — WordPress permalink handling. Nginx first tries to serve the URI as a file (`$uri`), then as a directory (`$uri/`), and if neither exists, falls back to `index.php` with the original query string passed through (`$is_args$args`).

`location ~ \.php$ { ... }` — Routes all `.php` requests to php-fpm via the Unix socket. `snippets/fastcgi-php.conf` sets the standard FastCGI parameters, and `fastcgi_optimize.conf` layers the tuned buffer/timeout settings on top.

`include /etc/nginx/includes/browser_caching.conf;` — Pulls in the caching location blocks at the server level so they apply across all locations.

`access_log ... combined buffer=256k flush=60m;` — Buffered access logging. Rather than writing to disk on every request, Nginx accumulates up to 256KB of log data in memory and flushes it at most every 60 minutes.

`error_log ...;` — Per-site error log, separate from the global Nginx error log.

Save and exit nano.

---

## 31.7 Step 6 — Enable the Site by Creating a Symlink

Config files in `sites-available` are just stored there — Nginx only loads configs that are symlinked into `sites-enabled`. Create the symlink:

```bash
sudo ln -s /etc/nginx/sites-available/expertwp.help.conf /etc/nginx/sites-enabled/
```

Verify the symlink was created:

```bash
ls sites-* -l
```

Output:

```
sites-available:
total 8
-rw-r--r-- 1 root root 2414 Jul 10 20:10 default
-rw-r--r-- 1 root root  555 Jul 18 19:37 expertwp.help.conf

sites-enabled:
total 0
lrwxrwxrwx 1 root root 34 Jul 10 18:52 default -> /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root 45 Jul 18 19:37 expertwp.help.conf -> /etc/nginx/sites-available/expertwp.help.conf
```

The `expertwp.help.conf` symlink in `sites-enabled` now points back to the original file in `sites-available`. Any edits to the original are immediately reflected.

---

## 31.8 Step 7 — Test Nginx Configuration and Reload

Always test the Nginx configuration syntax before reloading:

```bash
sudo nginx -t
```

Expected output (success):

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If the test passes, reload Nginx to apply the new configuration without dropping existing connections:

```bash
sudo systemctl reload nginx
```

> **Important:** `reload` gracefully applies the new config. Using `restart` instead would terminate all active connections. Always prefer `reload` for config changes.

---

## 31.9 Troubleshooting — Nginx Syntax Test Failure

During the first syntax test run, the test failed with the following error:

```
2024/07/18 19:49:02 [emerg] 72416#72416: open() "/etc/nginx/include/fastcgi_optimize.conf"
failed (2: No such file or directory) in /etc/nginx/sites-enabled/expertwp.help.conf:16
nginx: configuration file /etc/nginx/nginx.conf test failed
```

**Root cause:** The `include` path on line 16 of the server block had a typo — `/etc/nginx/include/` instead of `/etc/nginx/includes/` (missing the trailing `s`).

```nginx
# Wrong (caused test failure):
include /etc/nginx/include/fastcgi_optimize.conf;

# Correct:
include /etc/nginx/includes/fastcgi_optimize.conf;
```

The file was reopened in nano with line numbers enabled to locate the error quickly:

```bash
sudo nano expertwp.help.conf -l
```

After fixing the typo and saving, the syntax test was run again and passed. Nginx was then reloaded.

> **Workflow:** The correct cycle when modifying Nginx server blocks is always: edit → `sudo nginx -t` → if OK, `sudo systemctl reload nginx` → if test fails, fix and repeat.

---

## 31.10 Step 8 — Verify the Server Block is Active

### Browser Check

Navigating to `http://expertwp.help` in a browser at this stage returns a **403 Forbidden** response from Nginx.

This is expected — the `public_html` directory exists but is empty. There are no files yet for Nginx to serve, so it returns 403 rather than 404 because directory listing is disabled. WordPress hasn't been installed yet.

### curl Check

```bash
curl -I expertwp.help
```

Output:

```
HTTP/1.1 403 Forbidden
Date: Thu, 18 Jul 2024 17:41:36 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 146
Connection: keep-alive
```

The `curl -I` flag sends a HEAD request and prints only the response headers. The `403` from Nginx confirms the server block is live and correctly routing traffic — the domain is resolving, Nginx is responding, and the issue is simply that there's no content in the web root yet.

---

## 31.11 Summary

| File | Location | Purpose |
|------|----------|---------|
| `browser_caching.conf` | `/etc/nginx/includes/` | HTTP caching headers for static assets |
| `fastcgi_optimize.conf` | `/etc/nginx/includes/` | Tuned FastCGI buffer and timeout settings |
| `expertwp.help.conf` | `/etc/nginx/sites-available/` | Full server block for the domain |
| `expertwp.help.conf` symlink | `/etc/nginx/sites-enabled/` | Makes the server block active |

The Nginx configuration is modular by design — the two include files in `/etc/nginx/includes/` can be reused across multiple server blocks by adding a single `include` line, making future site additions straightforward.

---

| | | |
|:---|:---:|---:|
| [← Chapter 30 — Nginx Server Blocks, Browser Caching & FastCGI](./30-nginx-server-blocks-browser-caching-fastcgi.md) | [↑ Top](#chapter-31--nginx-server-block-fastcgi-optimisation--browser-caching) | [Chapter 32 — MariaDB Setup & WordPress Prep →](./32-mariadb-wordpress-installation-prep.md) |
