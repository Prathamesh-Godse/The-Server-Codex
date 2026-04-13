[← Back to Index](../index.md)

---

# Chapter 45 — Web Application Firewall: NinjaFirewall

*Installing NinjaFirewall in Full WAF mode so it intercepts every PHP request before WordPress loads — providing application-layer protection that no Nginx rule or plugin can match.*

---

## Contents

- [45.1 Overview](#451-overview)
- [45.2 Verifying the WordPress Installation Structure](#452-verifying-the-wordpress-installation-structure)
- [45.3 Installing NinjaFirewall](#453-installing-ninjafirewall)
- [45.4 Initial Dashboard State](#454-initial-dashboard-state)
- [45.5 Activating Full WAF Mode](#455-activating-full-waf-mode)
  - [Configuration Choices](#configuration-choices)
- [45.6 Result](#456-result)
- [45.7 Notes](#457-notes)

---

## 45.1 Overview

A Web Application Firewall (WAF) sits in front of the application and intercepts every incoming HTTP/HTTPS request before it reaches WordPress or any of its plugins. Based on a defined ruleset, the WAF either allows or denies the request. This is fundamentally different from a network-level firewall — it understands application-layer traffic and can detect and block SQL injection, file inclusion attacks, malicious query strings, and exploit attempts.

The WAF chosen for this setup is **NinjaFirewall (WP Edition)**. Unlike typical security plugins that hook into WordPress after it has already loaded, NinjaFirewall loads *before* WordPress, positioning itself as a true stand-alone firewall. It can hook, scan, sanitise, or reject any request sent to a PHP script before WordPress or any plugin processes it. This includes encoded PHP scripts, shell scripts, and backdoors.

- Plugin page: [https://wordpress.org/plugins/ninjafirewall/](https://wordpress.org/plugins/ninjafirewall/)
- Fast, low-footprint, and requires minimal system resources.

---

## 45.2 Verifying the WordPress Installation Structure

Before installing the plugin, confirm the WordPress `public_html` directory is intact and owned by the correct pool user:

```bash
cd /var/www/expertwp.help/public_html/
ls -la
```

Expected output confirms all core WordPress files and directories (`wp-admin`, `wp-content`, `wp-includes`, `wp-config.php`, `wp-login.php`, `xmlrpc.php`, etc.) are present and owned by the site's dedicated pool user (`expertwp:expertwp`), with correct permission modes (`rw-rw----`).

> The pool user ownership model ensures that NinjaFirewall's Full WAF mode can protect files at the filesystem level, isolating the site from the server OS and from other hosted sites.

---

## 45.3 Installing NinjaFirewall

NinjaFirewall is installed directly from the WordPress plugin repository via the admin dashboard.

**Path:** WordPress Admin → Plugins → Add New Plugin → Search: `ninja firewall`

From the search results, locate **NinjaFirewall (WP Edition) – Advanced Security Plugin and Firewall** by The Ninja Technologies Network and click **Install Now**, then **Activate**.

Once activated, the NinjaFirewall menu appears in the left sidebar of the WordPress admin panel.

---

## 45.4 Initial Dashboard State

After activation, navigating to **NinjaFirewall → Dashboard** shows:

| Field | Value |
|-------|-------|
| Firewall | Enabled |
| Mode | WordPress WAF (default) |
| Edition | WP Edition |
| Version | 4.5.11 – Security rules: 2024-07-01.1 |
| PHP SAPI | FPM-FCGI ~8.3.9 |

By default, NinjaFirewall runs in **WordPress WAF mode**, which hooks into WordPress after it loads. For maximum protection, **Full WAF mode** is the correct configuration for a dedicated VPS running Nginx + PHP-FPM.

---

## 45.5 Activating Full WAF Mode

Full WAF mode protects all scripts located inside the WordPress installation directory and all sub-directories — including scripts that are not part of the WordPress package. Even encoded PHP scripts, backdoors, and shell scripts get intercepted and filtered before execution.

Click **Activate Full WAF mode** on the dashboard.

### Configuration Choices

| Setting | Selected Value |
|---------|---------------|
| HTTP server / PHP server API | Nginx + CGI/FastCGI or PHP-FPM (recommended) |
| PHP initialization file | `.user.ini` (recommended) |
| Folders protected | `wp-admin/`, `wp-content/`, `wp-includes/` |
| Change management | Let NinjaFirewall make the necessary changes (recommended) |
| Sandbox mode | Enabled |

**Why `.user.ini` over `php.ini`?**
PHP-FPM respects `.user.ini` files placed in the document root. NinjaFirewall writes its bootstrap loader directive into this file so it is loaded before any PHP script executes. This is what enables it to intercept requests before WordPress initialises.

**Why sandbox mode?**
If NinjaFirewall encounters a problem during the Full WAF installation (e.g. a write permission issue), sandbox mode causes it to automatically roll back all changes, leaving the site in its previous working state.

Click **Finish Installation**.

---

## 45.6 Result

After completing the Full WAF mode installation, NinjaFirewall is now operating as a true Web Application Firewall — intercepting all PHP requests at the `public_html` level before they reach WordPress core. The firewall ruleset covers:

- Malicious query strings and URL patterns
- SQL injection attempts
- File injection and path traversal attacks
- Common exploits (XSS, GLOBALS tampering, base64-encoded payloads)
- Suspicious HTTP methods (`TRACE`, `DELETE`, `TRACK`)
- Known scanner and exploit tool user agents (`nikto`, `sqlmap`, `acunetix`, `nmap`, etc.)
- Spam query strings

The WordPress admin dashboard and all three critical directories (`wp-admin/`, `wp-content/`, `wp-includes/`) are actively protected.

---

## 45.7 Notes

- NinjaFirewall does not require any additional server-side configuration for Nginx — it operates entirely at the PHP level via `.user.ini`.
- The WP Edition is free. A premium WP+ Edition exists with extended ruleset coverage; the WP Edition is sufficient for most production sites.
- Firewall rules update independently of the plugin version. Check the **Security Rules** date on the dashboard periodically.
- The **Monitoring**, **Logs**, and **Event Notifications** sections in the NinjaFirewall sidebar can be configured to alert on blocked requests, which is useful for ongoing threat visibility.

---

| | | |
|:---|:---:|---:|
| [← Chapter 44 — Nginx Rate Limiting](./44-nginx-rate-limiting.md) | [↑ Top](#chapter-45--web-application-firewall-ninjafirewall) | [↑ Back to Index](../index.md) |
