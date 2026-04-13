[← Back to Index](../index.md)

---

# Chapter 44 — Nginx Rate Limiting for WordPress

*Targeted, low-risk rate limiting for the two highest-risk WordPress endpoints — `wp-login.php` and `xmlrpc.php` — using a per-site include file so that brute-force credential stuffing bots are dropped at the Nginx layer before PHP ever processes their requests.*

---

## Contents

- [44.1 Overview](#441-overview)
- [44.2 Step 1 — Define the Rate Limiting Zone in nginx.conf](#442-step-1--define-the-rate-limiting-zone-in-nginxconf)
- [44.3 Step 2 — Create the Rate Limiting Include File](#443-step-2--create-the-rate-limiting-include-file)
- [44.4 Step 3 — Verify the Include File Path](#444-step-3--verify-the-include-file-path)
- [44.5 Step 4 — Add the Include to the Site's Server Block](#445-step-4--add-the-include-to-the-sites-server-block)
- [44.6 Step 5 — Test and Reload](#446-step-5--test-and-reload)
- [44.7 Summary of Files Modified](#447-summary-of-files-modified)

---

## 44.1 Overview

Rate limiting in Nginx works by capping the number of GET and/or POST requests a single IP address can make within a given time window. Unlike broad DDoS mitigation (which needs to happen at the network edge), rate limiting is well-suited to targeted, high-risk WordPress endpoints — specifically `wp-login.php` and `xmlrpc.php`. These are the two endpoints most commonly hammered by brute-force credential stuffing bots.

The key advantage of applying rate limiting at the Nginx level, rather than relying on a WordPress security plugin, is that Nginx intercepts and drops the requests before PHP or WordPress ever processes them. This makes it significantly more efficient — a plugin-based approach still loads the entire WordPress stack for each blocked request, consuming PHP workers and database connections even for traffic that should be rejected outright.

When a request exceeds the configured rate limit, Nginx returns HTTP status code `444`. This is an Nginx-only response code that instructs the server to close the connection immediately without sending any response body — effectively making the server invisible to the attacking client.

---

## 44.2 Step 1 — Define the Rate Limiting Zone in `nginx.conf`

The rate limiting zone is defined at the `http` block level in the main `nginx.conf`. This makes it globally available to any server block that references it.

```bash
cd /etc/nginx/
sudo nano nginx.conf
```

Inside the `http` block, add the rate limiting zone definition (after Gzip/Brotli settings and before the Virtual Host Configs section):

```nginx
# Rate Limiting
limit_req_zone $binary_remote_addr zone=wp:10m rate=30r/m;
```

**Breaking down this directive:**

- `$binary_remote_addr` — uses the client's IP address (in compact binary form) as the key to track request counts per unique IP.
- `zone=wp:10m` — creates a shared memory zone named `wp` with 10 MB of storage. This is enough to track tens of thousands of unique IP addresses simultaneously.
- `rate=30r/m` — sets the allowed rate to 30 requests per minute per IP. This is a deliberately conservative limit suited to login and XML-RPC endpoints, not general site traffic.

Save the file and verify the configuration is valid:

```bash
sudo nginx -t
```

---

## 44.3 Step 2 — Create the Rate Limiting Include File

Rate limiting rules are isolated into a dedicated include file. This keeps the server block clean and makes the rules reusable.

```bash
cd /etc/nginx/includes/
sudo nano rate_limiting_example.com.conf
```

Add the following two location blocks:

```nginx
location = /wp-login.php {
    limit_req zone=wp burst=20 nodelay;
    limit_req_status 444;
    include snippets/fastcgi-php.conf;
    fastcgi_param HTTP_HOST $host;
    fastcgi_pass unix:/run/php/php8.3-fpm-example.com.sock;
    include /etc/nginx/includes/fastcgi_optimize.conf;
}

location = /xmlrpc.php {
    limit_req zone=wp burst=20 nodelay;
    limit_req_status 444;
    include snippets/fastcgi-php.conf;
    fastcgi_param HTTP_HOST $host;
    fastcgi_pass unix:/run/php/php8.3-fpm-example.com.sock;
    include /etc/nginx/includes/fastcgi_optimize.conf;
}
```

**Key directives explained:**

`location = /wp-login.php` — the `=` modifier is an exact match, meaning this block applies only to requests for that precise URI. No prefix matching or regex involved, which makes it the fastest matching type Nginx supports.

`limit_req zone=wp burst=20 nodelay;` — applies the `wp` rate limiting zone. The `burst=20` parameter allows up to 20 requests to queue temporarily if the rate is briefly exceeded (accommodating legitimate users who may open multiple tabs). `nodelay` means these burst requests are processed immediately rather than being artificially delayed — once the burst is exhausted, additional requests are rejected outright.

`limit_req_status 444;` — when the limit is exceeded, Nginx closes the connection with no response, which is more effective against bots than returning a standard HTTP error code.

The `fastcgi_pass` directive must point to the correct PHP-FPM socket for the site. Replace `example.com` in the socket path with the actual site's pool socket name.

Save the file.

---

## 44.4 Step 3 — Verify the Include File Path

Before referencing the file from the site config, confirm it exists at the correct path:

```bash
ls /etc/nginx/includes/
readlink -f rate_limiting_example.com.conf
```

---

## 44.5 Step 4 — Add the Include to the Site's Server Block

Open the site's Nginx server block configuration:

```bash
cd /etc/nginx/sites-available/
sudo nano example.com.conf
```

Add the rate limiting include inside the `server` block, after the existing security directives include and before the browser caching include. The final server block structure:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    listen 443 quic reuseport;
    http3 on;

    server_name example.com www.example.com;
    root /var/www/example.com/public_html;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    include /etc/nginx/ssl/ssl_example.com.conf;
    include /etc/nginx/ssl/ssl_all_sites.conf;

    include /etc/nginx/includes/http_headers.conf;
    include /etc/nginx/includes/nginx_security_directives.conf;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_param HTTP_HOST $host;
        fastcgi_pass unix:/run/php/php8.3-fpm-example.com.sock;
        include /etc/nginx/includes/fastcgi_optimize.conf;
    }

    # Rate limiting for high-risk WordPress endpoints
    include /etc/nginx/includes/rate_limiting_example.com.conf;

    include /etc/nginx/includes/browser_caching.conf;

    access_log /var/log/nginx/access.log.example.com.log combined buffer=256k flush=60m;
}
```

> **Important:** The `fastcgi_pass` socket path inside the rate limiting include file must match the site's PHP-FPM pool socket exactly. A mismatch will cause a Nginx configuration test failure.

---

## 44.6 Step 5 — Test and Reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

## 44.7 Summary of Files Modified

| File | Change |
|------|--------|
| `/etc/nginx/nginx.conf` | Added `limit_req_zone` directive in the `http` block |
| `/etc/nginx/includes/rate_limiting_example.com.conf` | Created — rate-limited location blocks for `wp-login.php` and `xmlrpc.php` |
| `/etc/nginx/sites-available/example.com.conf` | Added `include` for the rate limiting file |

> **Note for automation:** The rate limiting zone name (`wp`) and rate (`30r/m`) are defined in `nginx.conf` and referenced by name in the include file. When scripting, the zone name must be consistent between both files. The PHP-FPM socket path in the rate limiting include must be dynamically substituted with the correct per-site pool socket at deploy time.

---

| | | |
|:---|:---:|---:|
| [← Chapter 43 — Nginx DDoS Protection Considerations](./43-nginx-ddos-protection.md) | [↑ Top](#chapter-44--nginx-rate-limiting-for-wordpress) | [Chapter 45 — Web Application Firewall (NinjaFirewall) →](./45-web-application-firewall.md) |
