[← Back to Index](../index.md)

---

# Chapter 41 — HTTP Response Headers

*Configuring security-focused HTTP response headers in Nginx — `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Referrer-Policy`, and `Permissions-Policy` — using a modular include file that applies them consistently across every response.*

---

## Contents

- [41.1 Overview](#411-overview)
- [41.2 Headers Configured in This Chapter](#412-headers-configured-in-this-chapter)
- [41.3 Strategy: Using an Include File](#413-strategy-using-an-include-file)
- [41.4 Step 1 — Create the Security Headers Include File](#414-step-1--create-the-security-headers-include-file)
- [41.5 Step 2 — Include the Header File in the Site Server Block](#415-step-2--include-the-header-file-in-the-site-server-block)
- [41.6 Step 3 — Verify the Headers Are Being Served](#416-step-3--verify-the-headers-are-being-served)
- [41.7 Step 4 — Update browser_caching.conf to Also Carry Security Headers](#417-step-4--update-browser_cachingconf-to-also-carry-security-headers)
- [41.8 Step 5 — Verify Headers on Static Assets](#418-step-5--verify-headers-on-static-assets)
- [41.9 Summary of Files Modified](#419-summary-of-files-modified)

---

## 41.1 Overview

HTTP headers are a core part of every HTTP request and response cycle. They carry metadata that controls how browsers and clients interpret and handle server responses. For a hardened WordPress server, the response headers sent by Nginx are one of the most impactful and low-cost security improvements available — they require no changes to WordPress itself and no additional software.

Reference: [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)

---

## 41.2 Headers Configured in This Chapter

| Header | Purpose |
|--------|---------|
| `X-Frame-Options` | Prevents clickjacking by blocking the site from being rendered inside an iframe on another origin |
| `X-Content-Type-Options` | Stops MIME-type sniffing — browsers must respect the declared `Content-Type` |
| `X-XSS-Protection` | Legacy XSS filter for older browsers (modern browsers use CSP instead) |
| `Referrer-Policy` | Controls how much referrer information is included in requests leaving the site |
| `Permissions-Policy` | Allows or denies access to browser features (camera, microphone, geolocation, etc.) |
| `Content-Security-Policy` | Defines trusted sources for scripts, styles, images, and other resources (advanced — deferred) |

> **Note on `X-XSS-Protection`:** Modern browsers (Chrome, Firefox, Edge) have removed or deprecated their built-in XSS auditors. This header is kept for compatibility with older browsers, but a properly configured `Content-Security-Policy` is the correct long-term solution.

> **Note on `X-Frame-Options`:** The `frame-ancestors` directive in `Content-Security-Policy` supersedes this header in supporting browsers. Both can coexist safely.

---

## 41.3 Strategy: Using an Include File

Rather than embedding these headers directly inside each site's server block, they are placed in a dedicated include file. This keeps the site config clean and makes it trivial to apply the same headers to every site on the server by adding a single `include` line.

The include file lives in `/etc/nginx/includes/`, alongside the other modular config files already in use (gzip, browser caching, FastCGI settings, etc.).

---

## 41.4 Step 1 — Create the Security Headers Include File

```bash
cd /etc/nginx/includes/
sudo nano http_headers.conf
```

Contents:

```nginx
# -------------------------------------------------------
# Referrer-Policy — controls referrer info sent with requests
# Uncomment the desired directive, comment out the others
# -------------------------------------------------------
#add_header Referrer-Policy "no-referrer";
add_header Referrer-Policy "strict-origin-when-cross-origin";
#add_header Referrer-Policy "unsafe-url";

# -------------------------------------------------------
add_header X-Content-Type-Options "nosniff";
add_header X-Frame-Options "sameorigin";
add_header X-XSS-Protection "1; mode=block";
add_header Permissions-Policy 'accelerometer=(), camera=(), clipboard-read=(), clipboard-write=(),
geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=(), fullscreen=(self
"https://www.youtube.com")';
```

**What each directive does:**

`Referrer-Policy "strict-origin-when-cross-origin"` — sends the full URL as referrer for same-origin requests, but only the origin (no path) for cross-origin HTTPS requests, and nothing at all for cross-origin HTTP requests. A good balance between analytics usefulness and privacy.

`X-Content-Type-Options "nosniff"` — instructs browsers not to guess the content type. If Nginx declares a file as `text/css`, the browser must treat it as CSS even if its contents look like JavaScript. Prevents content-injection attacks.

`X-Frame-Options "sameorigin"` — allows the site to be embedded in an iframe only on pages served from the same origin. Blocks external sites from framing the site (clickjacking mitigation).

`X-XSS-Protection "1; mode=block"` — enables the legacy XSS filter in older IE/Safari browsers and tells the browser to block the page rather than sanitise it when an attack is detected.

`Permissions-Policy` — restricts access to browser hardware and sensitive APIs. The above configuration disables geolocation, camera, microphone, magnetometer, gyroscope, accelerometer, clipboard access, payments, and USB access by default, while permitting fullscreen for the same origin and YouTube (needed for embedded video players in WordPress).

---

## 41.5 Step 2 — Include the Header File in the Site Server Block

Open the site's Nginx config:

```bash
cd /etc/nginx/sites-available/
sudo nano example.com.conf
```

Add the include directive above the PHP processing location block:

```nginx
server {
    listen 443 ssl;
    http2 on;
    listen 443 quic reuseport;
    http3 on;

    server_name example.com www.example.com;
    root /var/www/example.com/public_html;
    index index.php;

    include /etc/nginx/ssl/ssl_example.com.conf;
    include /etc/nginx/ssl/ssl_all_sites.conf;

    # Security headers — place above the PHP location block
    include /etc/nginx/includes/http_headers.conf;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_param HTTP_HOST $host;
        fastcgi_pass unix:/run/php/php8.3-fpm-example.com.sock;
        include /etc/nginx/includes/fastcgi_optimize.conf;
    }
}
```

Test the configuration and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 41.6 Step 3 — Verify the Headers Are Being Served

```bash
curl -I https://example.com
```

After a successful reload, the response will include the new headers:

```
HTTP/2 200
referrer-policy: strict-origin-when-cross-origin
x-content-type-options: nosniff
x-frame-options: sameorigin
x-xss-protection: 1; mode=block
permissions-policy: geolocation=(), midi=(), sync-xhr=(), microphone=(), camera=(), ...
```

---

## 41.7 Step 4 — Update `browser_caching.conf` to Also Carry Security Headers

The browser caching include file serves static assets from `location` blocks that are separate from the main server block context. Since `add_header` directives in Nginx do **not** inherit from parent contexts when a child `location` block defines its own `add_header`, the security headers must be explicitly included inside each caching block as well.

```bash
sudo nano /etc/nginx/includes/browser_caching.conf
```

Update each location block to include both the caching directives and the security headers include:

```nginx
# Media, archives, and binary assets — cache for 1 year
location ~* \.(webp|3gp|gif|jpg|jpeg|png|ico|wmv|avi|asf|asx|mpg|mpeg|mp4|pls|mp3|mid|wav|swf|flv|exe|zip|tar|rar)$ {
    expires 365d;
    etag on;
    if_modified_since exact;
    add_header Pragma "public";
    add_header Cache-Control "public, no-transform";
    try_files $uri $uri/ /index.php?$args;
    include /etc/nginx/includes/http_headers.conf;
    access_log off;
}

# JavaScript — cache for 30 days
location ~* \.(js)$ {
    expires 30d;
    etag on;
    if_modified_since exact;
    add_header Pragma "public";
    add_header Cache-Control "public, no-transform";
    try_files $uri $uri/ /index.php?$args;
    include /etc/nginx/includes/http_headers.conf;
    access_log off;
}

# Stylesheets — cache for 30 days
location ~* \.(css)$ {
    expires 30d;
    etag on;
    if_modified_since exact;
    add_header Pragma "public";
    add_header Cache-Control "public, no-transform";
    try_files $uri $uri/ /index.php?$args;
    include /etc/nginx/includes/http_headers.conf;
    access_log off;
}

# Web fonts — cache for 30 days
location ~* \.(eot|svg|ttf|woff|woff2)$ {
    expires 30d;
    etag on;
    if_modified_since exact;
    add_header Pragma "public";
    add_header Cache-Control "public, no-transform";
    try_files $uri $uri/ /index.php?$args;
    include /etc/nginx/includes/http_headers.conf;
    access_log off;
}
```

**New caching directives added alongside the existing ones:**

`etag on` — enables ETag generation. The browser can use the ETag for conditional requests (`If-None-Match`), so Nginx returns a `304 Not Modified` instead of retransmitting the full file if the asset hasn't changed.

`if_modified_since exact` — pairs with the `Last-Modified` header for conditional GET requests. `exact` means the timestamp must match precisely (no rounding).

`add_header Pragma "public"` — legacy HTTP/1.0 caching directive, included for compatibility with old proxy servers.

`try_files $uri $uri/ /index.php?$args` — ensures requests for assets that don't exist on disk are passed to WordPress rather than returning a 404 directly from Nginx.

`include /etc/nginx/includes/http_headers.conf` — ensures security headers are present on all static asset responses, not just dynamic PHP responses.

Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 41.8 Step 5 — Verify Headers on Static Assets

```bash
curl -I https://example.com/wp-content/plugins/some-plugin/css/style.css
```

Expected response headers will include both caching metadata and all security headers:

```
HTTP/2 200
content-type: text/css
etag: "..."
expires: <30 days from now>
cache-control: max-age=2592000
pragma: public
cache-control: public, no-transform
referrer-policy: strict-origin-when-cross-origin
x-content-type-options: nosniff
x-frame-options: sameorigin
x-xss-protection: 1; mode=block
permissions-policy: geolocation=(), ...
```

---

## 41.9 Summary of Files Modified

| File | Change |
|------|--------|
| `/etc/nginx/includes/http_headers.conf` | Created — contains all security response header directives |
| `/etc/nginx/includes/browser_caching.conf` | Updated — each location block now includes `http_headers.conf` and the `etag`/`if_modified_since` directives |
| `/etc/nginx/sites-available/example.com.conf` | Updated — added `include /etc/nginx/includes/http_headers.conf` above the PHP location block |

---

| | | |
|:---|:---:|---:|
| [← Chapter 40 — HTTPS Verification & SSL Renewal](./40-https-verification-certbot-renewal.md) | [↑ Top](#chapter-41--http-response-headers) | [Chapter 42 — Nginx WordPress Security Directives →](./42-nginx-wordpress-security-directives.md) |
