[← Back to Index](../index.md)

---

# Chapter 7 — Server Provisioning

*Specifications, OS decisions, hosting provider selection, and creating a server instance — the decisions made before running a single command.*

---

## Contents

- [7.1 Server Specifications](#71-server-specifications)
  - [Development / Practice Server](#development--practice-server)
  - [Production Server](#production-server)
- [7.2 Server Distribution — Ubuntu 24.04 LTS](#72-server-distribution--ubuntu-2404-lts)
  - [Why Ubuntu 24.04 LTS](#why-ubuntu-2404-lts)
  - [Ubuntu Pro — When to Enable It](#ubuntu-pro--when-to-enable-it)
- [7.3 The Server Upgrade Rule — Never Upgrade in Place](#73-the-server-upgrade-rule--never-upgrade-in-place)
  - [The Correct Approach: Migrate, Don't Upgrade](#the-correct-approach-migrate-dont-upgrade)
- [7.4 Choosing a Hosting Provider](#74-choosing-a-hosting-provider)
- [7.5 Creating a Server Instance on Vultr](#75-creating-a-server-instance-on-vultr)
  - [Location Selection](#location-selection)
  - [Hostname Convention](#hostname-convention)
- [7.6 Summary](#76-summary)

---

## 7.1 Server Specifications

The appropriate server spec depends entirely on what the server is being used for. There are two distinct contexts.

### Development / Practice Server

Used for building and testing the setup before going live. Resource requirements are minimal since there is no real traffic.

| Resource | Specification |
|----------|--------------|
| CPU | Single core |
| RAM | 1–2 GB |
| Storage | SSD |
| Network | Multi Gb/s |

### Production Server

Used to serve live WordPress sites with real visitors. Requirements scale with the number and type of sites hosted.

| Resource | Specification |
|----------|--------------|
| CPU | Dual core (quad core preferred) |
| RAM | 2 GB minimum (4 GB+ recommended) |
| Storage | SSD or NVMe |
| Network | Multi Gb/s |

> **On RAM:** When in doubt, provision more RAM than you think you need. Running out of memory causes PHP-FPM workers to be killed, MariaDB to crash, and the site to go down. Excess RAM is cheap; recovery from an OOM event is not.

> **On bandwidth:** Be aware of bandwidth overage charges. Some providers bill aggressively beyond the included monthly allowance. Check the pricing page carefully before choosing a plan.

---

## 7.2 Server Distribution — Ubuntu 24.04 LTS

### Why Ubuntu 24.04 LTS

Ubuntu 24.04 LTS (Noble Numbat) is the distribution used for this server build. The key reasons:

- **Kernel + distro distinction:** Linux is the kernel; Ubuntu is the distribution built on top of it. Choosing a distribution means choosing a packaging system, release cadence, default tooling, and support lifecycle — Ubuntu's are well-suited to server use.
- **Standard LTS support:** Supported until **April 2029** — 5 years of security updates and bug fixes from Canonical.
- **Extended support via Ubuntu Pro:** The free Ubuntu Pro plan (available for up to 5 machines personally) extends support to **2036**, adding 7 years on top of the standard LTS window via the ESM (Expanded Security Maintenance) programme.
- **Livepatch:** Ubuntu Pro includes Livepatch, which applies kernel security patches without requiring a reboot — important for keeping uptime high on production servers.
- **Stability, security, and performance:** LTS releases are conservative by design. Packages are well-tested, defaults are hardened, and regressions are rare.
- **Large community and documentation:** The single largest body of Linux server tutorials, Stack Overflow answers, and vendor documentation targets Ubuntu.

### Ubuntu Pro — When to Enable It

Ubuntu Pro is free for personal use on up to 5 machines. However, enabling it on a fresh server before you are comfortable with the command line introduces additional package sources and background services that can make debugging harder.

> **Recommendation:** Enable Ubuntu Pro after the full server setup is complete and confirmed working — not during the initial build.

```bash
# Enable Ubuntu Pro after completing the server setup:
sudo pro attach <your-token>
```

Your token is available at https://ubuntu.com/pro after registering a free account.

---

## 7.3 The Server Upgrade Rule — Never Upgrade in Place

Ubuntu LTS releases a new version every 2 years. When a new LTS drops, the correct approach for production servers is **not** to run `do-release-upgrade`. Doing so on a server that has been in use for months carries serious risk:

- Hundreds of new packages will be installed or replaced
- Many configuration files that were manually modified will be overwritten or conflict
- The upgrade will likely cause extended downtime and will often end in a broken system requiring a full reinstallation anyway

### The Correct Approach: Migrate, Don't Upgrade

1. Provision a **new server instance** running the new LTS version
2. Build and configure the new server from scratch
3. Copy sites and databases to the new server
4. Test everything on the new server before changing DNS
5. Switch DNS to point to the new server
6. If issues arise, revert DNS back to the old server and fix them
7. Destroy the old server only once the new one is fully confirmed working

This approach gives you a clean new environment, zero-downtime migration, and a safe rollback path — none of which an in-place upgrade can guarantee.

---

## 7.4 Choosing a Hosting Provider

Any cloud or VPS provider can be used as long as it allows deploying Ubuntu 24.04 LTS. Key considerations when choosing:

- **Distribution image quality:** Some providers heavily customise their Ubuntu images, adding proprietary agents, modifying default configs, or altering package sources. This can introduce unexpected behaviour and make troubleshooting harder. Prefer providers that offer a near-stock Ubuntu image.
- **Hardware quality:** Cheap providers often use oversold hardware with slow disk I/O and inconsistent network performance. For a WordPress server, disk speed matters — MariaDB is I/O-bound on reads and writes.
- **Support:** Budget hosts typically provide no meaningful technical support.
- **Network:** Multi-gigabit uplinks and low-latency peering significantly affect page load times for real visitors.

**Vultr** (`vultr.com`) was chosen for this project for the following reasons:

- Stable, reliable infrastructure with consistently good uptime
- Competitive pricing with predictable billing
- Data centres across all major global regions — choose the one closest to your target audience
- Quick server provisioning — a new instance is ready in under a minute
- Ubuntu images are close to stock, minimising distribution-specific surprises
- Good support documentation and responsive ticket support

> Vultr periodically offers free credit promotions for new signups. Check the current offer on their site before registering.

---

## 7.5 Creating a Server Instance on Vultr

When creating a new server instance, the following choices are made through the Vultr control panel:

| Setting | What to Select |
|---------|---------------|
| **Server type** | Cloud Compute (shared vCPU) for dev; High Frequency or Dedicated CPU for production |
| **Server location** | Data centre closest to your primary audience |
| **Server image** | Ubuntu 24.04 LTS x64 |
| **Server size** | Match the dev or production spec tables above |
| **Hostname** | A meaningful name, e.g. `nginx-wp-prod-01` |

### Location Selection

Choose the data centre region that minimises latency for your visitors. Server location directly affects Time to First Byte (TTFB). If your primary audience is in India, choose a Mumbai or Singapore region — not a US or European one.

### Hostname Convention

Use a hostname that describes the server's purpose and makes it identifiable in a fleet. A consistent naming scheme matters when managing multiple servers:

```
nginx-wp-dev-01      # development server
nginx-wp-prod-01     # production server
```

The hostname is set during instance creation and can be referenced in scripts, SSH configs, and monitoring tools.

---

## 7.6 Summary

| Decision | Choice |
|----------|--------|
| Dev server spec | 1 vCPU, 1–2 GB RAM, SSD |
| Production server spec | 2–4 vCPU, 4+ GB RAM, NVMe SSD |
| Server OS | Ubuntu 24.04 LTS |
| LTS upgrade strategy | Migrate to new instance — never upgrade in place |
| Hosting provider | Vultr (or any provider with near-stock Ubuntu images) |
| Instance settings | Type, location, Ubuntu 24.04 image, size, hostname |

---

| | | |
|:---|:---:|---:|
| [← Chapter 6 — Local Machine Setup](./06-local-setup-and-server-os.md) | [↑ Top](#chapter-7--server-provisioning) | [Chapter 8 — First Login & Hardening Intro →](./08-first-login-and-hardening-intro.md) |
