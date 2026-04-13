[← Back to Index](../index.md)

---

# Chapter 15 — Hardening & Optimising the Server Distribution — Part 2

*Tuning kernel memory behaviour via `sysctl`, hardening shared memory, disabling IPv6, and applying a comprehensive set of network layer security and performance directives — all written into a single persistent override file.*

---

## Contents

- [15.1 Swappiness and VFS Cache Pressure](#151-swappiness-and-vfs-cache-pressure)
  - [Checking the Current Defaults](#checking-the-current-defaults)
  - [Where Kernel Overrides Live](#where-kernel-overrides-live)
  - [Setting Swappiness and Cache Pressure](#setting-swappiness-and-cache-pressure)
  - [Verify the New Values are Active](#verify-the-new-values-are-active)
- [15.2 Hardening Shared Memory](#152-hardening-shared-memory)
  - [What Shared Memory Is and Why It Needs Hardening](#what-shared-memory-is-and-why-it-needs-hardening)
  - [Adding the Hardened Mount Entry to fstab](#adding-the-hardened-mount-entry-to-fstab)
- [15.3 Disabling IPv6](#153-disabling-ipv6)
  - [Why Disable IPv6](#why-disable-ipv6)
  - [Disabling IPv6 via GRUB](#disabling-ipv6-via-grub)
- [15.4 Hardening and Optimising the Network Layer](#154-hardening-and-optimising-the-network-layer)
  - [Why Tune the TCP/IP Stack](#why-tune-the-tcpip-stack)
  - [Complete custom_overrides.conf](#complete-custom_overridesconf)
  - [What Each Block Does](#what-each-block-does)
  - [Applying the Changes](#applying-the-changes)

---

## 15.1 Swappiness and VFS Cache Pressure

### Checking the Current Defaults

Before modifying any kernel parameters, inspect the current live values:

```bash
sudo sysctl -a
```

Two values are relevant to swap behaviour:

- `vm.swappiness = 60` — the default. Controls how aggressively the kernel moves process memory out of RAM and into swap. A value of 60 is appropriate for desktop machines but wasteful on a server with predictable workloads.
- `vm.vfs_cache_pressure = 100` — the default. Controls how aggressively the kernel reclaims memory used for caching directory and inode objects.

### Where Kernel Overrides Live

Navigate to `/etc/sysctl.d/`, which is the directory for drop-in kernel parameter override files:

```bash
cd /etc/sysctl.d/
ls
```

The directory already contains several system-managed files (`10-console-messages.conf`, `10-kernel-hardening.conf`, `10-network-security.conf`, etc.). These should not be modified directly. Instead, create a custom override file that the kernel loads last:

```bash
sudo nano custom_overrides.conf
```

### Setting Swappiness and Cache Pressure

Add the following to `custom_overrides.conf`:

```ini
# SWAPPINESS AND CACHE PRESSURE
vm.swappiness = 1
vm.vfs_cache_pressure = 50
```

- `vm.swappiness = 1` — tells the kernel to use swap only as a last resort. RAM is always preferred over disk, which is the right behaviour for a web server.
- `vm.vfs_cache_pressure = 50` — halves the pressure on directory/inode cache. The kernel holds filesystem metadata in memory longer, which benefits Nginx and PHP-FPM as they repeatedly access the same directory structures.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`), then reboot:

```bash
sudo reboot
```

### Verify the New Values Are Active

After reconnecting, run `sysctl -a` and locate the two parameters:

```bash
sudo sysctl -a
```

The output should now show `vm.swappiness = 1` and `vm.vfs_cache_pressure = 50`, confirming the overrides loaded correctly from `custom_overrides.conf` at boot.

> **Automation note:** These two parameters go directly into the `custom_overrides.conf` file that will accumulate all kernel tuning in subsequent steps. After writing, run `sudo sysctl -p /etc/sysctl.d/custom_overrides.conf` to apply changes without rebooting during automation runs.

---

## 15.2 Hardening Shared Memory

### What Shared Memory Is and Why It Needs Hardening

`/dev/shm` is a tmpfs-backed shared memory segment. Processes use it to exchange data quickly without going through the filesystem. Because this space is world-accessible and sits in RAM, a misconfigured or exploited process can use it to escalate privileges, execute injected code, or pivot to other processes running on the same machine.

Confirm the current state:

```bash
mount | grep shm
```

The output will show something like:

```
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,inode64)
```

The hardened options need to be enforced permanently via `fstab` so they survive every reboot.

### Adding the Hardened Mount Entry to `fstab`

Navigate to `/etc` and back up `fstab` before modifying it:

```bash
cd /etc
sudo cp fstab fstab.bak
sudo nano fstab
```

Add the following line at the end of the file, below the existing swap entry:

```
# HARDEN SHARED MEMORY
none /dev/shm tmpfs defaults,noexec,nosuid,nodev 0 0
```

The three mount options do the following:

- `noexec` — prevents any binary from being executed directly from `/dev/shm`
- `nosuid` — prevents setuid bits from taking effect on files stored there
- `nodev` — prevents device files from being created in the shared memory segment

Save and exit, then reboot to apply:

```bash
sudo reboot
```

After reconnecting, verify the hardened options are active:

```bash
mount | grep shm
```

The output should now include `noexec` in the options list, confirming the hardened mount is in effect.

---

## 15.3 Disabling IPv6

### Why Disable IPv6

On a minimal Nginx/WordPress setup that serves IPv4 traffic, keeping IPv6 enabled adds attack surface without delivering any benefit. It requires additional firewall rules, introduces complexity in Nginx, UFW, and fail2ban configuration, and consumes kernel resources for protocol stack maintenance.

Check whether IPv6 is currently active:

```bash
ip a | grep inet6
```

If IPv6 is enabled, the output will show one or more `inet6` addresses. The goal is for this command to produce no output at all.

### Disabling IPv6 via GRUB

IPv6 is disabled at the kernel level by passing a parameter to the kernel at boot through the GRUB configuration file.

```bash
cd /etc/default
sudo nano grub
```

Locate the `GRUB_CMDLINE_LINUX` line:

```bash
GRUB_CMDLINE_LINUX=""
```

Change it to:

```bash
GRUB_CMDLINE_LINUX="ipv6.disable=1"
```

Save and exit. Then regenerate the GRUB boot menu:

```bash
sudo update-grub
```

The command sources the GRUB config, finds the kernel image, and writes the updated boot entry. Reboot:

```bash
sudo reboot
```

After reconnecting, verify IPv6 is gone:

```bash
ip a | grep inet6
```

No output means IPv6 is fully disabled at the kernel level. The MOTD login banner will also no longer display any IPv6 addresses.

> **Automation note:** In a bash script, `sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="ipv6.disable=1"/' /etc/default/grub` followed by `sudo update-grub` handles this cleanly. Do not use this approach if `GRUB_CMDLINE_LINUX` already contains other options — append instead.

---

## 15.4 Hardening and Optimising the Network Layer

### Why Tune the TCP/IP Stack

The default kernel network settings are tuned for general-purpose Linux systems. A production web server has a very different traffic profile: many simultaneous short-lived HTTP connections, burst traffic, and a need to withstand common network attacks like SYN floods and IP spoofing. The parameters applied here address both security (blocking attack vectors at the kernel level) and performance (increasing buffers and connection queues so the server handles more concurrent traffic without dropping packets).

All changes are written into the same `custom_overrides.conf` file created in section 15.1, appending below the swappiness directives.

```bash
cd /etc/sysctl.d/
sudo nano custom_overrides.conf
```

### Complete `custom_overrides.conf`

The final file covering all directives introduced across this chapter:

```ini
# SWAPPINESS AND CACHE PRESSURE
vm.swappiness = 1
vm.vfs_cache_pressure = 50

# IP SPOOFING
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1

# SYN FLOOD
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# SOURCE PACKET ROUTING
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Increase number of usable ports
net.ipv4.ip_local_port_range = 1024 65535

# Increase the size of file handles and inode cache and restrict core dumps
fs.file-max = 2097152
fs.suid_dumpable = 0

# Change the number of incoming connections and incoming connections backlog
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 262144

# Increase the maximum amount of memory buffers
net.core.optmem_max = 25165824

# Increase the default and maximum send/receive buffers
net.core.rmem_default = 31457280
net.core.rmem_max = 67108864
net.core.wmem_default = 31457280
net.core.wmem_max = 67108864
```

### What Each Block Does

**IP spoofing protection** (`rp_filter = 1`): Enables reverse path filtering. The kernel validates that the source address of every incoming packet is reachable via the interface it arrived on. Packets that fail this check — the signature of a spoofed-source attack — are silently dropped.

**SYN flood protection**: A SYN flood is a denial-of-service attack where an attacker sends a high volume of TCP SYN packets with forged source addresses, exhausting the server's half-open connection table. `tcp_syncookies = 1` enables SYN cookies as a fallback: when the backlog is full, the server encodes connection state into the initial sequence number and does not allocate a socket until the handshake completes. `tcp_max_syn_backlog = 2048` increases the queue for half-open connections. `tcp_synack_retries = 2` and `tcp_syn_retries = 5` reduce the number of retries before giving up on a stalled connection, freeing resources faster.

**Source packet routing** (`accept_source_route = 0`): Disables the ability for incoming packets to specify their own routing path through the network. This is a legacy IP feature that is almost never legitimate and is routinely abused in routing attacks.

**Usable port range** (`ip_local_port_range = 1024 65535`): Expands the range of ephemeral ports available for outbound connections. This matters for high-traffic servers that open many simultaneous outbound connections.

**File handles and core dumps** (`fs.file-max = 2097152`, `fs.suid_dumpable = 0`): Increases the system-wide limit on open file descriptors, which is critical for Nginx handling many simultaneous connections. `suid_dumpable = 0` prevents setuid programs from generating core dump files, which could leak sensitive memory content to disk.

**Connection queues** (`somaxconn = 65535`, `netdev_max_backlog = 262144`): `somaxconn` sets the maximum length of the listen backlog — the queue of completed connections waiting for `accept()`. The default of 4096 is easily exhausted under load. `netdev_max_backlog` is the queue size for packets arriving faster than the kernel can process them.

**Memory buffers** (`optmem_max`, `rmem_*`, `wmem_*`): These control the kernel socket buffer sizes. Increasing them allows the kernel to buffer more data per connection, improving throughput on high-bandwidth or high-latency paths.

### Applying the Changes

Save and exit from nano, then reboot:

```bash
sudo reboot
```

After reconnecting, verify the network parameters loaded:

```bash
sudo sysctl -a | grep suid
```

Confirm `fs.suid_dumpable = 0`. If it shows `2` instead, the `apport` crash reporting service is overriding it. Disable apport and reboot again:

```bash
sudo systemctl disable apport
sudo reboot
```

Then re-confirm with `sudo sysctl -a | grep suid`.

> **What comes next:** The next chapter covers congestion control (enabling BBR), file access time optimisation (`noatime` in fstab), open file limits via `/etc/security/limits.d/`, and enforcing those limits through PAM.

---

| | | |
|:---|:---:|---:|
| [← Chapter 14 — Harden & Optimise: Timezone, Swap](./14-harden-and-optimize-server-distribution.md) | [↑ Top](#chapter-15--hardening--optimising-the-server-distribution--part-2) | [↑ Back to Index](../index.md) |
