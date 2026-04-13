[← Back to Index](../index.md)

---

# Chapter 39 — SSL/TLS Certificates: Certbot, Let's Encrypt & Nginx HTTPS Configuration

*Obtaining free SSL/TLS certificates from Let's Encrypt with Certbot, generating DH parameters, creating modular SSL include files, and updating the Nginx server block to serve HTTPS with HTTP/2 and HTTP/3.*

---

## Contents

- [39.1 Overview](#391-overview)
- [39.2 Step 1 — Update the System](#392-step-1--update-the-system)
- [39.3 Step 2 — Install Certbot and the Cloudflare DNS Plugin](#393-step-2--install-certbot-and-the-cloudflare-dns-plugin)
- [39.4 Step 3 — Obtain the SSL Certificate](#394-step-3--obtain-the-ssl-certificate)
- [39.5 Step 4 — Create the nginx SSL Directory and Generate DH Parameters](#395-step-4--create-the-nginx-ssl-directory-and-generate-dh-parameters)
  - [4a — Create the ssl/ Directory](#4a--create-the-ssl-directory)
  - [4b — Generate the Diffie-Hellman Parameters File](#4b--generate-the-diffie-hellman-parameters-file)
- [39.6 Step 5 — Create the Site-Specific SSL Include File](#396-step-5--create-the-site-specific-ssl-include-file)
- [39.7 Step 6 — Create the General SSL Include File (All Sites)](#397-step-6--create-the-general-ssl-include-file-all-sites)
- [39.8 Step 7 — Update the Nginx Server Block for HTTPS](#398-step-7--update-the-nginx-server-block-for-https)
  - [HTTP → HTTPS Redirect Block](#http--https-redirect-block)
  - [Updated HTTPS Server Block — Listen Directives](#updated-https-server-block--listen-directives)
  - [Include the SSL Files](#include-the-ssl-files)
  - [Test and Reload](#test-and-reload)
- [39.9 Summary](#399-summary)

---

## 39.1 Overview

SSL/TLS encrypts the connection between the server and a visitor's browser, ensuring that traffic cannot be intercepted in transit. Beyond security, it is now a baseline requirement for trust (browsers flag HTTP sites as insecure), SEO (Google ranks HTTPS sites higher), and compliance.

The approach here uses **Certbot** to obtain free certificates from **Let's Encrypt** via the webroot validation method. Let's Encrypt certificates are valid for 90 days, so an automated renewal process (covered in Chapter 40) is essential.

What gets configured in this chapter:

- System update before installing Certbot
- Certbot and the Cloudflare DNS plugin installation
- Certificate issuance via webroot challenge
- A Diffie-Hellman parameters file for stronger key exchange
- A site-specific SSL include file (cert paths, unique per site)
- A shared SSL include file (protocols, ciphers, HSTS, HTTP/3 — reused by all sites)
- HTTP → HTTPS redirect and updated Nginx server block

> **Prerequisite:** The domain's **A record** (and `www` CNAME) must already be resolving to the server's public IP before running Certbot. The webroot challenge works by placing a temporary file inside the site's document root and having Let's Encrypt fetch it over HTTP — DNS must be correct for this to succeed.

---

## 39.2 Step 1 — Update the System

Always update the package index and apply any pending upgrades before installing new software:

```bash
sudo apt update
sudo apt upgrade
```

---

## 39.3 Step 2 — Install Certbot and the Cloudflare DNS Plugin

Install both Certbot and the Cloudflare DNS plugin in a single command. The Cloudflare plugin is included here so it is available when wildcard certificates are needed later (wildcard issuance requires DNS-01 validation rather than webroot):

```bash
sudo apt install certbot python3-certbot-dns-cloudflare
```

---

## 39.4 Step 3 — Obtain the SSL Certificate

Issue a certificate for both the bare domain and the `www` subdomain using the **webroot** plugin. The `-w` flag points Certbot at the site's document root so it can place the ACME challenge file where Nginx can serve it over HTTP:

```bash
sudo certbot certonly --webroot -w /var/www/expertwp.help/public_html/ \
  -d expertwp.help \
  -d www.expertwp.help
```

Certbot's interactive prompts during first run:

1. **Email address** — Enter a valid address for urgent renewal and security notices.
2. **Terms of Service** — Enter `y` to agree.
3. **EFF mailing list** — Enter `n` to decline (optional).

On success, Certbot confirms the certificate paths:

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/expertwp.help/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/expertwp.help/privkey.pem
This certificate expires on 2024-10-22.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.
```

The certificate files land at:

| File | Purpose |
|------|---------|
| `fullchain.pem` | The site certificate plus the CA chain — this is what Nginx serves to browsers |
| `privkey.pem` | The private key — kept private, never shared |
| `chain.pem` | The intermediate CA chain only — used for OCSP stapling |

---

## 39.5 Step 4 — Create the nginx SSL Directory and Generate DH Parameters

All SSL include files and the DH parameters file are stored in `/etc/nginx/ssl/` to keep them separate from site configurations.

### 4a — Create the `ssl/` Directory

```bash
cd /etc/nginx/
sudo mkdir ssl/
cd ssl/
```

### 4b — Generate the Diffie-Hellman Parameters File

The DH parameters file (`dhparam.pem`) defines the parameters used by OpenSSL during the Diffie-Hellman key exchange step of the TLS handshake. Using a server-generated file rather than the OpenSSL defaults adds an additional layer of security.

```bash
sudo openssl dhparam -out dhparam.pem 2048
```

This command takes several minutes to complete — it is computing a 2048-bit safe prime. **Do not interrupt it.** The terminal will display a stream of dots and `+` characters as it works.

```bash
ls
# dhparam.pem
```

---

## 39.6 Step 5 — Create the Site-Specific SSL Include File

This file holds the absolute paths to the three certificate files issued by Certbot for this particular domain. It is unique to each site — every additional domain hosted on the server gets its own copy pointing to its own certificate directory.

```bash
sudo nano /etc/nginx/ssl/ssl_expertwp.help.conf
```

Contents:

```nginx
ssl_certificate     /etc/letsencrypt/live/expertwp.help/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/expertwp.help/privkey.pem;
ssl_trusted_certificate /etc/letsencrypt/live/expertwp.help/chain.pem;
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 39.7 Step 6 — Create the General SSL Include File (All Sites)

This file contains all the SSL directives that apply to every site on the server — session caching, protocol versions, cipher suites, HSTS, and HTTP/3 settings. It is written once and included in every site's Nginx configuration.

```bash
sudo nano /etc/nginx/ssl/ssl_all_sites.conf
```

Contents:

```nginx
# CONFIGURATION RESULTS IN AN A+ RATING AT SSLLABS.COM
# DATE: MAY 2025

# SSL SESSION CACHING AND PROTOCOLS
ssl_session_cache   shared:SSL:20m;
ssl_session_timeout 180m;
ssl_protocols       TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;

# ssl_ciphers must be on a single line — do not split over multiple lines
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;

ssl_dhparam /etc/nginx/ssl/dhparam.pem;

# NOTE: Let's Encrypt removed OCSP stapling support on 7 May 2025.
# If using Let's Encrypt certificates, remove or comment both ssl_stapling lines.
#ssl_stapling        on;
#ssl_stapling_verify on;

# Resolver set to Cloudflare — timeout up to 30s
resolver         1.1.1.1 1.0.0.1;
resolver_timeout 15s;

ssl_session_tickets off;

# HSTS — force browsers to use HTTPS for one year
add_header Strict-Transport-Security "max-age=31536000;" always;
# Once ALL subdomains are on HTTPS, switch to the line below instead:
# add_header Strict-Transport-Security "max-age=31536000; includeSubDomains;" always;

# Enable QUIC and HTTP/3
ssl_early_data on;
add_header Alt-Svc 'h3=":$server_port"; ma=86400';
add_header x-quic 'H3';
quic_retry on;
```

Save and exit.

**Key directives explained:**

| Directive | Purpose |
|-----------|---------|
| `ssl_session_cache shared:SSL:20m` | Stores SSL session data in a 20 MB shared cache across all worker processes, avoiding full handshakes on repeat connections |
| `ssl_session_timeout 180m` | Sessions remain in cache for 3 hours |
| `ssl_protocols TLSv1.2 TLSv1.3` | Disables all older, insecure protocol versions (SSLv3, TLS 1.0, TLS 1.1) |
| `ssl_prefer_server_ciphers on` | The server's cipher preference order is used rather than the client's |
| `ssl_ciphers` | A curated, ordered list of AEAD cipher suites — stronger/faster ones come first |
| `ssl_dhparam` | Points to the DH parameters file generated in Step 4b |
| `ssl_session_tickets off` | Disables session tickets to preserve forward secrecy |
| `Strict-Transport-Security` | HSTS header tells browsers to only connect via HTTPS for the next year |
| `ssl_early_data on` | Enables TLS 1.3 0-RTT — allows data to be sent on the first TLS flight |
| `Alt-Svc / x-quic` | Advertises HTTP/3 availability to browsers via response headers |
| `quic_retry on` | Enables the QUIC retry mechanism for additional resilience against spoofing |

> **OCSP stapling note:** Let's Encrypt removed OCSP stapling support in May 2025. If you are using Let's Encrypt certificates, ensure both `ssl_stapling` directives remain commented out in this file.

---

## 39.8 Step 7 — Update the Nginx Server Block for HTTPS

The site's Nginx configuration needs two changes: a redirect server block that sends all HTTP traffic to HTTPS, and updates to the main server block to listen on 443 with SSL/HTTP2/HTTP3 and include the two SSL files.

```bash
sudo nano /etc/nginx/sites-available/expertwp.help.conf
```

### HTTP → HTTPS Redirect Block

Add or replace the port 80 server block with a permanent redirect:

```nginx
server {
    listen 80;
    server_name expertwp.help www.expertwp.help;

    # 301 = permanent redirect
    return 301 https://expertwp.help$request_uri;
}
```

### Updated HTTPS Server Block — Listen Directives

Replace the single `listen 80` in the main server block with the following four listen lines to enable SSL, HTTP/2, and HTTP/3 (QUIC):

```nginx
# First site on the server — uses reuseport for QUIC
listen 443 ssl;
http2  on;
listen 443 quic reuseport;
http3  on;
```

> **Note:** `reuseport` is specified only on the first site's QUIC listen directive. Any additional sites on the same server use `listen 443 quic;` without `reuseport`.

### Include the SSL Files

Inside the HTTPS server block, include both SSL configuration files:

```nginx
include /etc/nginx/ssl/ssl_expertwp.help.conf;
include /etc/nginx/ssl/ssl_all_sites.conf;
```

### Test and Reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 39.9 Summary

After this chapter the server:

- Serves the site exclusively over HTTPS with a valid Let's Encrypt certificate
- Redirects all port 80 HTTP traffic permanently to HTTPS
- Uses a curated cipher suite and DH parameters to achieve an A+ rating on ssllabs.com
- Advertises and serves HTTP/3 over QUIC alongside HTTP/2

The SSL configuration is split into two reusable include files — a site-specific cert file and a shared protocol/cipher file — so adding a second site only requires issuing a new certificate and creating one new cert include file; the shared `ssl_all_sites.conf` is simply included again.

---

| | | |
|:---|:---:|---:|
| [← Chapter 38 — open_basedir](./38-open-basedir.md) | [↑ Top](#chapter-39--ssltls-certificates-certbot-lets-encrypt--nginx-https-configuration) | [Chapter 40 — HTTPS Verification, Mixed Content & SSL Renewal →](./40-https-verification-certbot-renewal.md) |
