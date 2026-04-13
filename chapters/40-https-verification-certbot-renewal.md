[← Back to Index](../index.md)

---

# Chapter 40 — HTTPS Verification, Fixing Mixed Content & Automating SSL Renewal

*Verifying HTTPS with `curl`, confirming SSL and HTTP/3 scores externally, fixing WordPress mixed content warnings by updating internal URLs to HTTPS, and automating certificate renewal with a root cron job.*

---

## Contents

- [40.1 Step 1 — Verify Redirects and Response Headers with curl](#401-step-1--verify-redirects-and-response-headers-with-curl)
- [40.2 Step 2 — Verify SSL Rating at SSL Labs](#402-step-2--verify-ssl-rating-at-ssl-labs)
- [40.3 Step 3 — Verify HTTP/3 at http3check.net](#403-step-3--verify-http3-at-http3checknet)
- [40.4 Step 4 — Fix Mixed Content by Updating WordPress URLs to HTTPS](#404-step-4--fix-mixed-content-by-updating-wordpress-urls-to-https)
- [40.5 Step 5 — Certbot Management Commands](#405-step-5--certbot-management-commands)
- [40.6 Step 6 — Automate Certificate Renewal with a Root Cron Job](#406-step-6--automate-certificate-renewal-with-a-root-cron-job)

---

## 40.1 Step 1 — Verify Redirects and Response Headers with `curl`

With the site config saved and Nginx reloaded, use `curl -I` to inspect the response headers for all four URL combinations. This confirms that HTTP redirects correctly, HTTPS responds with HTTP/2, and that HSTS and HTTP/3 advertisement headers are present.

```bash
curl -I http://example.com
curl -I http://www.example.com
curl -I https://www.example.com
curl -I https://example.com
```

**Expected output for HTTP requests (`http://` and `http://www.`):**

```
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
```

Both bare and `www` HTTP requests should return a `301` pointing to the canonical HTTPS URL. This confirms the redirect-only server block is working.

**Expected output for `https://www.example.com`:**

```
HTTP/2 301
location: https://example.com/
x-redirect-by: WordPress
strict-transport-security: max-age=31536000;
alt-svc: h3=":443"; ma=86400
x-quic: H3
```

The `www` HTTPS request returns a `301` to the non-`www` canonical URL — this secondary redirect is issued by WordPress based on the Site Address configured during installation, not by Nginx directly. The `strict-transport-security`, `alt-svc`, and `x-quic` headers confirm HSTS and HTTP/3 advertisement are active.

**Expected output for `https://example.com`:**

```
HTTP/2 200
strict-transport-security: max-age=31536000;
alt-svc: h3=":443"; ma=86400
x-quic: H3
```

A `200` over HTTP/2 with the HSTS and HTTP/3 headers present means the configuration is correct end to end.

---

## 40.2 Step 2 — Verify SSL Rating at SSL Labs

Navigate to [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/), enter the domain name, and tick "Do not show the results on the boards" before submitting.

The cipher suite and TLS configuration in `ssl_all_sites.conf` (documented in Chapter 39) is specifically tuned to achieve an **A+ rating**. The summary page confirms:

- Certificate: valid, correctly chained
- Protocol Support: TLS 1.2 and TLS 1.3 only
- Key Exchange: strong (ECDSA + DHE)
- Cipher Strength: full score
- HTTP Strict Transport Security (HSTS) with long duration deployed

---

## 40.3 Step 3 — Verify HTTP/3 at http3check.net

Navigate to [https://http3check.net/](https://http3check.net/) and enter the domain. A successful result shows:

```
✓ QUIC is supported
✓ HTTP/3 is supported
```

The tool establishes an actual QUIC connection to the server and reports connection IDs, packets received, and handshake timing.

**In-browser verification via DevTools:**

1. Open Chrome DevTools → Network tab.
2. Hard-refresh the page (`Ctrl+Shift+R`).
3. Right-click any column header in the request list and enable the **Protocol** column.
4. Resources served over HTTP/3 show `h3` in the Protocol column.

---

## 40.4 Step 4 — Fix Mixed Content by Updating WordPress URLs to HTTPS

After enabling HTTPS, the WordPress database still holds `http://` in its WordPress Address and Site Address fields. This causes **mixed content warnings** — the page loads over HTTPS but WordPress generates internal links and asset URLs using the old HTTP scheme, so browsers flag the page as partially insecure.

Fix this in the WordPress dashboard at **Settings → General**.

Change both fields from `http://` to `https://`:

| Field | Before | After |
|-------|--------|-------|
| WordPress Address (URL) | `http://example.com` | `https://example.com` |
| Site Address (URL) | `http://example.com` | `https://example.com` |

Scroll to the bottom and click **Save Changes**. WordPress will write the updated URLs to `wp_options` in the database and use them for all subsequent URL generation.

> **Note:** The URL in these two fields must exactly match the redirect target in the Nginx HTTP server block. If WordPress was installed at `https://www.example.com`, use `www` in both fields and ensure the Nginx redirect on port 80 points to `https://www.example.com$request_uri`.

After saving, WordPress logs you out and redirects back to the admin login page over HTTPS, confirming the update took effect.

---

## 40.5 Step 5 — Certbot Management Commands

Before setting up automated renewal, these are the key Certbot management commands.

**List all installed certificates and their details:**

```bash
sudo certbot certificates
```

Output shows the certificate name, serial number, key type, covered domains, expiry date, and paths to the fullchain and private key files:

```
Found the following certs:
  Certificate Name: example.com
    Serial Number: 4c93c06569ccdb4a76acc56f730eb09db9a
    Key Type: ECDSA
    Domains: example.com www.example.com
    Expiry Date: 2024-10-22 16:54:33+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/example.com/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/example.com/privkey.pem
```

**Delete a certificate interactively:**

```bash
sudo certbot delete
```

Certbot lists installed certificates and prompts for a selection. Enter `c` to cancel without deleting.

**Test that renewal will work without actually renewing:**

```bash
sudo certbot renew --dry-run
```

This simulates the full renewal process against Let's Encrypt's staging environment. A successful run outputs:

```
Simulating renewal of an existing certificate for example.com and www.example.com
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/example.com/fullchain.pem (success)
```

Always run this before setting up automated renewal to confirm the webroot path and domain configuration are correct.

**Force a renewal dry-run (simulates even if the cert is not near expiry):**

```bash
sudo certbot renew --force-renewal --dry-run
```

---

## 40.6 Step 6 — Automate Certificate Renewal with a Root Cron Job

Let's Encrypt certificates are valid for **90 days**. The renewal strategy here is to force-renew all certificates on the 14th and 28th of every month, then reload Nginx one hour later so the new certificates are picked up. Running renewal twice a month means the certificate is always refreshed well before the 90-day expiry.

The cron must run as root because `certbot` requires root to write the renewed certificate files, and `systemctl reload nginx` requires root privileges.

Open the root crontab:

```bash
sudo crontab -e
```

On first run, the system prompts for an editor — select `1` for nano.

Add the following two entries at the bottom of the crontab file:

```cron
# m  h  dom mon dow   command
00   1  14,28  *  *   certbot renew --force-renewal >/dev/null 2>&1
00   2  14,28  *  *   systemctl reload nginx >/dev/null 2>&1
```

Cron field breakdown:

| Field | Value | Meaning |
|-------|-------|---------|
| Minute | `00` | At the top of the hour |
| Hour | `1` / `2` | 1:00 AM for renewal, 2:00 AM for reload |
| Day of month | `14,28` | 14th and 28th of each month |
| Month | `*` | Every month |
| Day of week | `*` | Any day |

The one-hour gap between renewal (1:00 AM) and the Nginx reload (2:00 AM) gives Certbot time to complete the renewal process for all certificates before Nginx is asked to pick them up.

`>/dev/null 2>&1` suppresses both stdout and stderr output. If output is needed for debugging, redirect to a log file instead:

```cron
00 1 14,28 * * certbot renew --force-renewal >> /var/log/certbot-renew.log 2>&1
00 2 14,28 * * systemctl reload nginx >> /var/log/certbot-renew.log 2>&1
```

Save and exit nano (`Ctrl+O`, `Enter`, `Ctrl+X`). The cron daemon picks up the changes immediately — no restart required.

> **Important:** Set a personal reminder for the day after the first scheduled renewal to run `sudo certbot certificates` and verify the expiry date has advanced. Do this check for the first two or three cycles until confident the automation is working reliably.

### Non-Interactive Cron Setup (for automation scripts)

The cron entries can be written non-interactively using a pipeline:

```bash
(sudo crontab -l 2>/dev/null; echo "00 1 14,28 * * certbot renew --force-renewal >/dev/null 2>&1"; echo "00 2 14,28 * * systemctl reload nginx >/dev/null 2>&1") | sudo crontab -
```

This appends to any existing root crontab entries rather than overwriting them.

> **WP-CLI note for automation:** The WordPress URL update cannot be done via the dashboard in a script. Use WP-CLI instead: `wp option update siteurl 'https://example.com'` and `wp option update home 'https://example.com'`. Run as the site's pool user or with `--allow-root` if running as root.

---

| | | |
|:---|:---:|---:|
| [← Chapter 39 — SSL/TLS Certificates](./39-ssl-certificates.md) | [↑ Top](#chapter-40--https-verification-fixing-mixed-content--automating-ssl-renewal) | [↑ Back to Index](../index.md) |
