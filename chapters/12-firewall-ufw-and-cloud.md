[← Back to Index](../index.md)

---

# Chapter 12 — Firewall: UFW & Cloud Firewall

*Configuring two layers of network defence — UFW at the OS level and the Vultr cloud firewall at the network perimeter — with the ports required to serve a WordPress site over HTTPS.*

---

## Contents

- [12.1 Overview](#121-overview)
- [12.2 Ports Used in This Setup](#122-ports-used-in-this-setup)
- [12.3 UFW Configuration](#123-ufw-configuration)
  - [Step 1 — Check Current UFW Status](#step-1--check-current-ufw-status)
  - [Path A — UFW Pre-Enabled by Host](#path-a--ufw-pre-enabled-by-host)
  - [Path B — UFW Not Pre-Enabled by Host](#path-b--ufw-not-pre-enabled-by-host)
- [12.4 Cloud Firewall (Vultr)](#124-cloud-firewall-vultr)
  - [Setting Up the Vultr Cloud Firewall](#setting-up-the-vultr-cloud-firewall)
  - [Linking the Firewall Group to the Server](#linking-the-firewall-group-to-the-server)
  - [Verifying the Cloud Firewall is Active](#verifying-the-cloud-firewall-is-active)
- [12.5 Quick Reference](#125-quick-reference)

---

## 12.1 Overview

A firewall is a network security system — software or hardware-based — that controls incoming and outgoing traffic by inspecting data packets and deciding whether to allow or block them based on a defined rule set. On a Linux server, the kernel-level firewall is managed by `iptables` or `nftables`. Writing rules for these directly is powerful but error-prone, so **UFW (Uncomplicated Firewall)** is used as a front-end that makes rule management straightforward.

Beyond the OS-level firewall, Vultr offers a **cloud firewall** that operates at the network perimeter — before traffic even reaches the server. Running both layers together gives defence in depth: the cloud firewall acts as the outer gate, and UFW handles finer-grained OS-level control.

---

## 12.2 Ports Used in This Setup

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 443 | UDP | HTTPS (HTTP/3 / QUIC) |
| — | ICMP | Ping (enabled by default) |

---

## 12.3 UFW Configuration

The UFW setup process differs slightly depending on whether the hosting provider pre-enables UFW or not. Both paths end with the same active rule set.

### Step 1 — Check Current UFW Status

Always check the current state of UFW before making changes:

```bash
sudo ufw status verbose
```

If the host pre-enabled UFW, the output will show `Status: active` with SSH (port 22) already allowed. If UFW is not enabled by the host, the output will show `Status: inactive`.

---

### Path A — UFW Pre-Enabled by Host

When the hosting provider has already enabled UFW with SSH allowed, the default policy will already be `deny incoming / allow outgoing`. Add the HTTP and HTTPS rules on top of the existing SSH rule.

**Add HTTP and HTTPS rules:**

```bash
sudo ufw allow http
sudo ufw allow https/tcp
sudo ufw allow https/udp
```

Each command responds with `Rule added` and `Rule added (v6)` — UFW automatically creates both IPv4 and IPv6 rules.

**Reload UFW to apply the new rules without a full restart:**

```bash
sudo ufw reload
```

**Verify the full rule set:**

```bash
sudo ufw status verbose
```

Expected output:

```
Status: active
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
443/udp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
443/udp (v6)               ALLOW IN    Anywhere (v6)
```

**Reboot and verify the firewall persists:**

```bash
sudo reboot
```

After reconnecting with `ssh 2404`, run `sudo ufw status verbose` again to confirm UFW is still active and all rules survived the reboot.

---

### Path B — UFW Not Pre-Enabled by Host

If the host has not enabled UFW (`Status: inactive`), the full setup is done manually. The critical rule to add first is SSH — failing to do so before enabling the firewall would lock you out.

**Set default policies — deny all incoming, allow all outgoing:**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

**Allow all required services:**

```bash
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https/tcp
sudo ufw allow https/udp
```

> **Important:** `sudo ufw allow ssh` must be run **before** enabling the firewall. Skipping this step will block your own SSH connection when UFW is activated.

**Enable UFW:**

```bash
sudo ufw enable
```

UFW will warn that the command may disrupt existing SSH connections. Confirm with `y`:

```
Firewall is active and enabled on system startup
```

This also configures UFW to start automatically on every boot.

**Verify the rule set:**

```bash
sudo ufw status verbose
```

The output should show the same active rule set as in Path A.

**Reboot and verify persistence:**

```bash
sudo reboot
```

Reconnect with `ssh 2404` and run `sudo ufw status verbose` to confirm the firewall is still active after the reboot.

---

## 12.4 Cloud Firewall (Vultr)

A cloud firewall operates at the provider network level — it filters traffic before it reaches the server's network interface. This is an additional layer on top of UFW, not a replacement.

When both are active, a port must be allowed by **both** firewalls for traffic to reach the application. This is intentional — it adds redundancy.

### Setting Up the Vultr Cloud Firewall

In the Vultr dashboard, navigate to **Products → Network → Firewall**, then click **+ Add Firewall Group**.

Configure the following inbound IPv4 rules:

| Action | Protocol | Port | Source |
|--------|----------|------|--------|
| accept | ICMP | — | 0.0.0.0/0 |
| accept | SSH | 22 | your-ip/32 (restricted) |
| accept | TCP (HTTP) | 80 | 0.0.0.0/0 |
| accept | TCP (HTTPS) | 443 | 0.0.0.0/0 |
| accept | UDP | 443 | 0.0.0.0/0 |
| drop | any | 0–65535 | 0.0.0.0/0 (default) |

> SSH is deliberately restricted to a specific IP (e.g. `41.4.18.80/32`) rather than `Anywhere`. This limits SSH access to your own machine, reducing brute-force exposure at the network perimeter.

### Linking the Firewall Group to the Server

After configuring the rules, link the firewall group to the server under **Linked Instances**. Once linked, the rule propagation notice confirms it may take up to 120 seconds for the rules to apply across all linked servers.

### Verifying the Cloud Firewall is Active

Attempt an SSH connection from a different IP after linking the instance to the firewall group (with SSH restricted to a specific IP) — it should be refused:

```
ssh: connect to host 95.179.129.251 port 22: Connection refused
```

After adding your own IP to the SSH allow rule, the connection works again:

```bash
ssh 2404
# Enter passphrase for key '/home/andrew/.ssh/my2404server_keys':
# Welcome to Ubuntu 24.04 LTS...
```

This confirms both UFW and the Vultr cloud firewall are active and correctly configured.

---

## 12.5 Quick Reference

```bash
# --- Check UFW status ---
sudo ufw status verbose

# --- If UFW is pre-enabled by host: add HTTP/HTTPS rules ---
sudo ufw allow http
sudo ufw allow https/tcp
sudo ufw allow https/udp
sudo ufw reload

# --- If UFW is NOT pre-enabled: full setup from scratch ---
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh          # critical: do this BEFORE enabling
sudo ufw allow http
sudo ufw allow https/tcp
sudo ufw allow https/udp
sudo ufw enable

# --- Verify rules ---
sudo ufw status verbose

# --- Reboot and re-verify persistence ---
sudo reboot
# after reconnecting:
sudo ufw status verbose
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 11 — SSH Config & Server Updates](./11-ssh-config-and-server-updates.md) | [↑ Top](#chapter-12--firewall-ufw--cloud-firewall) | [Chapter 13 — Fail2Ban →](./13-fail2ban.md) |
