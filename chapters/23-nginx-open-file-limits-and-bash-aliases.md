[← Back to Index](../index.md)

---

# Chapter 23 — Nginx Open File Limits & Bash Aliases

*Diagnosing the current open file limit for the Nginx worker process, raising it via `worker_rlimit_nofile`, verifying the new limit took effect, and setting up shell aliases to speed up repetitive server management tasks.*

---

## Contents

- [23.1 Understanding the "Too Many Open Files" Error](#231-understanding-the-too-many-open-files-error)
- [23.2 Checking the Current Open File Limit](#232-checking-the-current-open-file-limit)
- [23.3 Raising the Open File Limit in nginx.conf](#233-raising-the-open-file-limit-in-nginxconf)
- [23.4 Testing and Reloading Nginx](#234-testing-and-reloading-nginx)
- [23.5 Verifying the New Open File Limit](#235-verifying-the-new-open-file-limit)
- [23.6 Setting Up Bash Aliases](#236-setting-up-bash-aliases)
  - [Creating the Aliases File](#creating-the-aliases-file)
  - [Activating the Aliases](#activating-the-aliases)
- [23.7 Summary](#237-summary)

---

## 23.1 Understanding the "Too Many Open Files" Error

Each connection Nginx handles involves at least one open file descriptor — the socket itself, plus any static files being served, cached objects, log file handles, and upstream connections. When the number of simultaneously open file descriptors reaches the process limit set by the OS, Nginx starts returning errors.

The workflow to fix this is:

1. Identify the Nginx worker process ID (PID)
2. Inspect the current open file limit for that PID via `/proc/<PID>/limits`
3. Raise the limit using `worker_rlimit_nofile` in `nginx.conf`
4. Test and reload Nginx
5. Re-identify the new worker PID and confirm the limit changed

---

## 23.2 Checking the Current Open File Limit

First, find the PID of the Nginx worker process running as `www-data`:

```bash
ps aux | grep www-data
```

The output lists all processes owned by `www-data`. The relevant line will say `nginx: worker process`. Note its PID (the second column).

Then read the resource limits for that PID directly from the kernel:

```bash
cat /proc/<PID>/limits
```

Replace `<PID>` with the actual process ID from the previous step. The output shows a table of soft and hard limits. The key row to check is `Max open files` — in the initial state this was **30000**, set by the `worker_rlimit_nofile 30000` directive already present in `nginx.conf`.

---

## 23.3 Raising the Open File Limit in `nginx.conf`

Open the main Nginx configuration file:

```bash
sudo nano /etc/nginx/nginx.conf
```

The `worker_rlimit_nofile` directive sits in the **main context** (outside all blocks, at the top of the file). Update it from `30000` to `45000`:

```nginx
worker_rlimit_nofile 45000;
```

The full main context with all performance directives in place:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

worker_rlimit_nofile 45000;
worker_priority -10;
timer_resolution 100ms;
pcre_jit on;

events {
    worker_connections 4096;
    accept_mutex on;
    accept_mutex_delay 200ms;
    use epoll;
}
```

`worker_rlimit_nofile` sets the maximum number of open files the Nginx worker processes are allowed to have. Setting it higher than `worker_connections` is important — each active connection can require multiple file descriptors (the socket, a cached file, a log entry). The value `45000` comfortably accommodates the `4096` worker connections plus overhead from static file serving, caching, and logging.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 23.4 Testing and Reloading Nginx

Always validate the configuration syntax before reloading:

```bash
sudo nginx -t
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Then apply the change with a graceful reload:

```bash
sudo systemctl reload nginx
```

---

## 23.5 Verifying the New Open File Limit

Because Nginx was reloaded, the worker process gets a new PID. Find it again:

```bash
ps aux | grep www-data
```

Look for the `nginx: worker process` line and note the new PID. Then confirm the limit has been updated:

```bash
cat /proc/<NEW_PID>/limits
```

The `Max open files` row should now show **45000** for both the soft and hard limits, confirming the change is live.

---

## 23.6 Setting Up Bash Aliases

Repeatedly typing long commands like `sudo nginx -t` or `sudo systemctl reload nginx` during development and tuning gets tedious fast. Bash aliases provide short, memorable shortcuts for these.

### Creating the Aliases File

Bash reads `~/.bash_aliases` automatically if it exists, via a sourcing line in `~/.bashrc`. Any alias defined there is available in every new interactive shell session without any extra setup.

Navigate to the home directory and open `.bash_aliases` in nano. If the file doesn't exist yet, nano creates it as a new file:

```bash
cd
nano .bash_aliases
```

Add the following aliases:

```bash
alias ngt='sudo nginx -t'
alias ngr='sudo systemctl reload nginx'
alias fpmr='sudo systemctl restart php8.3-fpm'
alias ngin='cd /etc/nginx/includes && ls'
alias ngsa='cd /etc/nginx/sites-available/ && ls'
```

| Alias | Expands to | Purpose |
|-------|-----------|---------|
| `ngt` | `sudo nginx -t` | Test Nginx configuration syntax |
| `ngr` | `sudo systemctl reload nginx` | Gracefully reload Nginx |
| `fpmr` | `sudo systemctl restart php8.3-fpm` | Restart PHP-FPM |
| `ngin` | `cd /etc/nginx/includes && ls` | Jump to the includes directory and list |
| `ngsa` | `cd /etc/nginx/sites-available/ && ls` | Jump to sites-available and list |

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Activating the Aliases

Aliases defined in `.bash_aliases` are sourced by `.bashrc` on shell startup. To load them in the current session without logging out, exit the SSH session and reconnect:

```bash
exit
ssh 2404
```

Once back in, the aliases are immediately available. To verify:

```bash
ngt
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

```bash
ngr    # Reloads Nginx silently with no output on success

ngin   # Changes to /etc/nginx/includes/ and lists the files:
       # basic_settings.conf  brotli.conf  buffers.conf  file_handle_cache.conf  gzip.conf  timeouts.conf
```

> **Note on first alias run:** The first time an alias is run after saving `.bash_aliases` (without re-sourcing or reconnecting), it returns `Command 'ngt' not found`. This is expected — the alias file is only loaded at shell startup. Reconnecting via SSH picks up the changes cleanly.

---

## 23.7 Summary

| Task | Command |
|------|---------|
| Find Nginx worker PID | `ps aux \| grep www-data` |
| Check process open file limits | `cat /proc/<PID>/limits` |
| Edit nginx.conf | `sudo nano /etc/nginx/nginx.conf` |
| Test Nginx config | `sudo nginx -t` (or `ngt`) |
| Reload Nginx | `sudo systemctl reload nginx` (or `ngr`) |
| Restart PHP-FPM | `sudo systemctl restart php8.3-fpm` (or `fpmr`) |
| Go to Nginx includes dir | `ngin` |
| Go to sites-available dir | `ngsa` |

`worker_rlimit_nofile` must always be set higher than `worker_connections` and high enough to account for file descriptors beyond just active connections — static files, logs, upstream sockets, and cached assets all consume file descriptors simultaneously.

---

| | | |
|:---|:---:|---:|
| [← Chapter 22 — Wiring Nginx Include Files](./22-nginx-conf-wiring-and-reload.md) | [↑ Top](#chapter-23--nginx-open-file-limits--bash-aliases) | [Chapter 24 — Harden MariaDB →](./24-harden-mariadb.md) |
