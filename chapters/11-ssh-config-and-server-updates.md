[← Back to Index](../index.md)

---

# Chapter 11 — SSH Config File & Server Updates

*Simplifying server access with a local SSH config alias, and running the standard post-provisioning update sequence to bring all packages current before further configuration.*

---

## Contents

- [11.1 Why a Config File](#111-why-a-config-file)
- [11.2 Config File Parameters](#112-config-file-parameters)
- [11.3 Creating the Config File](#113-creating-the-config-file)
- [11.4 Connecting via the Alias](#114-connecting-via-the-alias)
- [11.5 Server Update Sequence](#115-server-update-sequence)
- [11.6 Reconnecting After Reboot](#116-reconnecting-after-reboot)
- [11.7 Quick Reference](#117-quick-reference)

---

## 11.1 Why a Config File

Connecting to a server using SSH key authentication normally requires specifying the private key path, the remote user, and the IP every time:

```bash
ssh -i ~/.ssh/private_key user@ip
```

Instead of repeating this, an SSH config file stored at `~/.ssh/config` on the **local machine** lets you define a named alias (a `Host` entry) for each server. After configuring it, logging in is as simple as:

```bash
ssh alias
```

> **Note:** The config file is created and lives on your local PC or Mac — not on the remote server. It goes inside the `~/.ssh/` directory.

---

## 11.2 Config File Parameters

Each entry in `~/.ssh/config` supports the following key directives:

| Directive | Purpose |
|---|---|
| `Host` | The alias name you'll type to connect |
| `HostName` | The actual IP address or domain of the server |
| `User` | The remote username to log in as |
| `IdentityFile` | Path to the private key on your local machine |
| `ServerAliveInterval` | How often (in seconds) to send a keepalive packet to prevent timeouts |
| `ServerAliveCountMax` | How many unanswered keepalives before dropping the connection |

---

## 11.3 Creating the Config File

Navigate to the home directory on your local machine, then open (or create) the config file with nano:

```bash
cd
nano .ssh/config
```

Add an entry for the server:

```
Host 2404
    HostName 95.179.129.251
    User andrew
    IdentityFile ~/.ssh/my2404server_keys
    ServerAliveInterval 60
    ServerAliveCountMax 120
```

- `Host 2404` — the alias; running `ssh 2404` will use this block
- `ServerAliveInterval 60` — sends a keepalive every 60 seconds
- `ServerAliveCountMax 120` — gives up after 120 missed keepalives (2 hours of no response)

Save and exit nano (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 11.4 Connecting via the Alias

With the config saved, log into the server using just the alias:

```bash
ssh 2404
```

SSH picks up the key, user, and host automatically from the config block. After entering the key passphrase, you land directly on the server:

```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-36-generic x86_64)
...
andrew@2404:~$
```

The tab title in the terminal also changes to reflect the active session (`andrew@2404`), confirming the alias resolved correctly.

---

## 11.5 Server Update Sequence

Immediately after provisioning or accessing a fresh server, the package list and installed packages need to be brought up to date. This is a four-step sequence run on the **remote server**.

**Step 1 — Refresh the package index**

```bash
sudo apt update
```

This fetches the latest package metadata from the configured APT repositories. It doesn't install anything — it just updates the local list of what's available and what has upgrades pending.

**Step 2 — Upgrade installed packages**

```bash
sudo apt upgrade
```

APT will show a summary of what will be installed and upgraded, along with disk space requirements, and prompt for confirmation (`[Y/n]`). Enter `Y` to proceed.

During the upgrade, the system may detect a kernel version mismatch — the running kernel differs from the newly installed one. This is expected and is resolved by the reboot in step 4.

**Step 3 — Remove obsolete packages**

```bash
sudo apt autoremove
```

Cleans up packages that were pulled in as dependencies but are no longer required by any installed package.

**Step 4 — Reboot the server**

```bash
sudo reboot
```

A reboot is necessary after kernel upgrades to load the new kernel. The SSH session will drop:

```
Connection to 95.179.129.251 closed by remote host.
Connection to 95.179.129.251 closed.
```

---

## 11.6 Reconnecting After Reboot

Once the server finishes rebooting (usually 30–60 seconds), reconnect using the alias:

```bash
ssh 2404
```

The server is now fully updated and ready for the next configuration steps.

---

## 11.7 Quick Reference

```bash
# --- Local machine: create/edit SSH config ---
cd
nano .ssh/config

# --- SSH config block structure ---
# Host <alias>
#     HostName <server_ip>
#     User <username>
#     IdentityFile ~/.ssh/<private_key_name>
#     ServerAliveInterval 60
#     ServerAliveCountMax 120

# --- Connect using alias ---
ssh 2404

# --- On the server: full update sequence ---
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo reboot

# --- Reconnect after reboot ---
ssh 2404
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 10 — sudo, Updates & SSH Keys](./10-hardening-sudo-updates-ssh-keys.md) | [↑ Top](#chapter-11--ssh-config-file--server-updates) | [Chapter 12 — Firewall: UFW & Cloud →](./12-firewall-ufw-and-cloud.md) |
