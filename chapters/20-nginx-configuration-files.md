[← Back to Index](../index.md)

---

# Chapter 20 — Nginx Configuration Files

*How Nginx configuration works — directives, contexts, configuration testing, the location matching system, and the `try_files` directive that powers WordPress pretty permalinks.*

---

## Contents

- [20.1 Directives](#201-directives)
- [20.2 Configuration Testing and Reloading](#202-configuration-testing-and-reloading)
  - [Testing the Configuration](#testing-the-configuration)
  - [Fixing Errors](#fixing-errors)
  - [Reloading Nginx](#reloading-nginx)
- [20.3 Nginx Contexts](#203-nginx-contexts)
- [20.4 Location Context Modifiers](#204-location-context-modifiers)
  - [Exact Match (=)](#exact-match-)
  - [No Modifier — Prefix Match](#no-modifier--prefix-match)
  - [Case-Sensitive Regex (~)](#case-sensitive-regex-)
  - [Case-Insensitive Regex (~*)](#case-insensitive-regex-)
  - [Preferential Prefix Match (^~)](#preferential-prefix-match-)
- [20.5 The try_files Directive](#205-the-try_files-directive)
  - [How Each Argument is Resolved](#how-each-argument-is-resolved)
- [20.6 WordPress Pretty Permalinks](#206-wordpress-pretty-permalinks)
  - [Standard WordPress try_files Configuration](#standard-wordpress-try_files-configuration)
  - [Advanced Usage — WP Super Cache Integration](#advanced-usage--wp-super-cache-integration)
  - [Verifying a PHP Script Exists Before Passing to FastCGI](#verifying-a-php-script-exists-before-passing-to-fastcgi)

---

## 20.1 Directives

Nginx behaviour is controlled through **directives**. A directive is an instruction that sets a specific option or parameter for an Nginx module. The syntax is straightforward: the directive name comes first, followed by its value, and every directive ends with a semicolon.

```nginx
root /var/www/html;
index index.html index.htm;
server_name example.com;
```

**Rules to remember:**

- Every directive must be placed in the correct context — putting a directive in the wrong context will cause Nginx to fail its configuration test.
- Duplicate directives are not allowed within the same context.
- Always test the configuration before reloading Nginx. Reloading is what activates any changes you've made.

---

## 20.2 Configuration Testing and Reloading

The golden rule when editing Nginx config files: **test first, reload second**. If there is an error in the config, Nginx will report it at test time rather than silently breaking the live server.

### Testing the Configuration

```bash
sudo nginx -t
```

When there is a syntax error — for example, using `roots` instead of `root` — Nginx will pinpoint the exact file and line number:

```
2024/07/10 20:08:13 [emerg] 17182#17182: unknown directive "roots" in /etc/nginx/sites-enabled/default:41
nginx: configuration file /etc/nginx/nginx.conf test failed
```

### Fixing Errors

If the test fails, re-open the file, locate and correct the mistake, save, then test again:

```bash
sudo nano /etc/nginx/sites-available/default -l   # -l shows line numbers for easier navigation
sudo nginx -t
```

On success:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Reloading Nginx

Only after a clean test, reload Nginx to apply the changes:

```bash
sudo systemctl reload nginx
```

> **Note:** Reload is preferred over restart — it applies the new configuration gracefully without dropping active connections.

---

## 20.3 Nginx Contexts

A **context** is a directive that contains other directives inside curly braces `{}`. It defines the scope in which the enclosed directives apply. Nginx has four main contexts, nested inside one another in a fixed hierarchy.

```
main context
└── events context
└── http context
    └── server context
        └── location context
```

| Context | Purpose |
|---------|---------|
| `main` | Top-level settings — worker processes, PID file, included modules |
| `events` | Connection handling — max connections, use of epoll, multi_accept |
| `http` | All HTTP settings — MIME types, gzip, SSL protocols, virtual hosts |
| `server` | Virtual host definition — listen port, server_name, document root |
| `location` | Per-URI rules — routing, file serving, PHP passing, rewrites |

A typical structure:

```nginx
# main context
user www-data;
worker_processes 1;
pid /run/nginx.pid;

events {
    worker_connections 2048;
    multi_accept on;
    use epoll;
}

http {
    sendfile on;
    tcp_nopush on;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    server {
        listen 80 default_server;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;
        server_name _;

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        }
    }
}
```

---

## 20.4 Location Context Modifiers

The `location` block is how Nginx decides which configuration to apply to a given request URI. A **modifier** placed before the URI pattern changes how that pattern is matched. Nginx evaluates modifiers in a defined priority order.

| Modifier | Match Type | Priority |
|----------|-----------|----------|
| `=` | Exact match — case sensitive | 1 (highest) |
| `^~` | Prefix match — takes priority over regex | 2 |
| `~` | Case-sensitive regular expression match | 3 |
| `~*` | Case-insensitive regular expression match | 3 |
| (none) | Prefix match — lowest priority, evaluated last | 4 |

### Exact Match (`=`)

Matches the URI exactly as written. Case-sensitive.

```nginx
# Only blocks requests to exactly /xmlrpc.php
# Does NOT match /XmlRpc.php or /XMLRPC.PHP
location = /xmlrpc.php {
    deny all;
}
```

### No Modifier — Prefix Match

Without any modifier, Nginx matches any URI that *starts with* the given prefix. Case sensitive.

```nginx
# Matches /tax, /taxis, /ta — anything starting with /ta
# Does NOT match /Tax (wrong case)
location /tax {
    return 301 domain.com;
}
```

### Case-Sensitive Regex (`~`)

Uses a regular expression to match the URI. Case-sensitive.

```nginx
# Blocks access to files ending in .ico (not .ICO)
location ~ \.ico$ {
    deny all;
}
```

### Case-Insensitive Regex (`~*`)

Same as `~` but the match ignores case — useful for blocking file extensions regardless of how they are capitalised.

```nginx
# Blocks .txt AND .TXT
location ~* \.txt$ {
    deny all;
}
```

A practical WordPress security example — blocking PHP execution inside the uploads directory regardless of case:

```nginx
location ~* ^/wp\-content/uploads/.*\.((?:php[1-7]?|pht|phtml?|phps))$ {
    deny all;
}
```

### Preferential Prefix Match (`^~`)

Matches as a prefix (like no modifier), but has higher priority than any regex location. Once Nginx finds a `^~` match, it stops evaluating regex blocks entirely.

```nginx
# Redirects any request starting with /images to /new_images
location ^~ /images {
    return 301 /new_images;
}
```

---

## 20.5 The `try_files` Directive

The `try_files` directive instructs Nginx to check for the existence of files or directories in a specified order. It serves the first one it finds. If none of the listed options exist, it falls through to a final fallback — typically a named location, a redirect, or an error code.

```nginx
location / {
    try_files $uri $uri/ /other/index.html;
}
```

### How Each Argument is Resolved

Given a request for `example.com/image.jpg` and a site root of `/var/www/example.com/public_html`:

**`$uri`** — Nginx checks if the requested file exists as a static file on disk:

```
/var/www/example.com/public_html/image.jpg
```

If found, serve it. If not, move to the next condition.

**`$uri/`** — Nginx looks for a directory with the same name as the request and tries to serve an index file inside it:

```
/var/www/example.com/public_html/image.jpg/index.html
```

If no such directory exists, move to the next condition.

**`/other/index.html`** — the fallback. If the first two conditions both fail, Nginx serves this file:

```
/var/www/example.com/public_html/other/index.html
```

If this file also doesn't exist, Nginx returns a `404`.

---

## 20.6 WordPress Pretty Permalinks

The `try_files` directive is directly responsible for WordPress Pretty Permalinks working correctly. Without it, WordPress URLs like `example.com/my-post-title/` would return 404 because there is no physical file at that path — the request needs to be handed off to `index.php`.

### Standard WordPress `try_files` Configuration

```nginx
location / {
    try_files $uri $uri/ /index.php$is_args$args;
}
```

**How the fallback works:**

- `$uri` — Nginx checks for the requested file at the URL path (e.g. a static asset)
- `$uri/` — if no file, Nginx looks for a directory index
- `/index.php$is_args$args` — if neither exists (which is the case for all WordPress permalink URLs), Nginx passes the request internally to `index.php`, appending any query string arguments
  - `$is_args` evaluates to `?` if `$args` is non-empty, and empty otherwise — this ensures the query string separator is only added when needed
  - `$args` contains the actual query string parameters

### Advanced Usage — WP Super Cache Integration

When using a caching plugin like WP Super Cache, `try_files` can first check for a pre-generated static HTML cache file before falling back to PHP:

```nginx
# Check for cached static file; if it doesn't exist, pass to WordPress
location / {
    try_files /wp-content/cache/supercache/$http_host/$cache_uri/index-https.html $uri $uri/ /index.php?$args;
}
```

### Verifying a PHP Script Exists Before Passing to FastCGI

Inside a PHP location block, `try_files` can also guard against passing non-existent PHP files to the FastCGI processor:

```nginx
# Check that the PHP script exists before passing it to php-fpm
try_files $fastcgi_script_name =404;
```

This returns a `404` if the requested PHP file doesn't exist on disk, preventing potential security issues from being passed to the upstream PHP process.

---

| | | |
|:---|:---:|---:|
| [← Chapter 19 — Server Mail with msmtp](./19-server-mail-msmtp.md) | [↑ Top](#chapter-20--nginx-configuration-files) | [↑ Back to Index](../index.md) |
