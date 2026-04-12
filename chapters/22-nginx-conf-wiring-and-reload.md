[← Back to Index](../index.md)

---

# Chapter 22 — Completing the Nginx Configuration: Wiring the Include Files

*Populating the remaining include files, replacing the `nginx.conf` http context with clean `include` directives, and working through two real configuration errors encountered and fixed before the final clean reload.*

> This chapter picks up directly from Chapter 21, inside `/etc/nginx/includes/`.

---

## Contents

- [22.1 Populate timeouts.conf](#221-populate-timeoutsconf)
- [22.2 Populate gzip.conf](#222-populate-gzipconf)
- [22.3 Populate brotli.conf](#223-populate-brotliconf)
- [22.4 Populate file_handle_cache.conf](#224-populate-file_handle_cacheconf)
- [22.5 Verify All Include Files Are in Place](#225-verify-all-include-files-are-in-place)
- [22.6 Rewrite the nginx.conf http Context](#226-rewrite-the-nginxconf-http-context)
- [22.7 Test and Troubleshoot the Configuration](#227-test-and-troubleshoot-the-configuration)
  - [Error 1 — Missing Semicolon on an Include Line](#error-1--missing-semicolon-on-an-include-line)
  - [Error 2 — Typo in the Include Filename](#error-2--typo-in-the-include-filename)
- [22.8 Reload Nginx](#228-reload-nginx)
- [22.9 Final nginx.conf — Verified State](#229-final-nginxconf--verified-state)
- [22.10 Key Troubleshooting Takeaways](#2210-key-troubleshooting-takeaways)

---

## 22.1 Populate `timeouts.conf`

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

Save with `Ctrl+X`, then `Y` to confirm writing the file.

---

## 22.2 Populate `gzip.conf`

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

---

## 22.3 Populate `brotli.conf`

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

---

## 22.4 Populate `file_handle_cache.conf`

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

---

## 22.5 Verify All Include Files Are in Place

Once all six files are written, step back to the Nginx root and confirm the directory looks correct:

```bash
cd ..
ls
ls includes/
```

The `ls includes/` output should show all six files:

```
basic_settings.conf  brotli.conf  buffers.conf  file_handle_cache.conf  gzip.conf  timeouts.conf
```

---

## 22.6 Rewrite the `nginx.conf` http Context

Open `nginx.conf` and replace the existing default http context content with `include` directives. The goal is to strip out all the inline directives and replace each section with a single pointed include.

```bash
sudo nano nginx.conf
```

**What to remove:**
- All directives under `# Basic Settings` (sendfile, tcp_nopush, types_hash_max_size, etc.)
- The SSL comments and any SSL-related directives
- The single `gzip on;` line under `# Gzip Settings` along with all the commented-out gzip lines
- The entire `#mail { ... }` commented section at the bottom of the file
- Under `# Logging Settings`, change `access_log /var/log/nginx/access.log;` to `access_log off;` — per-site logging will be enabled in each server block instead. Leave `error_log` enabled.

**The final `nginx.conf` http context:**

```nginx
http {

        ##
        # Basic Settings
        ##
        include /etc/nginx/includes/basic_settings.conf;

        # Buffer Settings
        include /etc/nginx/includes/buffers.conf;

        # Timeout Settings
        include /etc/nginx/includes/timeouts.conf;

        # File Handle Cache Settings
        include /etc/nginx/includes/file_handle_cache.conf;

        ##
        # Logging Settings
        ##
        access_log off;

        ##
        # Gzip & Brotli Settings
        ##
        include /etc/nginx/includes/gzip.conf;
        include /etc/nginx/includes/brotli.conf;

        ##
        # Virtual Host Configs
        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```

Save the file.

---

## 22.7 Test and Troubleshoot the Configuration

With the `nginx.conf` edited, run the config test immediately:

```bash
sudo nginx -t
```

### Error 1 — Missing Semicolon on an Include Line

The first test produced an error:

```
2024/07/14 19:31:18 [emerg] 42001#42001: invalid number of arguments in "include"
directive in /etc/nginx/nginx.conf:33
nginx: configuration file /etc/nginx/nginx.conf test failed
```

The error points to line 33 of `nginx.conf`. Re-open the file with the `-l` flag to display line numbers, which makes it faster to navigate to the problem line:

```bash
sudo nano nginx.conf -l
```

Inspecting line 33 revealed the `timeouts.conf` include directive was missing its trailing semicolon:

```nginx
# Before (broken):
include /etc/nginx/includes/timeouts.conf

# After (fixed):
include /etc/nginx/includes/timeouts.conf;
```

Save, then re-test:

```bash
sudo nginx -t
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Error 2 — Typo in the Include Filename

A subsequent edit introduced another error — a missing underscore in the filename:

```
2024/07/14 19:34:06 [emerg] 42014#42014: open() "/etc/nginx/includes/basicsettings.conf"
failed (2: No such file or directory) in /etc/nginx/nginx.conf:23
nginx: configuration file /etc/nginx/nginx.conf test failed
```

The error is clear: Nginx is trying to open `basicsettings.conf` but the actual file on disk is `basic_settings.conf`. First verify the correct filename:

```bash
ls /etc/nginx/includes/
```

```
basic_settings.conf  brotli.conf  buffers.conf  file_handle_cache.conf  gzip.conf  timeouts.conf
```

Re-open with line numbers and correct line 23:

```bash
sudo nano nginx.conf -l
```

```nginx
# Before (broken):
include /etc/nginx/includes/basicsettings.conf;

# After (fixed):
include /etc/nginx/includes/basic_settings.conf;
```

Save, then test again:

```bash
sudo nginx -t
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

## 22.8 Reload Nginx

With the test passing cleanly, reload Nginx to apply all configuration changes:

```bash
sudo systemctl reload nginx
```

A silent exit (no output, prompt returns immediately) confirms the reload was successful. Nginx is now running with the hardened and optimised configuration across all contexts.

---

## 22.9 Final `nginx.conf` — Verified State

The complete verified `nginx.conf` at the end of this session:

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

events {
        worker_connections 4096;
        accept_mutex on;
        accept_mutex_delay 200ms;
        use epoll;
}

http {

        ##
        # Basic Settings
        ##
        include /etc/nginx/includes/basic_settings.conf;

        # Buffer Settings
        include /etc/nginx/includes/buffers.conf;

        # Timeout Settings
        include /etc/nginx/includes/timeouts.conf;

        # File Handle Cache Settings
        include /etc/nginx/includes/file_handle_cache.conf;

        ##
        # Logging Settings
        ##
        access_log off;

        ##
        # Gzip & Brotli Settings
        ##
        include /etc/nginx/includes/gzip.conf;
        include /etc/nginx/includes/brotli.conf;

        ##
        # Virtual Host Configs
        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```

---

## 22.10 Key Troubleshooting Takeaways

Two config errors appeared during this session, both caught cleanly by `sudo nginx -t` before Nginx was reloaded.

**Missing semicolon on an `include` directive** — Nginx reported `invalid number of arguments in "include" directive` and pointed to the exact line number. The fix was simply adding the missing `;`.

**Typo in an include filename** — Nginx reported `open() "..." failed (2: No such file or directory)` and again pointed to the exact line. Cross-referencing with `ls /etc/nginx/includes/` confirmed the correct filename.

Both errors highlight why `sudo nginx -t` must always be run before `sudo systemctl reload nginx`. The `-l` flag on nano (`sudo nano nginx.conf -l`) is invaluable — it displays line numbers in the editor, making it instant to jump to the exact line the error message references.

---

| | | |
|:---|:---:|---:|
| [← Chapter 21 — Hardening & Optimising Nginx](./21-nginx-harden-and-optimize.md) | [↑ Top](#chapter-22--completing-the-nginx-configuration-wiring-the-include-files) | [Chapter 23 — Nginx Open File Limits & Bash Aliases →](./23-nginx-open-file-limits-and-bash-aliases.md) |
