[← Back to Index](../index.md)

---

# Chapter 16 — Congestion Control, File Access Times & Open File Limits

*Three kernel and OS-level optimisations targeting specific bottlenecks: enabling BBR TCP congestion control, eliminating unnecessary filesystem access time writes, and raising the system-wide open file descriptor ceiling.*

---

## Contents

- [16.1 TCP Congestion Control — BBR](#161-tcp-congestion-control--bbr)
  - [What is BBR?](#what-is-bbr)
  - [Verify the Currently Active Algorithm](#verify-the-currently-active-algorithm)
- [16.2 File Access Times — noatime](#162-file-access-times--noatime)
  - [What is the Problem?](#what-is-the-problem)
  - [Inspect the Current Filesystem and Mounts](#inspect-the-current-filesystem-and-mounts)
  - [Back Up and Edit /etc/fstab](#back-up-and-edit-etcfstab)
  - [Reboot and Verify](#reboot-and-verify)
- [16.3 Open File Limits](#163-open-file-limits)
  - [Why This Matters](#why-this-matters)
  - [Check Current Limits](#check-current-limits)
  - [Set New Limits via /etc/security/limits.d/](#set-new-limits-via-etcsecuritylimitsd)
  - [Enable PAM Limits](#enable-pam-limits)
  - [Reboot and Confirm](#reboot-and-confirm)
  - [What Comes Next](#what-comes-next)
- [16.4 Summary of Changes](#164-summary-of-changes)

---

## 16.1 TCP Congestion Control — BBR

### What is BBR?

BBR (Bottleneck Bandwidth and Round-Trip Time) is a TCP congestion control algorithm developed by Google. Unlike the default `cubic` algorithm, which reacts to packet loss, BBR models the network path to maximise throughput and minimise queuing delay. For a web server, this translates to significantly faster data delivery and lower latency under load.

### Verify the Currently Active Algorithm

Check which congestion control algorithms are available on the kernel, and which one is currently active:

```bash
sudo sysctl net.ipv4.tcp_available_congestion_control
sudo sysctl net.ipv4.tcp_congestion_control
```

Expected output after the previous section's configuration:

```
net.ipv4.tcp_available_congestion_control = reno cubic bbr
net.ipv4.tcp_congestion_control = bbr
```

BBR is already loaded and active at this point, having been configured in the sysctl custom overrides in Chapter 15. This check confirms the setting persisted across the reboot.

---

## 16.2 File Access Times — `noatime`

### What is the Problem?

By default, Linux filesystems track the last time every file was **read** (not just written). Each read operation triggers a corresponding write to update the `atime` (access time) metadata. On a busy web server serving hundreds of files per second, this creates a constant stream of unnecessary I/O that hurts performance.

Disabling this behaviour with the `noatime` mount option eliminates those writes entirely, which can produce measurable performance improvements on high-traffic servers.

> **Warning:** The `/etc/fstab` layout varies between hosting providers and installation methods. Always inspect your own fstab before editing. An incorrect entry can prevent the server from booting. If you are unsure, consult your host's documentation before proceeding.

### Inspect the Current Filesystem and Mounts

First, confirm the root filesystem device and type:

```bash
df -h
```

Then view the full list of currently active mounts to identify the correct mount options already in use:

```bash
sudo cat /proc/mounts
```

The key line to look for is the root filesystem entry. Before the change it shows `relatime`:

```
/dev/vda2 / ext4 rw,relatime 0 0
```

### Back Up and Edit `/etc/fstab`

Navigate to `/etc` and open the fstab file:

```bash
cd /etc
sudo cp fstab fstab.bak
sudo nano fstab
```

Locate the root filesystem entry (identified by UUID or device path). Add `,noatime` to its options field.

**Before:**

```
/dev/disk/by-uuid/70afd07d-9b8c-402d-a5ee-5ec616852272 / ext4 defaults 0 1
```

**After:**

```
/dev/disk/by-uuid/70afd07d-9b8c-402d-a5ee-5ec616852272 / ext4 defaults,noatime 0 1
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Reboot and Verify

Apply the change by rebooting:

```bash
sudo reboot
```

After reconnecting, confirm the `noatime` option is active in the live mount table:

```bash
sudo cat /proc/mounts
```

The root filesystem entry should now show `noatime` instead of `relatime`:

```
/dev/vda2 / ext4 rw,noatime 0 0
```

---

## 16.3 Open File Limits

### Why This Matters

On Linux, every open network connection (socket) counts as an open file. The default per-process limit for open file descriptors is very low — typically 1024 for soft limits. For a web server handling hundreds or thousands of simultaneous connections, this ceiling causes the "too many open files" error, which kills performance and can crash services.

There are two types of limits:

- **Hard limit** — the absolute maximum, set by root. A process cannot raise its soft limit beyond this value.
- **Soft limit** — the effective limit applied to running processes. Can be raised up to the hard limit by the process owner.

### Check Current Limits

Before making any changes, check the existing hard and soft limits:

```bash
ulimit -Hn   # Hard limit
ulimit -Sn   # Soft limit
```

On a fresh Ubuntu 24.04 server, the default values are typically:

```
Hard: 1048576
Soft: 1024
```

The soft limit of 1024 is the bottleneck.

### Set New Limits via `/etc/security/limits.d/`

Navigate to the limits drop-in directory and create a custom override file:

```bash
cd /etc/security/limits.d/
sudo nano custom_directives.conf
```

Add the following entries to raise both hard and soft limits to 120,000 for all users and for root specifically:

```
#<domain>    <type>    <item>      <value>
*            soft      nofile      120000
*            hard      nofile      120000
root         soft      nofile      120000
root         hard      nofile      120000
```

Save and exit.

> **Note:** If copy-pasting the above into nano does not apply correctly, type the entries manually using the Tab key between columns and press Enter at the end of each line.

### Enable PAM Limits

The kernel limit file alone is not enough — PAM (Pluggable Authentication Modules) must be told to enforce these limits at login time. The `pam_limits.so` module reads `/etc/security/limits.d/` and applies the values to each session.

Navigate to the PAM configuration directory:

```bash
cd /etc/pam.d/
ls
```

Open `common-session` and append the `pam_limits.so` directive at the end of the file:

```bash
sudo nano common-session
```

Add the following line at the bottom:

```
session required    pam_limits.so
```

Then do the same for `common-session-noninteractive`, which governs non-login sessions (cron jobs, scripts, systemd services):

```bash
sudo nano common-session-noninteractive
```

Add the same line at the bottom:

```
session required    pam_limits.so
```

Save both files.

### Reboot and Confirm

Reboot to apply all changes:

```bash
sudo reboot
```

After reconnecting via SSH, verify the limits have taken effect:

```bash
ulimit -Hn   # Should now return 120000
ulimit -Sn   # Should now return 120000
```

### What Comes Next

With system-wide open file limits raised to 120,000, the individual services — Nginx, MariaDB, and PHP — each need their own file descriptor limits configured within their respective service unit files. This prevents the "too many open files" error at the application layer and is covered in later chapters.

---

## 16.4 Summary of Changes

| Area | File Modified | Change |
|---|---|---|
| TCP congestion control | `/etc/sysctl.d/custom_overrides.conf` | Verified `bbr` is active |
| File access times | `/etc/fstab` | Added `noatime` to root filesystem entry |
| Open file limits | `/etc/security/limits.d/custom_directives.conf` | Set hard/soft `nofile` to 120000 for all users |
| PAM session limits | `/etc/pam.d/common-session` | Added `session required pam_limits.so` |
| PAM non-interactive limits | `/etc/pam.d/common-session-noninteractive` | Added `session required pam_limits.so` |

---

| | | |
|:---|:---:|---:|
| [← Chapter 15 — Harden & Optimise OS Part 2](./15-harden-and-optimize-server-distribution-2.md) | [↑ Top](#chapter-16--congestion-control-file-access-times--open-file-limits) | [Chapter 17 — DNS & Cloudflare Setup →](./17-dns-cloudflare-domain-setup.md) |
