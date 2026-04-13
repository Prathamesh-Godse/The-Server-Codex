[← Back to Index](../index.md)

---

# Chapter 43 — Nginx DDoS Protection: Considerations for WordPress

*Why Nginx-level DDoS mitigation is not the right tool for WordPress deployments, and where protection should actually be applied — at the CDN or network-perimeter level.*

---

## Contents

- [43.1 Why Nginx DDoS Protection Falls Short for WordPress](#431-why-nginx-ddos-protection-falls-short-for-wordpress)
- [43.2 Recommended Approach](#432-recommended-approach)
- [43.3 What Is Covered in This Project Instead](#433-what-is-covered-in-this-project-instead)

---

## 43.1 Why Nginx DDoS Protection Falls Short for WordPress

Nginx's rate limiting works by capping incoming requests to a configured threshold that is intended to reflect the expected load from real users. The core problem with this approach is that it is extremely difficult to estimate and set the right values correctly for a WordPress site.

WordPress generates a wide variety of legitimate request patterns:

- Authenticated admin and editor sessions hit different endpoints than public visitors
- Plugins and themes may make backend requests that look similar to bot traffic
- REST API calls, XML-RPC, and cron requests all contribute to the request volume

Because of this variability, setting a rate limit that is both low enough to stop an attack and high enough to not block real users or plugin functionality becomes impractical. A misconfigured rate limit can break site functionality, block legitimate plugin execution, or reject authenticated user requests — all while still not providing meaningful protection against a real DDoS event.

Additionally, for DDoS protection to be effective at any meaningful scale, it needs to be enforced at a level above the web server — at the network edge or CDN layer — before traffic ever reaches the Nginx process. By the time a volumetric attack reaches the server, Nginx is already under resource pressure.

---

## 43.2 Recommended Approach

For sites that face or are at risk of DDoS attacks, the appropriate solution is to move protection upstream:

- **Enable Cloudflare** in front of the server. Cloudflare absorbs volumetric attacks at the network edge, before malicious traffic reaches the VPS. It also provides bot management, rate limiting rules, and challenge pages that are far better suited to WordPress traffic patterns.
- **Contact the hosting provider** to enable DDoS protection at the infrastructure level. Many providers offer network-level scrubbing that filters traffic before it reaches the server.

These options operate at a scale and at a layer where DDoS protection is actually effective.

---

## 43.3 What Is Covered in This Project Instead

Rather than implementing Nginx-level DDoS rate limiting (which carries a high risk of false positives and site breakage on WordPress), the hardening work in this project focuses on targeted, low-risk rate limiting for specific high-risk endpoints — namely `wp-login.php` and `xmlrpc.php` — which is covered in Chapter 44. This is distinct from DDoS mitigation and is a focused, surgical measure rather than a broad traffic cap.

> **Note for automation:** This chapter requires no scripted commands. The decision here is architectural — handle DDoS at the CDN or host level, not at Nginx. Any automation script should document this assumption and leave a placeholder comment directing the operator to configure Cloudflare or equivalent upstream protection.

---

| | | |
|:---|:---:|---:|
| [← Chapter 42 — Nginx WordPress Security Directives](./42-nginx-wordpress-security-directives.md) | [↑ Top](#chapter-43--nginx-ddos-protection-considerations-for-wordpress) | [Chapter 44 — Nginx Rate Limiting for WordPress →](./44-nginx-rate-limiting.md) |
