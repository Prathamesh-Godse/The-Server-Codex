[← Back to Index](../index.md)

---

# Chapter 13 — Fail2Ban: Installation, Configuration & Management

*Installing Fail2Ban from the official release, configuring aggressive SSH brute-force protection with a 7-day ban policy, and managing bans in production.*

---

## Contents

- [13.1 Overview](#131-overview)
- [13.2 Installation](#132-installation)
  - [Step 1 — Download the Package](#step-1--download-the-package)
  - [Step 2 — Install the Package](#step-2--install-the-package)
  - [Step 3 — Verify the Service is Running](#step-3--verify-the-service-is-running)
- [13.3 Configuration](#133-configuration)
  - [Step 1 — Create jail.local](#step-1--create-jaillocal)
  - [Step 2 — Edit jail.local](#step-2--edit-jaillocal)
  - [Step 3 — Restart Fail2Ban](#step-3--restart-fail2ban)
- [13.4 Monitoring](#134-monitoring)
  - [Viewing Logs](#viewing-logs)
  - [Checking the sshd Jail Status](#checking-the-sshd-jail-status)
- [13.5 Unbanning Your Own IP](#135-unbanning-your-own-ip)
- [13.6 Quick Reference](#136-quick-reference)

---

## 13.1 Overview

Fail2ban is an intrusion prevention software framework that protects servers against brute-force attacks. It works by monitoring log files for repeated authentication failures and banning the offending IP addresses by dynamically inserting firewall rules to block them. The rules Fail2ban enforces are called **jails** — each jail monitors a specific service (e.g. SSH) and applies a ban when the configured failure threshold is crossed.

---

## 13.2 Installation

Fail2ban is installed from a `.deb` package downloaded directly from the official GitHub releases page rather than from the Ubuntu package repositories, ensuring the latest stable version is used.

### Step 1 — Download the Package

Visit the Fail2ban releases page to find the latest version:

```
https://github.com/fail2ban/fail2ban/releases/
```

From the Assets section of the latest release, copy the URL of the `_all.deb` file, then download it to the server using `wget` with the `-O` flag to name the output file:

```bash
wget -O fail2ban.deb https://github.com/fail2ban/fail2ban/releases/download/1.1.0/fail2ban_1.1.0-1.upstream1_all.deb
```

- `-O fail2ban.deb` — saves the downloaded file under the name `fail2ban.deb` regardless of the original filename
- The version in the URL should match the latest release available at the time of installation

wget will follow redirects through GitHub's CDN and confirm the save with `'fail2ban.deb' saved`. Verify the file is present:

```bash
ls -l
# -rw-rw-r-- 1 andrew andrew 320112 Apr 26 00:35 fail2ban.deb
```

### Step 2 — Install the Package

Install the downloaded `.deb` file using `dpkg`:

```bash
sudo dpkg -i fail2ban.deb
```

Then run `apt` to resolve and install any missing dependencies:

```bash
sudo apt -f install
```

The `-f` flag stands for "fix broken" — it scans for unmet dependencies from the `dpkg` install and resolves them automatically.

### Step 3 — Verify the Service is Running

```bash
sudo systemctl status fail2ban
```

The output should show `Active: active (running)`, confirming the service started immediately after installation:

```
● fail2ban.service - LSB: Start/stop fail2ban
     Loaded: loaded (/etc/init.d/fail2ban; generated)
     Active: active (running) since Wed 2024-07-03 16:59:33 UTC; 1min 4s ago
```

---

## 13.3 Configuration

Fail2ban ships with a default configuration file at `/etc/fail2ban/jail.conf`. This file should **never be edited directly** — package updates would overwrite any changes made to it. Instead, a local override file called `jail.local` is created, which Fail2ban merges on top of the defaults at runtime.

### Step 1 — Create `jail.local`

```bash
cd /etc/fail2ban
ls
sudo cp jail.conf jail.local
```

Running `ls` again confirms `jail.local` now exists alongside `jail.conf`.

### Step 2 — Edit `jail.local`

```bash
sudo nano jail.local
```

#### Tune the global `[DEFAULT]` ban parameters

Locate the three key parameters in the `[DEFAULT]` section and update them as follows:

```ini
# How long an IP is banned (d = days, h = hours, m = minutes)
bantime  = 7d

# Time window in which failures are counted
findtime = 3h

# Number of failures before a ban is triggered
maxretry = 3
```

**What these mean:**

- `bantime = 7d` — a banned IP stays blocked for 7 days (default is 10 minutes)
- `findtime = 3h` — the `maxretry` count is measured over a 3-hour window (default is 10 minutes)
- `maxretry = 3` — an IP is banned after just 3 failed attempts within the `findtime` window (default is 5)

This combination means: if any IP fails SSH authentication 3 times within a 3-hour window, it gets blocked for a full week.

#### Enable and configure the `[sshd]` jail

Scroll down to the `[sshd]` section. By default it looks like:

```ini
[sshd]

#mode   = normal
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

Modify it to enable the jail and set it to aggressive mode:

```ini
[sshd]

mode    = aggressive
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
enabled = true
```

- `mode = aggressive` — enables the most thorough detection, combining normal, ddos, and extra filter patterns to catch a wider range of brute-force attempts
- `enabled = true` — explicitly activates this jail

Save and exit nano (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Step 3 — Restart Fail2Ban to Apply the Configuration

```bash
sudo systemctl restart fail2ban
```

---

## 13.4 Monitoring

### Viewing Logs

Fail2ban writes its activity to `/var/log/fail2ban.log`:

```bash
sudo cat /var/log/fail2ban.log
```

Key log entries to look for:

```
INFO    Starting Fail2ban v1.1.0.1
INFO    Daemon started
INFO    Creating new jail 'sshd'
INFO    Jail 'sshd' uses systemd {}
INFO    maxRetry: 3
INFO    findtime: 10800
INFO    banTime: 604800
INFO    Jail 'sshd' started
```

- `banTime: 604800` — 604800 seconds = 7 days, confirming `bantime = 7d`
- `findtime: 10800` — 10800 seconds = 3 hours, confirming `findtime = 3h`
- `maxRetry: 3` — confirms the retry threshold

The `auth.log` file records all PAM/sudo activity and is also useful for tracing what Fail2ban is acting on:

```bash
sudo cat /var/log/auth.log
# or page through it with:
sudo less /var/log/auth.log
```

### Checking the sshd Jail Status

To see live statistics on how many IPs Fail2ban has banned:

```bash
sudo fail2ban-client status sshd
```

Example output:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 10
|  |- Total failed:     79
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 90
   |- Total banned:     90
   `- Banned IP list:   106.245.142.182 106.70.252.79 ...
```

This output confirms Fail2ban is actively working — 90 IPs banned after 79 total failed attempts detected on the sshd service.

---

## 13.5 Unbanning Your Own IP

If your own IP gets banned (e.g. from too many failed passphrase attempts during testing), you have three options:

**Option 1 — Get a new IP:** Reboot or power-cycle your router. If your ISP uses dynamic IPs, you'll likely get a new public IP on reconnect.

**Option 2 — Manual unban via console:** Log in through the hosting provider's web console and unban yourself from there.

First, find your current public IP address (search "what is my ip" in a browser). Then check recent logins to confirm the banned IP:

```bash
last -n3
```

Then use `fail2ban-client` to remove the ban:

```bash
sudo fail2ban-client set sshd unbanip <your_ip_address>
```

Replace `<your_ip_address>` with your actual public IP (e.g. `41.4.18.80`).

---

## 13.6 Quick Reference

```bash
# --- Download and install Fail2ban ---
# Check https://github.com/fail2ban/fail2ban/releases/ for latest version URL
wget -O fail2ban.deb <latest_.deb_url>
sudo dpkg -i fail2ban.deb
sudo apt -f install
sudo systemctl status fail2ban

# --- Create and edit the local config ---
cd /etc/fail2ban
sudo cp jail.conf jail.local
sudo nano jail.local

# In [DEFAULT] section, set:
# bantime  = 7d
# findtime = 3h
# maxretry = 3

# In [sshd] section, set:
# mode    = aggressive
# enabled = true

# --- Restart to apply changes ---
sudo systemctl restart fail2ban

# --- Monitor ---
sudo cat /var/log/fail2ban.log
sudo fail2ban-client status sshd

# --- Unban your own IP ---
last -n3
sudo fail2ban-client set sshd unbanip <your_ip_address>
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 12 — Firewall: UFW & Cloud](./12-firewall-ufw-and-cloud.md) | [↑ Top](#chapter-13--fail2ban-installation-configuration--management) | [Chapter 14 — Harden & Optimise: Timezone, Swap →](./14-harden-and-optimize-server-distribution.md) |
