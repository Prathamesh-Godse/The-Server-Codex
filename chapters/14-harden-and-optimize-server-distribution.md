[← Back to Index](../index.md)

---

# Chapter 14 — Hardening & Optimising the Server Distribution

*Tightening the base OS before deploying any web services — setting the correct timezone, configuring a swap file, and laying the stable foundation that Nginx, MariaDB, PHP, and WordPress will run on top of.*

---

## Contents

- [14.1 Timezone Configuration](#141-timezone-configuration)
  - [Check the Current Timezone](#check-the-current-timezone)
  - [List Available Timezones](#list-available-timezones)
  - [Set the Timezone](#set-the-timezone)
- [14.2 Swap File](#142-swap-file)
  - [What Swap Is and Why a VPS Needs It](#what-swap-is-and-why-a-vps-needs-it)
  - [Check Existing Swap Status](#check-existing-swap-status)
  - [Remove an Existing Swap File](#remove-an-existing-swap-file)
  - [Create a New Swap File](#create-a-new-swap-file)
  - [Set Correct Permissions](#set-correct-permissions)
  - [Format and Activate the Swap File](#format-and-activate-the-swap-file)
  - [Make the Swap File Persist Across Reboots](#make-the-swap-file-persist-across-reboots)

---

## 14.1 Timezone Configuration

### Why It Matters

The server timezone controls timestamps across the entire system — cron jobs, log files, application events, SSL certificate renewals. Leaving it at UTC is fine in many cases, but if scheduled tasks need to fire at times that match a specific region, the timezone must be set correctly from the start.

### Check the Current Timezone

```bash
sudo timedatectl
```

This outputs the current local time, universal time, RTC time, timezone, and NTP sync status. Out of the box, a fresh Ubuntu 24.04 VPS will typically show `Etc/UTC`.

### List Available Timezones

```bash
sudo timedatectl list-timezones
```

This produces a long list. To narrow it down, pipe through `grep` with a city or region name:

```bash
sudo timedatectl list-timezones | grep Paris
sudo timedatectl list-timezones | grep Europe
sudo timedatectl list-timezones | grep Johan
```

Each search returns only matching entries, making it easy to confirm the exact string needed (e.g. `Europe/Paris`, `Africa/Johannesburg`).

### Set the Timezone

```bash
sudo timedatectl set-timezone Europe/Paris
```

Replace `Europe/Paris` with whichever timezone applies to the server's location or the region it serves. After setting it, verify:

```bash
sudo timedatectl
date
```

`timedatectl` shows the updated `Time zone` field; `date` confirms the local clock is correct.

> **Automation note:** In a bash script, this can be driven by a variable: `TIMEZONE="Europe/Paris"` → `sudo timedatectl set-timezone "$TIMEZONE"`. No reboot required — the change is immediate.

---

## 14.2 Swap File

### What Swap Is and Why a VPS Needs It

Swap is disk space the kernel uses as overflow when physical RAM is exhausted. On a VPS, there is no swap partition by default — the disk is pre-partitioned by the provider and there is no dedicated swap slice. Instead, a swap *file* is created directly on the root filesystem and registered with the kernel.

A swap file is not a replacement for RAM. It is a safety net: it prevents the OOM killer from terminating processes during brief memory spikes. The recommended size is at least equal to the amount of RAM, up to double — so a 1 GB server gets a 1–2 GB swap file, a 4 GB server gets a 4–8 GB swap file.

### Check Existing Swap Status

Before doing anything, confirm whether swap is already present:

```bash
sudo swapon -s
```

If the server already has a swap file, this will list it. Use `htop` to cross-check:

```bash
htop
```

The `Swp` bar at the top of `htop` shows current swap usage. If size is non-zero, swap is active.

---

### Remove an Existing Swap File

If the existing swap file needs to be removed — for example to recreate it at a different size — deactivate it first, then delete it and clean up `/etc/fstab`.

**Deactivate the swap file:**

```bash
sudo swapoff /swapfile
```

**Back up `/etc/fstab` before modifying it:**

```bash
cd /etc
sudo cp fstab fstab.bak
```

**Remove the swap line from `/etc/fstab`:**

```bash
sudo nano /etc/fstab
```

Delete the line that references `/swapfile`. Save and exit.

**Delete the swap file from disk:**

```bash
sudo rm /swapfile
```

Verify it is gone:

```bash
ls -l /
sudo swapon -s
```

An empty result from `swapon -s` means no swap is active. Reboot the server to ensure the fstab change is reflected cleanly:

```bash
sudo reboot
```

After reconnecting, run `sudo swapon -s` again — it should return nothing.

---

### Create a New Swap File

With the old swap file removed (or if starting from scratch), create a fresh one. The following command creates a **2 GB** swap file. Adjust `count` proportionally for a different size — the formula is `size_in_MB × 1024`:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1024 count=2097152
```

- `if=/dev/zero` — reads an infinite stream of zero bytes as input
- `of=/swapfile` — writes the output to `/swapfile` at the root of the filesystem
- `bs=1024` — block size of 1024 bytes (1 KB)
- `count=2097152` — number of blocks; 2097152 × 1024 bytes = 2 GB

The command reports how many bytes were written and the transfer speed when it completes. Verify the file is present:

```bash
ls -l /
```

The `swapfile` entry should show a size of `2147483648` bytes (2 GB).

### Set Correct Permissions

The swap file must be readable and writable only by root. If other users can read it, they can potentially read sensitive kernel memory:

```bash
sudo chmod 600 /swapfile
```

Running `ls -l /` will now show the permissions as `-rw-------`, confirming only root has access.

### Format and Activate the Swap File

Mark the file as a Linux swap area:

```bash
sudo mkswap /swapfile
```

The output confirms the swap space version, size, and assigns a UUID.

Activate the swap file immediately (without a reboot):

```bash
sudo swapon /swapfile
```

Verify it is active:

```bash
sudo swapon -s
```

The output should list `/swapfile` with type `file`, size approximately `2097148` KB, used `0`, and priority `-2`.

### Make the Swap File Persist Across Reboots

Activating swap with `swapon` is session-only — it disappears after a reboot. To make it permanent, add it to `/etc/fstab`.

Navigate to `/etc` and back up `fstab` before editing:

```bash
cd /etc
sudo cp fstab fstab.bak
sudo nano fstab
```

Add the following line at the end of the file:

```
/swapfile swap swap defaults 0 0
```

The complete `fstab` will look similar to this:

```
# /etc/fstab: static file system information.
#
# <file system>                             <mount point>  <type>  <options>  <dump>  <pass>
/dev/disk/by-uuid/70afd07d-9b8c-402d...    /              ext4    defaults    0       1
/dev/disk/by-uuid/7C3C-1CDF                /boot/efi      vfat    defaults    0       1
/swapfile                                  swap           swap    defaults    0       0
```

Save and exit. Reboot to confirm the swap file activates automatically:

```bash
sudo reboot
```

After reconnecting, verify:

```bash
sudo swapon -s
```

The `/swapfile` entry should be listed exactly as before, confirming it survives reboots. The MOTD login banner will also show `Swap usage: 0%` alongside system information.

---

> **What comes next:** Chapter 15 continues the OS hardening process — kernel memory tuning via `sysctl`, hardening shared memory, disabling IPv6, and applying a comprehensive set of network layer security and performance directives.

---

| | | |
|:---|:---:|---:|
| [← Chapter 13 — Fail2Ban](./13-fail2ban.md) | [↑ Top](#chapter-14--hardening--optimising-the-server-distribution) | [Chapter 15 — Harden & Optimise: Kernel & Network →](./15-harden-and-optimize-server-distribution-2.md) |
