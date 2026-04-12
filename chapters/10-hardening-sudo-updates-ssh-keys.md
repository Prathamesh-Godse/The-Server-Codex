[← Back to Index](../index.md)

---

# Chapter 10 — Hardening Continued: sudo, Server Updates & SSH Key Authentication

*Completing the hardening process — using sudo as the non-root user, applying system updates, generating an SSH key pair, uploading it to the server, and disabling password authentication entirely.*

> Commands run **on the server** are prefixed with `sudo`. Commands run **on the local machine** have no prefix.

---

## Contents

- [10.1 Understanding sudo](#101-understanding-sudo)
- [10.2 Applying Server Updates](#102-applying-server-updates)
- [10.3 SSH Key Authentication Overview](#103-ssh-key-authentication-overview)
- [10.4 Step 1 — Generate the SSH Key Pair (Local Machine)](#104-step-1--generate-the-ssh-key-pair-local-machine)
- [10.5 Step 2 — Upload the Public Key to the Server](#105-step-2--upload-the-public-key-to-the-server)
- [10.6 Step 3 — Disable Password Authentication (Server)](#106-step-3--disable-password-authentication-server)
- [10.7 Step 4 — Verify Password Login is Blocked](#107-step-4--verify-password-login-is-blocked)
- [10.8 Summary of All Commands](#108-summary-of-all-commands)

---

## 10.1 Understanding `sudo`

Now that root SSH login is disabled, all privileged operations on the server are performed by the non-root user prefixed with `sudo` (superuser do). `sudo` temporarily invokes root-level privileges for a single command without requiring a full root login.

Key behaviours of `sudo`:

- Type `sudo` before any command that requires root privileges
- On first use in a session, `sudo` prompts for the **non-root user's own password** (not root's)
- The password is cached for **5 minutes** — subsequent `sudo` commands in the same session do not re-prompt
- To clear the sudo password cache immediately: `sudo -k`

```bash
# Example: running a privileged command as the non-root user
sudo systemctl restart nginx

# Clear the cached sudo password
sudo -k
```

> **Without sudo:** Running a privileged command without `sudo` as a non-root user will fail. For example, running `reboot` directly produces: `Call to Reboot failed: Interactive authentication required.` The correct command is `sudo reboot`. Similarly, opening a system config file with `nano /etc/ssh/sshd_config` without `sudo` opens it as read-only — nano shows `[ File '/etc/ssh/sshd_config' is unwritable ]` at the bottom.

---

## 10.2 Applying Server Updates

After logging in as the non-root user for the first time, the Ubuntu login banner may report pending updates or a kernel upgrade:

```
*** System restart required ***
Pending kernel upgrade!
Running kernel version:
  6.8.0-35-generic
Diagnostics:
  The currently running kernel version is not the expected kernel version 6.8.0-36-generic.
```

A kernel upgrade requires a reboot to take effect. Apply updates and reboot immediately as part of the initial setup.

### Update the package index and upgrade all packages:

```bash
sudo apt update && sudo apt upgrade -y
```

### Reboot the server to apply the new kernel:

```bash
sudo reboot
```

Output confirms the reboot is scheduled:

```
Broadcast message from root@2404 on pts/1 (Sun 2024-06-30 16:54:05 UTC):

The system will reboot now!

Connection to 95.179.129.251 closed by remote host.
Connection to 95.179.129.251 closed.
```

The SSH connection drops immediately as the server reboots. Wait 20–30 seconds before attempting to reconnect. On the first reconnect attempt you may get `Connection refused` — this is normal while the server is still coming back up:

```bash
ssh andrew@95.179.129.251
# ssh: connect to host 95.179.129.251 port 22: Connection refused
```

Try again after a few seconds. On a successful reconnect the banner confirms the updated kernel is now running:

```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-36-generic x86_64)
```

---

## 10.3 SSH Key Authentication Overview

Password authentication over SSH is vulnerable to brute-force attacks. SSH key authentication replaces the password with a cryptographic key pair, making brute-force effectively impossible. Once SSH key authentication is set up and password authentication is disabled, the only way to log in is by possessing the correct private key file.

| Step | Where | Action |
|------|-------|--------|
| 1 | Local machine | Generate the key pair with `ssh-keygen` |
| 2 | Local machine | Upload the public key to the server with `ssh-copy-id` |
| 3 | Server | Disable password authentication in `sshd_config.d/50-cloud-init.conf` |
| 4 | Local machine | Verify key-only login works, then verify password login is blocked |

> **Important:** The key pair is generated on your **local machine** (PC or Mac), not on the server. The private key never leaves your local machine. Only the public key is sent to the server.

---

## 10.4 Step 1 — Generate the SSH Key Pair (Local Machine)

Log out of the server first:

```bash
exit
```

On the **local machine**, navigate to home and generate the key pair:

```bash
cd
ssh-keygen -t rsa -b 4096
```

`-t rsa` specifies the RSA algorithm. `-b 4096` sets the key length to 4096 bits — stronger than the default 3072.

`ssh-keygen` prompts for a filename and an optional passphrase:

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/andrew/.ssh/id_rsa): .ssh/my2404server_keys
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in .ssh/my2404server_keys
Your public key has been saved in .ssh/my2404server_keys.pub
The key fingerprint is:
SHA256:KpI7HVN827zp2jmAD9cMWD52VhpzEkMPv1F9T0ZRcp4 andrew@hp
```

When prompted for the filename, a **custom name** is used instead of the default `id_rsa` — this makes it easy to identify which key belongs to which server when managing multiple servers:

```
Enter file in which to save the key (/home/andrew/.ssh/id_rsa): .ssh/my2404server_keys
```

For the passphrase, you can leave it empty (press Enter twice) for no passphrase, or enter one for an additional layer of security. If a passphrase is set, it will be required every time the key is used to connect.

### Verify the key files were created:

```bash
ls .ssh/ -l
```

```
-rw------- 1 andrew andrew  978 Jun 29 20:10 known_hosts
-rw------- 1 andrew andrew  142 Jun 29 20:10 known_hosts.old
-rw------- 1 andrew andrew 3414 Jun 30 19:09 my2404server_keys
-rw-r--r-- 1 andrew andrew  735 Jun 30 19:09 my2404server_keys.pub
```

Two files are created: the private key (`my2404server_keys`, permissions `600` — owner read/write only) and the public key (`my2404server_keys.pub`, permissions `644` — world-readable). The private key must never be shared or have its permissions loosened.

---

## 10.5 Step 2 — Upload the Public Key to the Server

`ssh-copy-id` automates the process of installing the public key into the server's `~/.ssh/authorized_keys` file. It still uses password authentication for this one-time operation:

```bash
ssh-copy-id -i .ssh/my2404server_keys.pub andrew@95.179.129.251
```

`-i` specifies the public key file to install:

```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: ".ssh/my2404server_keys.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
andrew@95.179.129.251's password:

Number of key(s) added: 1

Now try logging in the machine, with:   "ssh 'andrew@95.179.129.251'"
and check to make sure that only the key(s) you wanted were added.
```

### Test key-based login immediately:

```bash
ssh -i .ssh/my2404server_keys andrew@95.179.129.251
```

If a passphrase was set during key generation, you are prompted for it now:

```
Enter passphrase for key '.ssh/my2404server_keys':
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-36-generic x86_64)
...
andrew@2404:~$
```

Key authentication is working. The server is accepting the key without a password prompt.

---

## 10.6 Step 3 — Disable Password Authentication (Server)

With key-based login confirmed working, password authentication is now disabled. This means **only** users with the correct private key can connect — password brute-force attacks become completely ineffective.

This change is made in `/etc/ssh/sshd_config.d/50-cloud-init.conf` — the same drop-in file where `PermitRootLogin` was set earlier.

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

> **Note:** Opening this file without `sudo` fails silently — nano opens it but marks it as unwritable. Always use `sudo` for system config files.

The file currently contains:

```
PasswordAuthentication yes
PermitRootLogin no
```

Change `PasswordAuthentication yes` to `PasswordAuthentication no`:

```
PasswordAuthentication no
PermitRootLogin no
```

Save with `Ctrl+O`, confirm, then exit with `Ctrl+X`.

### Restart the SSH daemon to apply the change:

```bash
sudo systemctl restart ssh
```

Exit the session:

```bash
exit
```

---

## 10.7 Step 4 — Verify Password Login is Blocked

Test that password authentication is now rejected. The following command forces SSH to attempt password-only login (disabling public key authentication):

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no andrew@95.179.129.251
```

Expected output:

```
andrew@95.179.129.251's password:
Permission denied, please try again.
andrew@95.179.129.251's password:
Permission denied, please try again.
andrew@95.179.129.251's password:
andrew@95.179.129.251: Permission denied (publickey,password).
```

Password login is confirmed blocked. The final line `Permission denied (publickey,password)` is the definitive confirmation — the server is now accepting only public key authentication.

### Confirm key-based login still works:

```bash
ssh -i .ssh/my2404server_keys andrew@95.179.129.251
```

```
Enter passphrase for key '.ssh/my2404server_keys':
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-36-generic x86_64)
...
andrew@2404:~$
```

The server is now fully locked down to SSH key authentication only.

---

## 10.8 Summary of All Commands

```bash
# --- ON THE LOCAL MACHINE ---

# Generate SSH key pair (4096-bit RSA, custom filename)
cd
ssh-keygen -t rsa -b 4096
# When prompted for filename: .ssh/my2404server_keys

# Verify key files created
ls .ssh/ -l

# Upload public key to server (one-time, uses password)
ssh-copy-id -i .ssh/my2404server_keys.pub username@server_ip

# Connect using SSH key
ssh -i .ssh/my2404server_keys username@server_ip

# Test that password login is now blocked
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no username@server_ip

# --- ON THE SERVER (as non-root user with sudo) ---

# Apply system updates and reboot
sudo apt update && sudo apt upgrade -y
sudo reboot

# Disable password authentication
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
# Change: PasswordAuthentication yes → PasswordAuthentication no

# Restart SSH to apply
sudo systemctl restart ssh
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 9 — Server Hardening](./09-server-hardening.md) | [↑ Top](#chapter-10--hardening-continued-sudo-server-updates--ssh-key-authentication) | [↑ Back to Index](../index.md) |
