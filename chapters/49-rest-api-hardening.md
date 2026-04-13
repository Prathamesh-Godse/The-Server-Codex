[← Back to Index](../index.md)

---

# Chapter 49 — Hardening WordPress: Restricting the REST API

*Blocking unauthenticated access to the WordPress REST API — preventing user enumeration and eliminating a common attack surface for plugin vulnerabilities — while keeping full API access for authenticated users.*

---

## Contents

- [49.1 Overview](#491-overview)
- [49.2 The Problem](#492-the-problem)
- [49.3 Solution: Disable REST API Plugin](#493-solution-disable-rest-api-plugin)
- [49.4 Installation](#494-installation)
  - [Step 1 — Install the Plugin](#step-1--install-the-plugin)
  - [Step 2 — Verify the Plugin is Active](#step-2--verify-the-plugin-is-active)
  - [Step 3 — Configure Endpoint Exceptions (Optional)](#step-3--configure-endpoint-exceptions-optional)
- [49.5 Considerations](#495-considerations)

---

## 49.1 Overview

The WordPress REST API is a built-in interface that allows external applications and JavaScript clients to interact with a WordPress site by sending and receiving JSON data over HTTP. While it enables legitimate functionality — the block editor, theme previews, and many plugins depend on it — it is also an increasingly common attack vector. A notable real-world example is the Bricks Builder vulnerability, where attackers exploited unauthenticated REST API access to execute arbitrary code on WordPress sites.

The REST API cannot be completely disabled without breaking core WordPress functionality. The approach taken here is to restrict unauthenticated access to it — allowing authenticated (logged-in) users full access while blocking all REST API endpoints from the public.

---

## 49.2 The Problem

By default, the WordPress REST API is publicly accessible. Any visitor, including automated bots and attackers, can query endpoints such as:

```
https://example.com/wp-json/wp/v2/users
https://example.com/wp-json/wp/v2/posts
https://example.com/wp-json/
```

The user enumeration endpoint (`/wp/v2/users`) in particular leaks WordPress usernames — the same usernames that are then targeted in brute-force attacks against `wp-login.php`. Beyond enumeration, plugin-specific REST endpoints have historically been the entry point for remote code execution vulnerabilities when left exposed to unauthenticated requests.

---

## 49.3 Solution: Disable REST API Plugin

The WordPress core itself provides no clean toggle to restrict REST API access by authentication status. The recommended approach is to install the **Disable REST API** plugin by Dave McHale, which has 90,000+ active installations and a five-star rating.

Once activated, the plugin blocks all REST API endpoints from unauthenticated (not logged in) users by default. Any request to the REST API from a visitor who is not authenticated receives:

```json
{
  "code": "rest_cannot_access",
  "message": "DRA: Only authenticated users can access the REST API.",
  "data": {
    "status": 401
  }
}
```

Authenticated users (logged-in admins, editors, etc.) retain full REST API access, so the WordPress admin dashboard and all authenticated functionality continue working normally.

---

## 49.4 Installation

### Step 1 — Install the Plugin

Navigate to **Plugins → Add New Plugin** in the WordPress admin dashboard. Search for `disable rest` and locate **Disable REST API** by Dave McHale in the search results. Click **Install Now**, then **Activate**.

Alternatively, install directly from:
`https://wordpress.org/plugins/disable-rest-api/`

### Step 2 — Verify the Plugin is Active

Once activated, the plugin works immediately without any required configuration. To verify it is working, visit the REST API root in a browser while logged out:

```
https://example.com/wp-json/
```

A correctly configured site returns a `401` error with the `rest_cannot_access` message. If the API returns a JSON object listing all endpoints, the plugin is not active.

### Step 3 — Configure Endpoint Exceptions (Optional)

The plugin settings page is accessible under **Settings → Disable REST API** in the WordPress admin.

The default configuration restricts all endpoints for unauthenticated users. The settings page lists every registered REST API endpoint grouped by namespace (e.g., `/wp/v2`, `/oembed/1.0`). Each endpoint can be individually toggled back on if a specific use case requires it — for example, a contact form plugin that relies on an unauthenticated REST route for form submission.

The rules dropdown also allows separate configurations for different user roles. By default:

- **Unauthenticated Users** — all endpoints blocked
- **Subscribers, Editors, Admins** — full access (all defined user roles retain access by default until explicitly changed)

> **Important:** Always research and test plugin behaviour on a development server before deploying to production. Some page builders, caching plugins, and third-party integrations rely on specific REST endpoints being publicly accessible.

---

## 49.5 Considerations

**This plugin does not eliminate REST API attack surface entirely.** It prevents unauthenticated access, which blocks the most common automated scanning and exploitation attempts. Authenticated REST API requests are still possible — any logged-in user with appropriate capabilities can interact with the API. Proper role assignment and strong admin passwords remain important.

**WooCommerce sites** require REST API access for several storefront operations. Test thoroughly before enabling this plugin on a WooCommerce installation.

**Page builders** (including Bricks, Elementor, Divi) may use REST endpoints for live preview and editor communication. Most of these only need the API accessible to logged-in users, so the default plugin configuration is generally compatible. Verify after activation that the page builder editor functions correctly.

**REST API and `wp-cron`** — WordPress's scheduled task system sometimes triggers REST requests internally. This is typically authenticated server-to-server traffic and is not affected by the plugin.

---

| | | |
|:---|:---:|---:|
| [← Chapter 48 — Database Privilege Hardening](./48-database-hardening.md) | [↑ Top](#chapter-49--hardening-wordpress-restricting-the-rest-api) | [↑ Back to Index](../index.md) |
