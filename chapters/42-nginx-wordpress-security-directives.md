[← Back to Index](../index.md)

---

# Chapter 42 — Nginx WordPress Security Directives

*A request-layer firewall ruleset added directly to Nginx — blocking sensitive file access, preventing PHP execution in writable directories, filtering suspicious query strings, SQL injection, file injection, common exploits, spam, and known bad user agents — before any request reaches PHP or WordPress.*

---

## Contents

- [42.1 Overview of What Gets Blocked](#421-overview-of-what-gets-blocked)
- [42.2 Step 1 — Create the Security Directives Include File](#422-step-1--create-the-security-directives-include-file)
- [42.3 Key Points About the Ruleset Design](#423-key-points-about-the-ruleset-design)
- [42.4 Step 2 — Include the File in the Site Server Block](#424-step-2--include-the-file-in-the-site-server-block)
- [42.5 Step 3 — Verify the Firewall Rules Are Active](#425-step-3--verify-the-firewall-rules-are-active)
- [42.6 Step 4 — Handle Plugins That Need PHP Execution in the Plugins Directory](#426-step-4--handle-plugins-that-need-php-execution-in-the-plugins-directory)
- [42.7 Final Site Config Structure](#427-final-site-config-structure)
- [42.8 Summary of Files Modified](#428-summary-of-files-modified)

---

## 42.1 Overview of What Gets Blocked

| Category | What It Does |
|----------|-------------|
| Sensitive file access | Blocks direct access to `wp-config.php`, `readme.html`, `license.txt`, `.ini` files, `wp-admin/install.php`, `wp-includes/` PHP files |
| PHP execution in writable dirs | Denies PHP execution inside `uploads/`, `plugins/`, and `themes/` directories |
| PHP override file | Blocks access to `.user.ini` |
| Request method filtering | Returns 403 for `TRACE`, `DELETE`, and `TRACK` methods |
| Suspicious query strings | Blocks path traversal, shell files, `etc/passwd`, script injection, base64 encoded payloads, and known CMS config parameters |
| SQL injection | Blocks `union...select`, `union...all...select`, and `concat(` patterns in query strings |
| File injection | Blocks `http://` references and relative path traversal in query parameters |
| Common exploits | Blocks GLOBALS, REQUEST, `proc/self/environ`, mosConfig, and base64 encode/decode in query strings |
| Spam | Blocks pharmaceutical spam keywords in query strings |
| Bad user agents | Blocks known vulnerability scanners: nikto, sqlmap, dirbuster, nessus, acunetix, openvas, and others |

---

## 42.2 Step 1 — Create the Security Directives Include File

Navigate to the Nginx includes directory:

```bash
cd /etc/nginx/includes/
sudo nano nginx_security_directives.conf
```

Paste the complete ruleset into the file:

```nginx
##
# WORDPRESS-SAFE NGINX FIREWALL RULESET
# Low false-positive risk
# Updated: 2026

# -------------------------------------------------------
# BLOCK ACCESS TO SENSITIVE WORDPRESS FILES
# -------------------------------------------------------

# xmlrpc.php — comment out to allow XML-RPC (required by some plugins/apps)
#location = /xmlrpc.php { deny all; }

location = /wp-config.php { deny all; }
location = /wp-admin/install.php { deny all; }

location ~* /readme\.html$ { deny all; }
location ~* /readme\.txt$ { deny all; }
location ~* /licence\.txt$ { deny all; }
location ~* /license\.txt$ { deny all; }

location ~ ^/wp-admin/includes/ { deny all; }
location ~ ^/wp-includes/[^/]+\.php$ { deny all; }
location ~ ^/wp-includes/js/tinymce/langs/.+\.php$ { deny all; }
location ~ ^/wp-includes/theme-compat/ { deny all; }

# -------------------------------------------------------
# BLOCK PHP EXECUTION IN UPLOADS, PLUGINS, AND THEMES
# -------------------------------------------------------
location ~* ^/wp-content/uploads/.*\.(?:php[1-7]?|pht|phtml?|phps)$ { deny all; }
location ~* ^/wp-content/plugins/.*\.(?:php[1-7]?|pht|phtml?|phps)$ { deny all; }
location ~* ^/wp-content/themes/.*\.(?:php[1-7]?|pht|phtml?|phps)$ { deny all; }

# Block the site PHP override file
location = /.user.ini { deny all; }

# -------------------------------------------------------
# FILTER REQUEST METHODS
# -------------------------------------------------------
if ( $request_method ~* ^(TRACE|DELETE|TRACK)$ ) { return 403; }

# -------------------------------------------------------
# FILTER SUSPICIOUS QUERY STRINGS
# -------------------------------------------------------
set $susquery 0;
if ( $args ~* "\.\.\/" ) { set $susquery 1; }
if ( $args ~* "\.(bash|git|hg|log|svn|swp|cvs)" ) { set $susquery 1; }
if ( $args ~* "etc/passwd" ) { set $susquery 1; }
if ( $args ~* "boot\.ini" ) { set $susquery 1; }
if ( $args ~* "ftp:" ) { set $susquery 1; }
if ( $args ~* "(<|%3C)script(>|%3E)" ) { set $susquery 1; }
if ( $args ~* "mosConfig_[a-zA-Z_]{1,21}(=|%3D)" ) { set $susquery 1; }
if ( $args ~* "base64_decode\(" ) { set $susquery 1; }
if ( $args ~* "%24&x" ) { set $susquery 1; }
if ( $args ~* "127\.0" ) { set $susquery 1; }
if ( $args ~* "(globals|encode|request|localhost|loopback|request|insert|concat|union|declare)" ) { set $susquery 1; }
if ( $args ~* "%[01][0-9A-F]" ) { set $susquery 1; }
# Whitelist legitimate WordPress query patterns that match the above rules
if ( $args ~ "^loggedout=true" ) { set $susquery 0; }
if ( $args ~ "^action=jetpack-sso" ) { set $susquery 0; }
if ( $args ~ "^action=rp" ) { set $susquery 0; }
if ( $http_cookie ~ "wordpress_logged_in_" ) { set $susquery 0; }
if ( $http_referer ~* "^https?://maps\.googleapis\.com/" ) { set $susquery 0; }
if ( $susquery = 1 ) { return 403; }

# -------------------------------------------------------
# BLOCK COMMON SQL INJECTIONS
# -------------------------------------------------------
set $block_sql_injections 0;
if ($query_string ~ "union.*select.*\(") { set $block_sql_injections 1; }
if ($query_string ~ "union.*all.*select.*") { set $block_sql_injections 1; }
if ($query_string ~ "concat.*\(") { set $block_sql_injections 1; }
if ($block_sql_injections = 1) { return 403; }

# -------------------------------------------------------
# BLOCK FILE INJECTIONS
# -------------------------------------------------------
set $block_file_injections 0;
if ($query_string ~ "[a-zA-Z0-9_]=http://") { set $block_file_injections 1; }
if ($query_string ~ "[a-zA-Z0-9_]=(\.\.//?)+" ) { set $block_file_injections 1; }
if ($query_string ~ "[a-zA-Z0-9_]=/([a-z0-9_.]//?)+") { set $block_file_injections 1; }
if ($block_file_injections = 1) { return 403; }

# -------------------------------------------------------
# BLOCK COMMON EXPLOITS
# -------------------------------------------------------
set $block_common_exploits 0;
if ($query_string ~ "(<|%3C).*script.*(>|%3E)") { set $block_common_exploits 1; }
if ($query_string ~ "GLOBALS(=|\[|\%[0-9A-Z]{0,2})") { set $block_common_exploits 1; }
if ($query_string ~ "_REQUEST(=|\[|\%[0-9A-Z]{0,2})") { set $block_common_exploits 1; }
if ($query_string ~ "proc/self/environ") { set $block_common_exploits 1; }
if ($query_string ~ "mosConfig_[a-zA-Z_]{1,21}(=|\%3D)") { set $block_common_exploits 1; }
if ($query_string ~ "base64_(en|de)code\(.*\)") { set $block_common_exploits 1; }
if ($block_common_exploits = 1) { return 403; }

# -------------------------------------------------------
# BLOCK SPAM QUERY STRINGS
# -------------------------------------------------------
set $block_spam 0;
if ($query_string ~ "\b(ultram|unicauca|valium|viagra|vicodin|xanax|ypxaieo)\b") { set $block_spam 1; }
if ($query_string ~ "\b(erections|hoodia|huronriveracres|impotence|levitra|libido)\b") { set $block_spam 1; }
if ($query_string ~ "\b(ambien|blue\spill|cialis|cocaine|ejaculation|erectile)\b") { set $block_spam 1; }
if ($query_string ~ "\b(lipitor|phentermin|pro[sz]ac|sandyauer|tramadol|troyhamby)\b") { set $block_spam 1; }
if ($block_spam = 1) { return 403; }

# -------------------------------------------------------
# BLOCK KNOWN BAD USER AGENTS (vulnerability scanners)
# -------------------------------------------------------
set $block_user_agents 0;
if ($http_user_agent ~ "Indy Library") { set $block_user_agents 1; }
if ($http_user_agent ~ "libwww-perl") { set $block_user_agents 1; }
if ($http_user_agent ~ "GetRight") { set $block_user_agents 1; }
if ($http_user_agent ~ "dirbuster") { set $block_user_agents 1; }
if ($http_user_agent ~ "nikto") { set $block_user_agents 1; }
if ($http_user_agent ~ "sqlmap") { set $block_user_agents 1; }
if ($http_user_agent ~ "fimap") { set $block_user_agents 1; }
if ($http_user_agent ~ "nessus") { set $block_user_agents 1; }
if ($http_user_agent ~ "whatweb") { set $block_user_agents 1; }
if ($http_user_agent ~ "Openvas") { set $block_user_agents 1; }
if ($http_user_agent ~ "jbrofuzz") { set $block_user_agents 1; }
if ($http_user_agent ~ "libwhisker") { set $block_user_agents 1; }
if ($http_user_agent ~ "webshag") { set $block_user_agents 1; }
if ($http_user_agent ~ "Acunetix-Product") { set $block_user_agents 1; }
if ($http_user_agent ~ "Acunetix") { set $block_user_agents 1; }
if ($block_user_agents = 1) { return 403; }
```

---

## 42.3 Key Points About the Ruleset Design

**`xmlrpc.php` is commented out by default.** If plugins or mobile apps rely on the XML-RPC API, leave it commented. If it is not needed, uncomment to deny all access — it is a common brute-force target.

**The suspicious query string section uses a `set $susquery` variable pattern** rather than direct `return 403` in each `if` block. This is necessary because Nginx's `if` blocks do not chain — the variable approach is the correct way to aggregate multiple conditions in Nginx config. The whitelist entries at the bottom reset `$susquery` to `0` for known-safe WordPress patterns that would otherwise be caught (e.g. Google Maps referrers, logged-in cookie holders, password reset actions).

**The file access location blocks use both exact matches (`=`) and regex matches (`~` and `~*`).** `~*` is case-insensitive — important for file extension matching since some attackers use mixed case extensions like `.PHP` or `.Php` to bypass case-sensitive rules.

**PHP execution is blocked in `uploads/`, `plugins/`, and `themes/` but not in the WordPress root or `wp-admin/`.** The regex covers `php`, `php1` through `php7`, `pht`, `phtml`, and `phps` — all valid PHP file extensions that a web server might execute.

---

## 42.4 Step 2 — Include the File in the Site Server Block

```bash
cd /etc/nginx/sites-available/
sudo nano example.com.conf
```

Add the security directives include alongside `http_headers.conf`, above the PHP processing location block:

```nginx
include /etc/nginx/includes/http_headers.conf;
include /etc/nginx/includes/nginx_security_directives.conf;
```

Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 42.5 Step 3 — Verify the Firewall Rules Are Active

Confirm the file path is correct:

```bash
readlink -f /etc/nginx/includes/nginx_security_directives.conf
# Should print: /etc/nginx/includes/nginx_security_directives.conf
```

Test that sensitive file access is blocked — navigate to `readme.html` in a browser:

```
https://example.com/readme.html  →  403 Forbidden
```

This confirms the location block for readme files is working. The `wp-config.php`, install page, and `wp-includes` PHP file rules can be verified the same way.

---

## 42.6 Step 4 — Handle Plugins That Need PHP Execution in the Plugins Directory

The ruleset blocks all PHP execution inside `wp-content/plugins/`. Some plugins legitimately execute PHP files that are exposed via a direct URL (for example, certain payment gateway callback handlers or license check endpoints).

To allow a specific plugin PHP file to execute while keeping the block in place for everything else, add an exact-match location block **after** the closing brace of the PHP processing block in the site config:

```nginx
# Allow a specific plugin file to execute PHP
location = /wp-content/plugins/some-plugin/callback.php {
    allow all;
    include snippets/fastcgi-php.conf;
    fastcgi_param HTTP_HOST $host;
    fastcgi_pass unix:/run/php/php8.3-fpm-example.com.sock;
    include /etc/nginx/includes/fastcgi_optimize.conf;
}
```

The `location =` exact match takes precedence over the regex deny in the security directives include. Nginx evaluates exact-match location blocks before regex location blocks, so the explicit allow wins.

Test, reload, and verify the file now executes:

```bash
sudo nginx -t
sudo systemctl reload nginx && sudo systemctl restart php8.3-fpm
```

---

## 42.7 Final Site Config Structure

After all includes added across Chapters 41 and 42, the complete site server block:

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

    # Optional: exact-match blocks for specific plugin PHP files
    # location = /wp-content/plugins/some-plugin/callback.php {
    #     allow all;
    #     include snippets/fastcgi-php.conf;
    #     fastcgi_param HTTP_HOST $host;
    #     fastcgi_pass unix:/run/php/php8.3-fpm-example.com.sock;
    #     include /etc/nginx/includes/fastcgi_optimize.conf;
    # }

    include /etc/nginx/includes/browser_caching.conf;

    access_log /var/log/nginx/access.log.example.com.log combined buffer=256k flush=60m;
    error_log /var/log/nginx/error.log.example.com.log;
}
```

---

## 42.8 Summary of Files Modified

| File | Change |
|------|--------|
| `/etc/nginx/includes/nginx_security_directives.conf` | Created — ~100-line WordPress firewall ruleset |
| `/etc/nginx/sites-available/example.com.conf` | Updated — added `include /etc/nginx/includes/nginx_security_directives.conf` |

> **Automation note:** `nginx_security_directives.conf` is a static file with no site-specific values — the same file can be applied to every site on the server. When scripting, write it out once as a heredoc to `/etc/nginx/includes/nginx_security_directives.conf`. Adding it to a new site is then just one `sed` or `tee` operation targeting the site's server block. The plugin-specific whitelist location blocks are the only part that varies per site.

---

| | | |
|:---|:---:|---:|
| [← Chapter 41 — HTTP Response Headers](./41-http-response-headers.md) | [↑ Top](#chapter-42--nginx-wordpress-security-directives) | [Chapter 43 — Nginx DDoS Protection Considerations →](./43-nginx-ddos-protection.md) |
