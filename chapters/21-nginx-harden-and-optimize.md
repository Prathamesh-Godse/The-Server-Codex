[← Back to Index](../index.md)

---

# Chapter 21 — Hardening & Optimising Nginx

*Tuning directives across the `main`, `events`, and `http` contexts of `nginx.conf`, and splitting the http context into dedicated include files stored in a custom `includes/` directory.*

> **Reference:** The full Nginx directive index is at [http://nginx.org/en/docs/dirindex.html](http://nginx.org/en/docs/dirindex.html) — close to 800 directives in total. This chapter covers the most impactful ones for a WordPress/PHP server.

---

## Contents

- [21.1 Back Up and Open nginx.conf](#211-back-up-and-open-nginxconf)
- [21.2 The include Directive](#212-the-include-directive)
- [21.3 Harden and Optimise the main Context](#213-harden-and-optimise-the-main-context)
- [21.4 Harden and Optimise the events Context](#214-harden-and-optimise-the-events-context)
- [21.5 Create the includes/ Directory and Config Files](#215-create-the-includes-directory-and-config-files)
- [21.6 basic_settings.conf](#216-basic_settingsconf)
- [21.7 buffers.conf](#217-buffersconf)
- [21.8 timeouts.conf](#218-timeoutsconf)
- [21.9 gzip.conf](#219-gzipconf)
- [21.10 brotli.conf](#2110-brotliconf)
- [21.11 file_handle_cache.conf](#2111-file_handle_cacheconf)
- [21.12 Wire the Include Files into nginx.conf](#2112-wire-the-include-files-into-nginxconf)
- [21.13 Test and Reload Nginx](#2113-test-and-reload-nginx)
- [21.14 Verify and Tune the Open File Limit](#2114-verify-and-tune-the-open-file-limit)
- [21.15 Bash Aliases for the Nginx Workflow](#2115-bash-aliases-for-the-nginx-workflow)

---

## 21.1 Back Up and Open `nginx.conf`

Before making any changes, navigate to the nginx directory and create a backup of the main config file.

```bash
cd /etc/nginx/
sudo cp nginx.conf nginx.conf.bak
ls -l
```

The backup appears in the directory as `nginx.conf.bak`. Open the original for editing:

```bash
sudo nano nginx.conf
```

---

## 21.2 The `include` Directive

The `include` directive pulls in another file (or all files matching a wildcard mask) directly into the configuration at the point where it appears. It can be used in any context.

```nginx
# Include a single file
include /etc/nginx/includes/basic_settings.conf;

# Include all .conf files in a directory
include vhosts/*.conf;
```

The included files must contain syntactically valid Nginx directives. This makes it possible to break a large `nginx.conf` into smaller, purpose-specific files.

---

## 21.3 Harden and Optimise the `main` Context

The main (top-level) context of `nginx.conf` controls global process behaviour. The following directives are added directly to the main context — outside of any `events {}` or `http {}` block.

```bash
sudo nano /etc/nginx/nginx.conf
```

Add the following to the main context:

```nginx
worker_rlimit_nofile 30000;
worker_priority -10;
timer_resolution 100ms;
pcre_jit on;
```

**What each directive does:**

`worker_rlimit_nofile 30000;` — Sets the maximum number of open file descriptors that Nginx worker processes are allowed to use. This needs to be at least as large as `worker_connections` multiplied by the number of worker processes, plus overhead for static files and upstream connections.

`worker_priority -10;` — Adjusts the scheduling priority of Nginx worker processes. A value of `-10` elevates Nginx above typical user-space applications without taking priority away from critical system processes.

`timer_resolution 100ms;` — Reduces the frequency of `gettimeofday()` system calls by setting how often Nginx updates its internal clock, reducing unnecessary syscall overhead on busy servers.

`pcre_jit on;` — Enables JIT (Just-In-Time) compilation for PCRE regular expressions used in Nginx location blocks. This can significantly speed up regex matching for servers with many location rules.

After adding these, the top of `nginx.conf` looks like this:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;
worker_rlimit_nofile 30000;
worker_priority -10;
timer_resolution 100ms;
pcre_jit on;
```

---

## 21.4 Harden and Optimise the `events` Context

The `events` context controls how Nginx handles connections at the OS level. Update the existing `events {}` block:

```nginx
events {
    worker_connections 4096;
    accept_mutex on;
    accept_mutex_delay 200ms;
    use epoll;
}
```

**What each directive does:**

`worker_connections 4096;` — Sets the maximum number of simultaneous connections a single worker process can handle, increased from the default of 768.

`accept_mutex on;` — Prevents the "thundering herd" problem where all workers wake up simultaneously to accept a new connection. With the mutex enabled, only one worker at a time accepts new connections.

`accept_mutex_delay 200ms;` — Controls how long a worker process waits before trying to acquire the accept mutex again if another worker holds it.

`use epoll;` — Instructs Nginx to use Linux's `epoll` event model, which is the most efficient connection-handling mechanism on Linux, scaling well for thousands of simultaneous connections.

---

## 21.5 Create the `includes/` Directory and Config Files

Rather than adding all http context directives directly into `nginx.conf`, they are separated into individual include files.

```bash
cd /etc/nginx/
sudo mkdir includes/
ls
```

Navigate into the new directory and create all the include files at once with `touch`:

```bash
cd /etc/nginx/includes/
sudo touch basic_settings.conf buffers.conf timeouts.conf file_handle_cache.conf gzip.conf brotli.conf
ls -l
```

This creates six empty files, each of which will hold a specific group of Nginx directives.

---

## 21.6 `basic_settings.conf`

```bash
sudo nano /etc/nginx/includes/basic_settings.conf
```

```nginx
##
# BASIC SETTINGS
##
charset utf-8;
sendfile on;
sendfile_max_chunk 512k;
tcp_nopush on;
tcp_nodelay on;
server_tokens off;
more_clear_headers 'Server';
more_clear_headers 'X-Powered';
server_name_in_redirect off;
server_names_hash_bucket_size 64;
variables_hash_max_size 2048;
types_hash_max_size 2048;
include /etc/nginx/mime.types;
default_type application/octet-stream;
```

**Key directives explained:**

`sendfile on;` — Enables the kernel-level `sendfile()` syscall to transfer files directly from disk to the network socket without copying through user space. Significant performance gain for static file serving.

`sendfile_max_chunk 512k;` — Limits the amount of data transferred per `sendfile()` call to prevent a single large file transfer from monopolising the worker.

`tcp_nopush on;` — Works with `sendfile` to batch response headers and the beginning of file data into one TCP packet, reducing the number of packets sent.

`tcp_nodelay on;` — Disables Nagle's algorithm, sending data immediately rather than buffering small packets. Important for low-latency responses.

`server_tokens off;` — Removes the Nginx version number from error pages and the `Server` response header, preventing version disclosure.

`more_clear_headers 'Server';` — Clears the `Server` header from all responses entirely. Requires the `ngx_headers_more` module. Prevents fingerprinting the web server software.

`more_clear_headers 'X-Powered';` — Removes the `X-Powered-By` header, which can leak PHP version information.

`default_type application/octet-stream;` — Sets the fallback MIME type for files Nginx cannot identify, triggering a download prompt instead of trying to render unknown content.

---

## 21.7 `buffers.conf`

```bash
sudo nano /etc/nginx/includes/buffers.conf
```

```nginx
##
# BUFFERS
##
client_body_buffer_size 256k;
client_body_in_file_only off;
client_header_buffer_size 64k;
# client_max_body_size - reduce to 8m after setting up the site.
# Large value is to allow theme, plugin or asset uploading.
client_max_body_size 100m;
connection_pool_size 512;
directio 4m;
ignore_invalid_headers on;
large_client_header_buffers 8 64k;
output_buffers 8 256k;
postpone_output 1460;
request_pool_size 32k;
```

**Key directives explained:**

`client_max_body_size 100m;` — Maximum allowed size of the client request body. Set high initially to allow WordPress theme and plugin uploads. **Reduce this to `8m` once the site is configured.**

`directio 4m;` — For files larger than 4MB, Nginx uses direct I/O (bypassing the OS page cache), which prevents large file transfers from flushing smaller, frequently accessed files from the cache.

`ignore_invalid_headers on;` — Drops requests with malformed HTTP headers rather than passing them upstream.

`postpone_output 1460;` — Nginx will wait until there is at least 1460 bytes of response data before sending, aligning output with a typical TCP Maximum Segment Size to reduce small packet overhead.

---

## 21.8 `timeouts.conf`

```bash
sudo nano /etc/nginx/includes/timeouts.conf
```

```nginx
##
# TIMEOUTS
##
keepalive_timeout 5;
keepalive_requests 500;
lingering_time 20s;
lingering_timeout 5s;
keepalive_disable msie6;
reset_timedout_connection on;
send_timeout 15s;
client_header_timeout 8s;
client_body_timeout 10s;
```

**Key directives explained:**

`keepalive_timeout 5;` — How long Nginx keeps an idle keep-alive connection open. A low value frees connections quickly on a busy server.

`keepalive_requests 500;` — Maximum number of requests that can be served over a single keep-alive connection before it is closed.

`reset_timedout_connection on;` — Immediately resets timed-out connections by sending a TCP RST, freeing memory associated with stale connections faster than a graceful close.

`client_header_timeout 8s;` / `client_body_timeout 10s;` — Short values protect against slow-read attacks (Slowloris-style).

---

## 21.9 `gzip.conf`

```bash
sudo nano /etc/nginx/includes/gzip.conf
```

```nginx
##
# GZIP
##
gzip on;
gzip_vary on;
gzip_disable "MSIE [1-6]\.";
gzip_static on;
gzip_min_length 1400;
gzip_buffers 32 8k;
gzip_http_version 1.0;
gzip_comp_level 5;
gzip_proxied any;
gzip_types text/plain text/css text/xml application/javascript application/x-javascript application/xml application/xml+rss application/ecmascript application/json image/svg+xml;
```

> **Note on the `gzip_types` line:** This is a single long directive. In nano it may wrap visually across two lines, but it must remain a single line ending with a semicolon.

**Key directives explained:**

`gzip_vary on;` — Adds a `Vary: Accept-Encoding` header, telling proxies and CDNs to cache both compressed and uncompressed versions of a resource.

`gzip_static on;` — Serves pre-compressed `.gz` files (if they exist alongside the originals) instead of compressing on the fly, saving CPU.

`gzip_min_length 1400;` — Only compress responses larger than 1400 bytes. Compressing tiny responses adds CPU overhead with no real benefit.

`gzip_comp_level 5;` — Compression level 1–9. Level 5 offers a good balance between CPU cost and compression ratio.

---

## 21.10 `brotli.conf`

Brotli is a modern compression algorithm developed by Google that typically achieves better compression ratios than gzip, especially for text content. It requires the `nginx-module-brotli` module installed in Chapter 18.

```bash
sudo nano /etc/nginx/includes/brotli.conf
```

```nginx
##
# BROTLI
##
brotli on;
brotli_comp_level 6;
brotli_static on;
brotli_types application/atom+xml application/javascript application/json application/rss+xml application/vnd.ms-fontobject application/x-font-opentype application/x-font-truetype application/x-font-ttf application/x-javascript application/xhtml+xml application/xml font/eot font/opentype font/otf font/truetype image/svg+xml image/vnd.microsoft.icon image/x-icon image/x-win-bitmap text/css text/javascript text/plain text/xml;
```

`brotli_comp_level 6;` — Level 1–11. Level 6 offers strong compression without excessive CPU usage.

`brotli_static on;` — Like `gzip_static`, serves pre-compressed `.br` files if they exist, eliminating on-the-fly compression overhead.

---

## 21.11 `file_handle_cache.conf`

The open file cache allows Nginx to cache file descriptors and metadata for frequently served files, avoiding repeated system calls to open and stat files on disk.

```bash
sudo nano /etc/nginx/includes/file_handle_cache.conf
```

```nginx
##
# FILE HANDLE CACHE
##
open_file_cache max=50000 inactive=60s;
open_file_cache_valid 120s;
open_file_cache_min_uses 2;
open_file_cache_errors off;
```

`open_file_cache max=50000 inactive=60s;` — Caches up to 50,000 file handles. Files not accessed within 60 seconds are evicted.

`open_file_cache_valid 120s;` — How often Nginx rechecks cached file information to see if files have changed on disk.

`open_file_cache_min_uses 2;` — A file must be accessed at least twice within the inactive period to stay in cache, preventing rarely-accessed files from occupying cache slots.

`open_file_cache_errors off;` — Does not cache file-not-found errors, so files recently created on disk are found immediately.

---

## 21.12 Wire the Include Files into `nginx.conf`

Open `nginx.conf` and replace the existing http context content with `include` directives. Remove the default basic settings directives, the SSL comments, the gzip section, and the mail context block. Change `access_log` to `off` — logging will be configured per site in the server context. Leave `error_log` enabled.

```bash
cd /etc/nginx
sudo nano nginx.conf
```

The final `nginx.conf` http context section:

```nginx
http {

    ##
    # Basic Settings
    ##
    include /etc/nginx/includes/basic_settings.conf;

    ##
    # Logging Settings
    ##
    access_log off;
    error_log /var/log/nginx/error.log;

    ##
    # Gzip Settings
    ##
    include /etc/nginx/includes/gzip.conf;

    # Brotli
    include /etc/nginx/includes/brotli.conf;

    ##
    # Buffer Settings
    ##
    include /etc/nginx/includes/buffers.conf;

    ##
    # Timeout Settings
    ##
    include /etc/nginx/includes/timeouts.conf;

    ##
    # File Handle Cache Settings
    ##
    include /etc/nginx/includes/file_handle_cache.conf;

    ##
    # Virtual Host Configs
    ##
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## 21.13 Test and Reload Nginx

Always test the configuration before reloading:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

A clean test output confirms everything is syntactically correct:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

## 21.14 Verify and Tune the Open File Limit

To verify the current limit is actually being applied, find the Nginx worker process ID and inspect its limits file:

```bash
# Find the nginx worker process ID
ps aux | grep www-data

# View the process limits (replace XXX with the actual PID)
cat /proc/XXX/limits
```

Look for the `Max open files` row. If the value does not reflect what was set in `nginx.conf`, increase `worker_rlimit_nofile` and reload:

```bash
sudo nano /etc/nginx/nginx.conf
# Update: worker_rlimit_nofile 45000;

sudo nginx -t
sudo systemctl reload nginx

# Get the new worker PID and verify again
ps aux | grep www-data
cat /proc/XXX/limits
```

---

## 21.15 Bash Aliases for the Nginx Workflow

To speed up the repetitive test-reload-navigate workflow, add bash aliases in `~/.bash_aliases`:

```bash
cd ~
nano .bash_aliases
```

Add the following:

```bash
alias server_update='sudo apt update && sudo apt upgrade && sudo apt autoremove'
alias ngt='sudo nginx -t'
alias ngr='sudo systemctl reload nginx'
alias fpmr='sudo systemctl restart php8.3-fpm'
alias ngsa='cd /etc/nginx/sites-available/ && ls'
alias ngin='cd /etc/nginx/includes/ && ls'
```

| Alias | Expands to | Purpose |
|-------|-----------|---------|
| `server_update` | `sudo apt update && sudo apt upgrade && sudo apt autoremove` | Full system update in one command |
| `ngt` | `sudo nginx -t` | Test Nginx configuration |
| `ngr` | `sudo systemctl reload nginx` | Reload Nginx (graceful, no dropped connections) |
| `fpmr` | `sudo systemctl restart php8.3-fpm` | Restart PHP-FPM |
| `ngsa` | `cd /etc/nginx/sites-available/ && ls` | Jump to sites-available and list files |
| `ngin` | `cd /etc/nginx/includes/ && ls` | Jump to the includes directory and list files |

To activate the aliases in the current session without logging out:

```bash
su andrew
```

Then test:

```bash
ngt     # should show "syntax is ok" and "test is successful"
ngsa    # should cd to sites-available and list it
ngin    # should cd to includes and list all .conf files
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 20 — Nginx Configuration Files](./20-nginx-configuration-files.md) | [↑ Top](#chapter-21--hardening--optimising-nginx) | [Chapter 22 — Wiring Nginx Include Files →](./22-nginx-conf-wiring-and-reload.md) |
