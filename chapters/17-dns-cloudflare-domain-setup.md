[← Back to Index](../index.md)

---

# Chapter 17 — DNS: Pointing a Domain to the Server

*Making the server reachable via a domain name — delegating DNS to Cloudflare and creating the A and CNAME records that point the domain at the server's IP.*

---

## Contents

- [17.1 Why Cloudflare for DNS?](#171-why-cloudflare-for-dns)
- [17.2 Prerequisites](#172-prerequisites)
- [17.3 Add the Domain to Cloudflare](#173-add-the-domain-to-cloudflare)
- [17.4 Update Nameservers at Your Registrar](#174-update-nameservers-at-your-registrar)
- [17.5 Create the Required DNS Records](#175-create-the-required-dns-records)
  - [A Record — Root Domain to Server IP](#a-record--root-domain-to-server-ip)
  - [CNAME Record — www Subdomain](#cname-record--www-subdomain)
- [17.6 Summary](#176-summary)

---

## 17.1 Why Cloudflare for DNS?

Rather than using the DNS service bundled with a domain registrar, Cloudflare's free DNS tier is used here for three key reasons:

- **Speed** — Cloudflare operates one of the fastest DNS networks globally, with extremely low query response times.
- **Security** — Built-in DDoS protection, DNSSEC support, and optional proxying through Cloudflare's edge network.
- **Propagation time** — DNS changes made in Cloudflare propagate significantly faster than changes made at most registrars.

---

## 17.2 Prerequisites

A registered domain name is required to proceed from this point in the project. It can be purchased from any registrar (Namecheap, GoDaddy, Porkbun, etc.) or an existing domain already in your possession can be used. The registrar itself does not matter — DNS management will be delegated to Cloudflare.

---

## 17.3 Add the Domain to Cloudflare

1. Sign up for a free account at [https://cloudflare.com](https://cloudflare.com).
2. From the dashboard, click **Add a site** and enter your domain name.
3. Select the **Free** plan.
4. Cloudflare will scan your domain's existing DNS records and import them automatically. Review the list and confirm.
5. Cloudflare will present two nameserver addresses (e.g. `ns1.cloudflare.com`, `ns2.cloudflare.com`).

---

## 17.4 Update Nameservers at Your Registrar

Log in to your domain registrar and replace the existing nameservers with the two nameservers Cloudflare provides. The exact steps vary by registrar, but the setting is typically found under **Domain Management → Nameservers**.

Once saved, Cloudflare will monitor the change and send an email notification when the domain has been successfully moved to their DNS service. Propagation typically completes within a few minutes to a few hours.

---

## 17.5 Create the Required DNS Records

After the domain is active on Cloudflare, two records need to be created (or updated if they already exist from the scan).

### A Record — Root Domain to Server IP

An **A record** maps a domain name directly to an IPv4 address. This is the primary record that points the root domain at the server.

| Field | Value |
|---|---|
| Type | `A` |
| Name | `yourdomain.com` (or `@` for the root) |
| Content | Your server's public IPv4 address |
| Proxy status | **DNS only** (grey cloud) |
| TTL | Auto |

> **Important:** Set the proxy status to **DNS only** at this stage. The server is not yet configured to correctly handle Cloudflare's proxied mode (orange cloud). This will be enabled later once the Nginx configuration is complete.

### CNAME Record — `www` Subdomain

A **CNAME record** creates an alias, pointing the `www` subdomain to the root domain. This ensures both `yourdomain.com` and `www.yourdomain.com` resolve to the same server.

| Field | Value |
|---|---|
| Type | `CNAME` |
| Name | `www` |
| Content | `yourdomain.com` |
| Proxy status | **DNS only** (grey cloud) |
| TTL | Auto |

---

## 17.6 Summary

At this point the domain is delegated to Cloudflare and both the root (`yourdomain.com`) and the `www` subdomain (`www.yourdomain.com`) resolve to the server's IP address. The Cloudflare proxy is intentionally left disabled for now — the server-side Nginx configuration must be in place before the orange-cloud proxied mode can be activated.

| Record | Name | Points to | Purpose |
|---|---|---|---|
| A | `yourdomain.com` | Server IP | Maps the root domain to the server |
| CNAME | `www` | `yourdomain.com` | Aliases the www subdomain to the root |

---

| | | |
|:---|:---:|---:|
| [← Chapter 16 — Congestion Control, File Access & Open File Limits](./16-congestion-control-file-access-open-file-limits.md) | [↑ Top](#chapter-17--dns-pointing-a-domain-to-the-server) | [Chapter 18 — Installing Nginx, MariaDB & PHP →](./18-installing-nginx-mariadb-php.md) |
