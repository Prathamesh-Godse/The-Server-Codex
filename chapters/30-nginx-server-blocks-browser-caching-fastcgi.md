[← Back to Index](../index.md)

---

# Chapter 30 — Nginx Server Blocks, Browser Caching & FastCGI Optimisation

*Creating and enabling an Nginx server block (virtual host) for the project site — covering server block theory, a live demonstration of reload vs restart, building the browser caching and FastCGI include files, and wiring everything together.*

---

## Contents

- [30.1 Part 1 — Understanding Nginx Server Blocks](#301-part-1--understanding-nginx-server-blocks)
  - [What is a Server Block?](#what-is-a-server-block)
  - [How sites-available and sites-enabled Work](#how-sites-available-and-sites-enabled-work)
  - [Server Block Configuration Workflow](#server-block-configuration-workflow)
  - [Reload vs Restart — Why Reload is Always Preferred](#reload-vs-restart--why-reload-is-always-preferred)
- [30.2 Part 2 — A Real Configuration Error and Its Consequences](#302-part-2--a-real-configuration-error-and-its-consequences)
- [30.3 Part 3 — Creating the Include Files](#303-part-3--creating-the-include-files)
  - [browser_caching.conf](#browser_cachingconf)
  - [fastcgi_optimize.conf](#fastcgi_optimizeconf)
- [30.4 Part 4 — Creating the Server Block](#304-part-4--creating-the-server-block)
  - [Step 1 — Verify DNS is Resolving](#step-1--verify-dns-is-resolving)
  - [Step 2 — Review the Default Server Block](#step-2--review-the-default-server-block)
  - [Step 3 — Verify the Document Root is in Place](#step-3--verify-the-document-root-is-in-place)
  - [Step 4 — Create the Server Block File](#step-4--create-the-server-block-file)
  - [Directive-by-Directive Breakdown](#directive-by-directive-breakdown)
- [30.5 Part 5 — Enable the Server Block with a Symlink](#305-part-5--enable-the-server-block-with-a-symlink)
- [30.6 Part 6 — Test and Reload](#306-part-6--test-and-reload)
- [30.7 Summary](#307-summary)

---

## 30.1 Part 1 — Understanding Nginx Server Blocks

### What is a Server Block?

An Nginx server block is the equivalent of an Apache virtual host file. Each server block defines how Nginx should respond to requests for a specific domain or IP address. Multiple server blocks can coexist on a single server, allowing many sites to be hosted simultaneously — each with its own document root, logging, PHP configuration, and caching rules.

The terms **server context** and **server block** refer to the same thing. Both terms are used in the Nginx documentation and community.

### How `sites-available` and `sites-enabled` Work

Nginx on Debian/Ubuntu follows a two-directory pattern:

- **`/etc/nginx/sites-available/`** — stores all server block configuration files, whether active or not. This is where files are created and edited.
- **`/etc/nginx/sites-enabled/`** — contains symbolic links pointing to files in `sites-available`. Nginx only loads the files that have a symlink here.

This separation allows a site's configuration to exist without being active — useful when taking a site temporarily offline without deleting its configuration.

The connection between the two directories and Nginx's main configuration is visible inside `nginx.conf` in the `http {}` block:

```nginx
##
# Virtual Host Configs
##
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

The default installation ships with a single default server block in `sites-available/default`, with a corresponding symlink in `sites-enabled/` pointing back to it:

```
sites-available/
└── default          ← the actual file

sites-enabled/
└── default -> /etc/nginx/sites-available/default   ← symlink
```

### Server Block Configuration Workflow

For every site added to the server, the steps are always the same:

1. Configure DNS — point the domain's A record to the server IP.
2. Create the server block file in `sites-available/`.
3. Create a symlink in `sites-enabled/` pointing to it.
4. Test the configuration with `ngt` (`sudo nginx -t`).
5. Reload Nginx with `ngr` (`sudo systemctl reload nginx`).
6. Install the site (WordPress, static files, etc.).

### Reload vs Restart — Why Reload is Always Preferred

**`sudo systemctl reload nginx`** — Sends a signal to the master process telling it to gracefully re-read its configuration. Worker processes finish serving their current requests, then reload. At no point does Nginx stop accepting new connections. **If the configuration contains a syntax error, the reload fails but Nginx continues running with the last known good configuration.** The site stays up.

**`sudo systemctl restart nginx`** — Stops Nginx entirely and starts it fresh. If the configuration has a syntax error, the start fails and Nginx remains stopped. **The site goes down.**

The practical rule: **always run `sudo nginx -t` before any reload or restart**, and always prefer `reload` over `restart` when applying configuration changes to a running production server.

---

## 30.2 Part 2 — A Real Configuration Error and Its Consequences

This sequence of events happened live and demonstrates exactly why the reload vs restart distinction matters in practice.

**The error** — while reviewing `nginx.conf`, a leading semicolon was accidentally left on the `user` directive:

```nginx
;user www-data;   ← WRONG — semicolons are not comment characters in Nginx
user www-data;    ← correct
```

Nginx uses `#` for comments. A `;` at the start of a line is a syntax error, not a comment.

**Step 1 — `ngt` catches it immediately:**

```bash
ngt
```

```
2024/07/17 20:36:15 [emerg] 66925#66925: unexpected ";" in /etc/nginx/nginx.conf:1
nginx: configuration file /etc/nginx/nginx.conf test failed
```

`nginx -t` caught the error before anything was applied. Nginx was still running normally.

**Step 2 — reload fails safely:**

```bash
sudo systemctl reload nginx
# Job for nginx.service failed.
```

The reload attempted to apply the broken config, failed, and left Nginx running on the previous good configuration. The site remained up throughout.

**Step 3 — restart takes Nginx down:**

```bash
sudo systemctl restart nginx
# Job for nginx.service failed because the control process exited with error code.
```

Unlike reload, restart actually stopped Nginx first. When the new broken config failed to start, Nginx had nothing to fall back to. The site went down.

**Step 4 — fix the error and start Nginx:**

```bash
sudo nano nginx.conf
# Fix: remove the leading semicolon from "user www-data;"
sudo systemctl start nginx
sudo systemctl status nginx
# Active: active (running) since Wed 2024-07-17 20:38:59 CEST; 7s ago
```

**The lesson: always use `ngt` before `ngr`, and never use restart on a live production server when reload will do.**

---

## 30.3 Part 3 — Creating the Include Files

Before building the server block, two include files are created in `/etc/nginx/includes/`. These will be referenced from within the server block and applied to all sites that include them.

### `browser_caching.conf`

```bash
cd /etc/nginx/includes/
sudo nano browser_caching.conf
```

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

`location ~* \.(ext)$` — case-insensitive regex match on file extensions. The `~*` operator makes it case-insensitive so `.JPG` and `.jpg` are both matched.

`expires 365d` / `expires 30d` — sets the `Expires` response header, telling the browser how long it can serve the cached version without re-requesting from the server. Images and downloads are cached for a year; JS, CSS, and fonts for 30 days (more likely to change between site updates).

`add_header Cache-Control "public, no-transform"` — `public` allows the response to be cached by both the browser and any intermediate CDN/proxy caches. `no-transform` prevents proxies from modifying the content.

`access_log off` — disables access logging for static assets. These files are requested thousands of times per day and logging every hit would inflate the access log with noise that is rarely useful for debugging.

---

### `fastcgi_optimize.conf`

```bash
sudo nano fastcgi_optimize.conf
```

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

**What each directive does:**

`fastcgi_connect_timeout 60` — maximum time to wait when connecting to the PHP-FPM socket.

`fastcgi_send_timeout 180` / `fastcgi_read_timeout 180` — how long Nginx waits for PHP-FPM to accept data and return a response. 180 seconds covers long-running WordPress operations like bulk plugin updates or import jobs.

`fastcgi_buffer_size 512k` — size of the buffer for reading the first part of the PHP-FPM response (typically the response headers). A larger buffer means fewer round-trips to read the response.

`fastcgi_buffers 512 16k` — total number and size of buffers for reading the PHP-FPM response body. `512 × 16k = 8MB` of buffer space, meaning responses up to 8MB are served entirely from memory without hitting disk.

`fastcgi_busy_buffers_size 1m` — the amount of buffer that can be actively sending data to the client while the rest is still being read from PHP-FPM.

`fastcgi_temp_file_write_size 4m` / `fastcgi_max_temp_file_size 4m` — controls how PHP-FPM responses that overflow the buffers are written to a temporary file on disk.

`fastcgi_intercept_errors on` — allows Nginx to intercept error responses from PHP-FPM (e.g. a PHP 500 error) and handle them with Nginx's own `error_page` directives.

---

## 30.4 Part 4 — Creating the Server Block

### Step 1 — Verify DNS is Resolving

Before creating the server block, confirm that both the bare domain and the `www` subdomain resolve to the correct server IP:

```bash
ping -c2 expertwp.help
ping -c2 www.expertwp.help
```

Both should resolve to the server's IP address with 0% packet loss. DNS must be live before testing the server block in a browser.

### Step 2 — Review the Default Server Block

```bash
cd /etc/nginx/sites-available/
ls
sudo nano default
```

The default file is examined as a reference. It should remain untouched and its symlink left in place — it handles requests for the server's IP directly and is useful as a catch-all fallback.

### Step 3 — Verify the Document Root is in Place

```bash
tree /var/www
ls /var/www/expertwp.help/public_html/
```

`tree` confirms the expected structure:

```
/var/www
├── expertwp.help
│   └── public_html
└── html
    └── index.nginx-debian.html
```

The `public_html/` directory is empty — ready for WordPress files.

### Step 4 — Create the Server Block File

```bash
sudo nano expertwp.help.conf
```

The complete server block for the site:

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

### Directive-by-Directive Breakdown

**`listen 80;`**
Tells Nginx to listen for HTTP connections on port 80. HTTPS (port 443) is added later after SSL certificates are installed.

**`server_name expertwp.help www.expertwp.help;`**
Defines which domain names this server block responds to. Nginx uses the `Host` header of incoming requests to determine which server block to route to.

**`root /var/www/expertwp.help/public_html;`**
Sets the document root — the directory on disk that Nginx maps to the web root. All file requests are resolved relative to this path.

**`index index.php;`**
The file Nginx looks for when a request comes in for a directory. Set to `index.php` because WordPress's entry point is `index.php`.

**`location / { try_files $uri $uri/ /index.php$is_args$args; }`**
The standard WordPress permalink configuration — `try_files` checks for a file, then a directory, and if neither exists falls back to `index.php` with the original query string. This allows WordPress to handle URLs like `/category/my-post/` that don't correspond to physical files on disk.

**`location ~ \.php$ { ... }`**
Passes PHP file requests to PHP-FPM via the Unix socket. `include snippets/fastcgi-php.conf;` handles setting the `SCRIPT_FILENAME` and other FastCGI parameters. `fastcgi_pass unix:/run/php/php8.3-fpm.sock;` specifies the PHP-FPM socket for PHP 8.3. `include /etc/nginx/includes/fastcgi_optimize.conf;` pulls in the buffer and timeout tuning.

**`include /etc/nginx/includes/browser_caching.conf;`**
Pulls in all four browser caching location blocks from the include file.

**`access_log ... combined buffer=256k flush=60m;`**
Site-specific access log. `buffer=256k` keeps 256KB of log data in memory before writing to disk, and `flush=60m` ensures the buffer is written at least every 60 minutes — both reduce disk I/O on high-traffic sites. The log filename embeds the domain name for clarity.

**`error_log /var/log/nginx/error.log.expertwp.help.log;`**
Site-specific error log. Keeps error output separate from the global Nginx error log and from other sites on the server.

Save and exit nano (`Ctrl+X`, `Y`, `Enter`).

---

## 30.5 Part 5 — Enable the Server Block with a Symlink

Navigate back to the Nginx root and verify both directories before creating the symlink:

```bash
cd /etc/nginx/
ls sites-*
```

Create the symbolic link to enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/expertwp.help.conf /etc/nginx/sites-enabled/expertwp.help.conf
```

`ln -s` creates a symbolic link. The first argument is the **target** (the actual file in `sites-available`), the second is the **link name** (where it appears in `sites-enabled`). Using the full absolute path for the target is important — a relative path would break if the symlink were ever moved.

Verify the symlink was created correctly:

```bash
ls sites-*
```

Expected output:

```
sites-available:
default  expertwp.help.conf

sites-enabled:
default -> /etc/nginx/sites-available/default
expertwp.help.conf -> /etc/nginx/sites-available/expertwp.help.conf
```

---

## 30.6 Part 6 — Test and Reload

```bash
ngt
```

A successful test returns:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test successful
```

Only after a clean test result, reload Nginx:

```bash
ngr
```

Nginx gracefully applies the new server block without interrupting existing connections.

Verify the document root is still empty and ready for site installation:

```bash
ls /var/www/expertwp.help/public_html/
```

An empty result confirms the server block is active and configured, waiting for WordPress to be installed.

---

## 30.7 Summary

| Task | Command / File |
|------|---------------|
| Browser caching rules | `/etc/nginx/includes/browser_caching.conf` |
| FastCGI optimisation | `/etc/nginx/includes/fastcgi_optimize.conf` |
| Server block file | `/etc/nginx/sites-available/expertwp.help.conf` |
| Enable site | `sudo ln -s /etc/nginx/sites-available/expertwp.help.conf /etc/nginx/sites-enabled/expertwp.help.conf` |
| Test config | `ngt` → `sudo nginx -t` |
| Apply config | `ngr` → `sudo systemctl reload nginx` |

> **Automation note:** In a bash script, the symlink can be created with:
> ```bash
> sudo ln -s /etc/nginx/sites-available/${DOMAIN}.conf /etc/nginx/sites-enabled/${DOMAIN}.conf
> ```
> The test and reload should always be chained conditionally — only reload if `nginx -t` succeeds:
> ```bash
> sudo nginx -t && sudo systemctl reload nginx
> ```

---

| | | |
|:---|:---:|---:|
| [← Chapter 29 — Web Root Directory Structure](./29-web-root-directory-structure.md) | [↑ Top](#chapter-30--nginx-server-blocks-browser-caching--fastcgi-optimisation) | [↑ Back to Index](../index.md) |
