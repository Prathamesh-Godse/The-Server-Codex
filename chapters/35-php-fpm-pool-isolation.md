[← Back to Index](../index.md)

---

# Chapter 35 — PHP-FPM Pool Isolation

*Creating a dedicated PHP-FPM pool for a WordPress site — its own system user, socket, resource limits, error logging, and Nginx routing — so that a compromise on one site cannot spread to others on the same server.*

---

## Contents

- [35.1 Overview — Why Pools Matter for Security](#351-overview--why-pools-matter-for-security)
- [35.2 Step 1 — Create a Dedicated System User for the Site](#352-step-1--create-a-dedicated-system-user-for-the-site)
- [35.3 Step 2 — Locate and Inspect the PHP-FPM Pool Directory](#353-step-2--locate-and-inspect-the-php-fpm-pool-directory)
- [35.4 Step 3 — Create a New Pool Configuration from the Default](#354-step-3--create-a-new-pool-configuration-from-the-default)
- [35.5 Step 4 — Edit the Pool Configuration File](#355-step-4--edit-the-pool-configuration-file)
  - [4.1 — Pool Name](#41--pool-name)
  - [4.2 — Pool User and Group](#42--pool-user-and-group)
  - [4.3 — Unix Socket Path](#43--unix-socket-path)
  - [4.4 — Resource Limits](#44--resource-limits)
  - [4.5 — PHP Error Logging](#45--php-error-logging)
- [35.6 Step 5 — Create the PHP-FPM Error Log File](#356-step-5--create-the-php-fpm-error-log-file)
- [35.7 Step 6 — Reload PHP-FPM and Verify the Socket](#357-step-6--reload-php-fpm-and-verify-the-socket)
- [35.8 Step 7 — Update the Nginx Server Block to Use the New Socket](#358-step-7--update-the-nginx-server-block-to-use-the-new-socket)
- [35.9 Step 8 — Test and Reload Nginx](#359-step-8--test-and-reload-nginx)
- [35.10 Summary](#3510-summary)

---

## 35.1 Overview — Why Pools Matter for Security

PHP-FPM supports a pool-based architecture where each site runs its own dedicated set of PHP worker processes under its own dedicated system user. This is one of the most structurally important hardening steps in this setup — it sits beneath everything else: permissions, `open_basedir`, and function restrictions all depend on the pool user being correct.

**Without pool isolation:** Every site on the server shares the same PHP process running as `www-data`. If a plugin vulnerability is exploited on one site, the attacker's PHP code executes as `www-data`, which has read access to every other site's files on the server. **With a dedicated pool:** That same exploit is contained entirely within the compromised site's own document root — no lateral movement is possible.

| Aspect | Default (`www` pool) | Dedicated pool |
|--------|---------------------|----------------|
| User | `www-data` | Site-specific user (e.g. `expertwp`) |
| Socket | `/run/php/php8.3-fpm.sock` | `/run/php/php8.3-fpm-expertwp.sock` |
| Isolation | None — all sites share one process | Full — each site is independent |
| Blast radius on compromise | Entire server | Single site's document root |

> **Note:** Pools add a small resource overhead per site since each pool maintains its own worker processes. This is an acceptable trade-off for the isolation gained. The right time to set up a pool is immediately after installing WordPress, before any hardening that depends on file ownership.

---

## 35.2 Step 1 — Create a Dedicated System User for the Site

Each pool runs as a dedicated system user. This user owns the site files and is the identity under which PHP processes execute. The `www-data` user (Nginx's runtime user) is then added to the pool user's group so Nginx can still read static assets.

```bash
# Create the pool user (no login shell, no home directory needed)
sudo useradd expertwp

# Add www-data to the pool user's group so Nginx can read non-PHP files
sudo usermod -a -G expertwp www-data

# Add the pool user to the www-data group (allows PHP to serve files Nginx owns)
sudo usermod -a -G www-data expertwp

# Add the current admin user to the pool group for management access
sudo usermod -a -G expertwp $USER
```

After running these commands, verify group membership:

```bash
id andrew
# uid=1002(andrew) gid=1002(andrew) groups=1002(andrew),100(users),1003(expertwp)

id expertwp
# uid=1003(expertwp) gid=1003(expertwp) groups=1003(expertwp),33(www-data)
```

The output confirms the `expertwp` user exists and belongs to the `www-data` group, and that the admin user `andrew` is now a member of the `expertwp` group.

> **Do not use `www-data` as the pool user.** Running everything as `www-data` defeats the isolation entirely. Always create a dedicated user per site.

---

## 35.3 Step 2 — Locate and Inspect the PHP-FPM Pool Directory

PHP-FPM pool configuration files live under the version-specific FPM directory:

```bash
cd /etc/php/8.3/fpm/pool.d
ls
```

Output:

```
www.conf
```

The `pool.d/` directory holds one `.conf` file per pool. The default `www.conf` defines the `[www]` pool that ships with PHP-FPM. This file is used as a template — it is copied and customised for each new site.

---

## 35.4 Step 3 — Create a New Pool Configuration from the Default

Rather than writing a pool config from scratch, copy the default `www.conf` as a starting point:

```bash
cd /etc/php/8.3/fpm/pool.d/
sudo cp www.conf expertwp.help.conf
ls
```

Output:

```
expertwp.help.conf  www.conf
```

Open the new file for editing:

```bash
sudo nano expertwp.help.conf
```

---

## 35.5 Step 4 — Edit the Pool Configuration File

The pool config file is well-commented. The key directives to change are documented below in the order they appear in the file.

### 4.1 — Pool Name

At the very top of the file, the pool name is set inside square brackets. Change it from the default `[www]` to a name matching the site:

```ini
[expertwp]
```

This name is used internally by PHP-FPM to identify the pool and is also available as the `$pool` variable within the config file itself.

### 4.2 — Pool User and Group

Find the `user` and `group` directives. Change both from `www-data` to the dedicated pool user created in Step 1:

```ini
user = expertwp
group = expertwp
```

PHP worker processes for this site will now run as `expertwp`, not as `www-data`.

### 4.3 — Unix Socket Path

The `listen` directive sets the path to the Unix socket this pool listens on for FastCGI connections from Nginx. Change it from the default to a site-specific path:

```ini
listen = /run/php/php8.3-fpm-expertwp.sock
```

Using a unique socket per site is essential — this is what allows Nginx to route requests for each site to the correct pool. The socket filename should clearly identify both the PHP version and the site name.

### 4.4 — Resource Limits

Set the open file descriptor and core size limits for workers in this pool:

```ini
rlimit_files = 15000
rlimit_core = 100
```

`rlimit_files` controls how many files a single PHP worker process can have open simultaneously. `15000` is a sensible value for a busy WordPress site. `rlimit_core` limits the size of any core dump files generated if a worker crashes.

### 4.5 — PHP Error Logging

At the bottom of the file, uncomment and configure the error logging directives. Update the `error_log` path to use a site-specific log file:

```ini
php_flag[display_errors] = off
php_admin_value[error_log] = /var/log/fpm-php.expertwp.log
php_admin_flag[log_errors] = on
```

- `display_errors = off` ensures PHP errors are never shown in the browser — they go to the log file only. Required in production.
- `error_log` points to the site-specific PHP error log created in the next step.
- `log_errors = on` activates error logging to that file.

Save and close the file.

---

## 35.6 Step 5 — Create the PHP-FPM Error Log File

The log file referenced in the pool config must exist before PHP-FPM is reloaded, and it must be readable and writable by both the pool user and `www-data`:

```bash
# Create an empty log file
sudo touch /var/log/fpm-php.expertwp.log

# Set ownership: pool user owns it, www-data group can read/write
sudo chown expertwp:www-data /var/log/fpm-php.expertwp.log

# Set permissions: owner and group can read/write, others cannot
sudo chmod 660 /var/log/fpm-php.expertwp.log
```

---

## 35.7 Step 6 — Reload PHP-FPM and Verify the Socket

Reload PHP-FPM to apply the new pool configuration:

```bash
sudo systemctl reload php8.3-fpm
```

Confirm the pool is active and the socket path is correct by grepping the config:

```bash
sudo grep "listen = /" /etc/php/8.3/fpm/pool.d/expertwp.help.conf
# listen = /run/php/php8.3-fpm-expertwp.sock
```

The socket file at `/run/php/php8.3-fpm-expertwp.sock` should now exist on the filesystem.

---

## 35.8 Step 7 — Update the Nginx Server Block to Use the New Socket

The Nginx virtual host for the site must be updated to pass PHP requests to the new pool's socket instead of the default one. Open the site's Nginx config:

```bash
cd /etc/nginx/sites-available/
sudo nano expertwp.help.conf
```

Locate the PHP processing location block and update the `fastcgi_pass` directive to the site-specific socket:

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
        # Updated from the default php8.3-fpm.sock to the site-specific socket
        fastcgi_pass unix:/run/php/php8.3-fpm-expertwp.sock;
        include /etc/nginx/includes/fastcgi_optimize.conf;
    }

    include /etc/nginx/includes/browser_caching.conf;

    access_log /var/log/nginx/access.log.expertwp.help.log combined buffer=256k flush=60m;
    error_log  /var/log/nginx/error.log.expertwp.help.log;
}
```

The critical change is `fastcgi_pass unix:/run/php/php8.3-fpm-expertwp.sock;` — this routes all PHP execution for this site through the dedicated pool.

---

## 35.9 Step 8 — Test and Reload Nginx

```bash
sudo nginx -t
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

```bash
sudo systemctl reload nginx
```

Both commands must succeed before proceeding. If `nginx -t` reports an error, the socket path in the `fastcgi_pass` directive is the most likely cause — verify it matches exactly what was set in the pool config's `listen` directive.

---

## 35.10 Summary

### Files Modified

| File | Change |
|------|--------|
| `/etc/php/8.3/fpm/pool.d/expertwp.help.conf` | New pool: name, user/group, socket path, resource limits, error logging |
| `/var/log/fpm-php.expertwp.log` | Created with `expertwp:www-data` ownership and `660` permissions |
| `/etc/nginx/sites-available/expertwp.help.conf` | `fastcgi_pass` updated to site-specific socket |

### Commands

```bash
# --- User setup ---
sudo useradd expertwp
sudo usermod -a -G expertwp www-data
sudo usermod -a -G www-data expertwp
sudo usermod -a -G expertwp $USER

# --- Pool config ---
cd /etc/php/8.3/fpm/pool.d/
sudo cp www.conf expertwp.help.conf
sudo nano expertwp.help.conf
# Edit: pool name, user, group, listen socket, rlimit_files, rlimit_core, error log

# --- Log file ---
sudo touch /var/log/fpm-php.expertwp.log
sudo chown expertwp:www-data /var/log/fpm-php.expertwp.log
sudo chmod 660 /var/log/fpm-php.expertwp.log

# --- Apply and verify ---
sudo systemctl reload php8.3-fpm
sudo grep "listen = /" /etc/php/8.3/fpm/pool.d/expertwp.help.conf

# --- Nginx ---
sudo nano /etc/nginx/sites-available/expertwp.help.conf
# Update: fastcgi_pass unix:/run/php/php8.3-fpm-expertwp.sock;
sudo nginx -t
sudo systemctl reload nginx
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 34 — WordPress Hardening Introduction](./34-wordpress-hardening-introduction.md) | [↑ Top](#chapter-35--php-fpm-pool-isolation) | [↑ Back to Index](../index.md) |
