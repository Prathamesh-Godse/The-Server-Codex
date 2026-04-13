[← Back to Index](../index.md)

---

# Chapter 46 — Protecting Site Assets with Hotlinking Protection

*Preventing external sites from embedding your images and consuming your server bandwidth — comparing Cloudflare's one-click toggle against the Nginx `valid_referers` approach, and when to use each.*

---

## Contents

- [46.1 Overview](#461-overview)
- [46.2 Option 1: Cloudflare Hotlink Protection](#462-option-1-cloudflare-hotlink-protection)
  - [How It Works](#how-it-works)
  - [Enabling in the Cloudflare Dashboard](#enabling-in-the-cloudflare-dashboard)
  - [Allowing Hotlinking for Specific Images](#allowing-hotlinking-for-specific-images)
- [46.3 Option 2: Nginx Hotlink Protection](#463-option-2-nginx-hotlink-protection)
  - [Considerations](#considerations)
  - [Configuration](#configuration)
- [46.4 Summary](#464-summary)

---

## 46.1 Overview

Hotlinking (also called inline linking) is when an external website embeds images or other media files directly from your server by referencing your URLs in their HTML. The image appears on their site, but every request for it consumes bandwidth from your server. At scale, this can cause a measurable increase in monthly hosting costs and degrade server performance for your own visitors.

Two approaches are available:

**Cloudflare Hotlink Protection** — Simple, one-click toggle in the Cloudflare dashboard. Operates at the CDN/proxy layer, so requests are intercepted before they ever reach the origin server. This is the preferred approach when the site is already behind Cloudflare.

**Nginx Hotlink Protection** — Configured directly in the Nginx server block using the `$http_referer` variable. Offers more granular control over which file types, paths, and referring domains are allowed or denied. However, it will not function correctly if Cloudflare is in front of the server (since Cloudflare's IPs become the apparent referer).

> **Rule of thumb:** If Cloudflare is set up and active on the domain, use Cloudflare's built-in hotlink protection. Only implement the Nginx-level approach if Cloudflare is not in use.

---

## 46.2 Option 1: Cloudflare Hotlink Protection

### How It Works

When Cloudflare receives an image request destined for your site, it checks whether the HTTP `Referer` header originates from your own domain. If the referer is an external site attempting to embed your image, the request is denied. Direct visitors to your domain can still view and download images normally.

Covered file types: `gif`, `ico`, `jpg`, `jpeg`, `png`

> **Important:** Hotlink protection will prevent images from appearing on aggregator sites like Google Images and Pinterest. If search visibility for images is a priority, factor this in before enabling.

### Enabling in the Cloudflare Dashboard

**Path:** Cloudflare Dashboard → Select Domain → Scrape Shield → Hotlink Protection → Toggle ON

No additional configuration is needed. The toggle takes effect immediately across Cloudflare's network.

The **Scrape Shield** section also contains:

| Feature | Description | Status |
|---------|-------------|--------|
| Email Address Obfuscation | Masks email addresses from bot scrapers while keeping them visible to real visitors | Enabled |
| Server-side Excludes | Hides specific content from disreputable visitors (deprecated as of 2024-06-14) | Enabled |
| Hotlink Protection | Prevents off-site linking to your images | Enabled |

### Allowing Hotlinking for Specific Images

If certain images need to be hotlink-accessible (e.g. a logo used by partner sites), place them in a directory named `hotlink-ok` anywhere within the site's directory structure. Cloudflare will skip hotlink checks for any image served from a path containing `hotlink-ok`.

Example paths that bypass hotlink protection:

```
https://example.com/hotlink-ok/logo.png
https://example.com/images/hotlink-ok/banner.png
https://example.com/wp-content/uploads/hotlink-ok/press-kit.jpg
```

---

## 46.3 Option 2: Nginx Hotlink Protection

### Considerations

Nginx-level hotlinking protection uses the `$http_referer` variable in the server block to check the origin of image requests. It is more configurable than the Cloudflare toggle — you can restrict specific file extensions, allow certain external domains (e.g. Google, social media crawlers), and customise the response code or replacement image.

**However**, this approach has a critical constraint: **it will not work correctly if Cloudflare is enabled**. Because all traffic arrives at the origin through Cloudflare's proxy, the `$http_referer` seen by Nginx reflects Cloudflare's infrastructure rather than the actual external site that embedded the image. This makes referer-based blocking unreliable.

Additional trade-offs compared to the Cloudflare approach:

- Requires disabling Cloudflare proxy (grey cloud in DNS), which removes CDN caching and DDoS mitigation benefits
- Processing referer checks on every image request adds marginal CPU overhead to the origin server
- Configuration errors can accidentally block legitimate traffic (e.g. social media previews, search engine crawlers)

### Configuration

The Nginx hotlinking protection block is added inside the site's server context:

```nginx
# /etc/nginx/sites-available/example.com.conf

location ~* \.(gif|ico|jpg|jpeg|png|webp|svg)$ {
    valid_referers none blocked server_names
                   ~\.example\.com
                   ~\.google\.
                   ~\.bing\.
                   ~\.facebook\.
                   ~\.twitter\.
                   ~\.pinterest\.;

    if ($invalid_referer) {
        return 403;
    }

    # Standard static asset caching
    expires 30d;
    add_header Cache-Control "public, no-transform";
    access_log off;
}
```

**Directive breakdown:**

| Directive | Purpose |
|-----------|---------|
| `valid_referers none` | Allows direct requests with no referer (e.g. typed URLs, bookmarks) |
| `valid_referers blocked` | Allows requests where the referer header is present but has been stripped by a firewall or proxy |
| `server_names` | Allows requests originating from the site itself |
| `~\.google\.` | Regex match — allows Google crawlers and image search |
| `if ($invalid_referer)` | Fires if the referer does not match any entry in `valid_referers` |
| `return 403` | Denies the request with a Forbidden response |

After editing, validate and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

> To return a placeholder image instead of a 403, replace `return 403;` with `rewrite ^ /images/hotlink-blocked.png break;` and place a suitable image at that path.

---

## 46.4 Summary

| | Cloudflare | Nginx |
|-|-----------|-------|
| Setup complexity | One toggle | Manual configuration required |
| Works with Cloudflare proxy | Yes | No |
| Granular control | Limited | High |
| Performance impact on origin | None | Marginal |
| Recommended when | Cloudflare is active | Cloudflare is not in use |

---

| | | |
|:---|:---:|---:|
| [← Chapter 45 — Web Application Firewall](./45-web-application-firewall.md) | [↑ Top](#chapter-46--protecting-site-assets-with-hotlinking-protection) | [Chapter 47 — Disabling File Modifications →](./47-disallow-file-mods.md) |
