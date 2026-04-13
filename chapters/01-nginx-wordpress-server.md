[← Back to Index](../index.md)

---

# Chapter 1 — Project Overview

*Production WordPress Server on Ubuntu 24.04 LTS*

---

A hardened, performance-optimised WordPress hosting environment built on Ubuntu 24.04 LTS using Nginx, MariaDB, and PHP. Designed to run multiple isolated WordPress sites on a single VPS with free SSL certificates, server-side caching, and Cloudflare DNS.

---

## Contents

- [1.1 Stack](#11-stack)
- [1.2 Goals](#12-goals)
- [1.3 Infrastructure](#13-infrastructure)
  - [Server](#server)
  - [Domain & DNS](#domain--dns)
- [1.4 Prerequisites](#14-prerequisites)
- [1.5 Configuration Principles](#15-configuration-principles)

---

## 1.1 Stack

| Component | Role |
|-----------|------|
| **Ubuntu 24.04 LTS** (Noble Numbat) | Operating system |
| **Nginx** | Web server and reverse proxy |
| **MariaDB** | Database server |
| **PHP** (with PHP-FPM pools) | Server-side scripting |
| **WordPress** | CMS |
| **Let's Encrypt / Certbot** | Free SSL/TLS certificates with auto-renewal |
| **Cloudflare** | DNS management and CDN |

---

## 1.2 Goals

- Google PageSpeed score of **97–100%**
- **A+** SSL rating
- **A(+)** HTTP response header security rating
- Multiple WordPress sites, each **isolated via PHP-FPM pools**
- **Wildcard SSL certificates** via Let's Encrypt
- Server-side caching at the Nginx level
- Hardened HTTP headers
- Tuned Nginx, MariaDB, and PHP configuration

---

## 1.3 Infrastructure

### Server

A commercially hosted VPS running **Ubuntu 24.04 LTS**. A real internet-facing server is required — local VMs cannot be used because SSL certificate issuance and DNS resolution depend on the server being publicly reachable.

### Domain & DNS

The domain is registered with any registrar, but DNS is managed through **Cloudflare** rather than the registrar's default nameservers. After registration, the nameservers are pointed to Cloudflare's. All DNS records — including the `A` record pointing to the server IP — are managed from the Cloudflare dashboard.

Cloudflare is used over registrar DNS for its faster propagation, DNSSEC support, DDoS mitigation, and free CDN proxying.

---

## 1.4 Prerequisites

The following Linux fundamentals are used throughout this setup:

**System**
- Linux file system hierarchy — `/etc`, `/var/www`, `/usr/local`, etc.
- Users, groups, file ownership, and permissions (`chmod`, `chown`)

**Package & Service Management**
- APT package manager and Ubuntu repositories
- `systemd` and `systemctl` for managing services

**File Editing**
- `nano` for editing configuration files

**Remote Access**
- SSH key-based authentication to connect to the server

**Scripting & Scheduling**
- Bash scripting basics
- Cron jobs — used for automated certificate renewal and maintenance tasks

---

## 1.5 Configuration Principles

- Each configuration file is written once, completely, from scratch — no incremental patching.
- Configuration directives are not mixed from different sources. Each component (Nginx, PHP, MariaDB) is configured as a cohesive unit.
- All configuration changes are followed by a service reload or restart and a status check to confirm the change took effect.

---

*Documentation continues section by section as each component is configured.*

---

| | | |
|:---|:---:|---:|
| [← Index](../index.md) | [↑ Top](#chapter-1--project-overview) | [Chapter 2 — Linux Essentials →](./02-linux-essentials.md) |
