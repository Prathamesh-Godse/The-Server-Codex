[← Back to Index](../index.md)

---

# Chapter 8 — First Login via SSH & Hardening Introduction

*Connecting to the server for the first time as root, managing the known_hosts file, and understanding what server hardening involves and why it matters.*

---

## Contents

- [8.1 Prerequisites](#81-prerequisites)
- [8.2 First SSH Login as Root](#82-first-ssh-login-as-root)
  - [Connecting to the Server](#connecting-to-the-server)
  - [First-Connection Fingerprint Prompt](#first-connection-fingerprint-prompt)
  - [Successful Login Output](#successful-login-output)
  - [Logging Out](#logging-out)
- [8.3 Managing the known_hosts File](#83-managing-the-known_hosts-file)
  - [Viewing the known_hosts File](#viewing-the-known_hosts-file)
  - [Removing a Stale Fingerprint](#removing-a-stale-fingerprint)
- [8.4 What is Server Hardening?](#84-what-is-server-hardening)
- [8.5 Summary of Commands](#85-summary-of-commands)

---

## 8.1 Prerequisites

Before attempting to connect, ensure the following are in place:

- A server instance has been created on Vultr running Ubuntu 24.04 LTS
- A terminal is open on the local machine (Linux/macOS Terminal or Git Bash on Windows)
- The server's IP address, root username, and root password are available from the Vultr control panel (under the server's **Overview** tab)

---

## 8.2 First SSH Login as Root

The server's IP address, username (`root`), and initial root password are retrieved from the Vultr control panel. The first connection is made using password authentication.

### Connecting to the Server

```bash
ssh root@server_ip_address
```

Replace `server_ip_address` with the actual IP shown in the Vultr dashboard. For example:

```bash
ssh root@95.179.129.251
```

### First-Connection Fingerprint Prompt

On the very first connection, SSH does not yet have the server's fingerprint stored locally. It displays the server's fingerprint and asks you to verify and accept it:

```
The authenticity of host '95.179.129.251 (95.179.129.251)' can't be established.
ED25519 key fingerprint is SHA256:6ZlyHkMtVdLkKUto1BSNG6i5o8xH9GotC2geJN9IzS4.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '95.179.129.251' (ED25519) to the list of known hosts.
```

Type `yes` and press Enter. The fingerprint is saved to `~/.ssh/known_hosts` on the local machine. All future connections to this IP will be verified silently against this stored fingerprint.

After accepting, enter the root password when prompted:

```
root@95.179.129.251's password:
```

> **Note:** When typing a password in the terminal, no characters or asterisks appear on screen. This is normal — the input is still being recorded. Type the password and press Enter.

### Successful Login Output

On successful authentication, Ubuntu displays a system information banner:

```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-35-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

System information as of Sat Jun 29 06:09:55 PM UTC 2024

  System load:  0.0               Processes:             128
  Usage of /:   25.1% of 22.88GB  Users logged in:       0
  Memory usage: 30%               IPv4 address for enp1s0: 95.179.129.251
  Swap usage:   0%

Last login: Sat Jun 29 18:08:41 2024 from 41.4.15.115
root@2404:~#
```

The prompt changes to `root@2404:~#`, confirming you are logged in as root on the remote server. The `#` symbol indicates a root shell.

### Logging Out

To close the SSH session and return to the local machine:

```bash
exit
```

Output:

```
logout
Connection to 95.179.129.251 closed.
```

---

## 8.3 Managing the `known_hosts` File

After the first connection, the server's fingerprint is stored in `~/.ssh/known_hosts` on the local machine.

### Viewing the known_hosts File

Navigate to the `.ssh` directory and inspect its contents:

```bash
cd ~/.ssh
ls -l
cat known_hosts
```

Each line in `known_hosts` corresponds to a server the local machine has previously connected to. When you reconnect, SSH compares the server's current fingerprint against the stored entry — if they match, the connection proceeds silently.

### Removing a Stale Fingerprint

If a server is rebuilt or its OS is reinstalled, its SSH host keys change. The next connection attempt will fail with a `REMOTE HOST IDENTIFICATION HAS CHANGED` warning. Remove the stale entry using `ssh-keygen -R`:

```bash
ssh-keygen -R 95.179.129.251
```

Output:

```
# Host 95.179.129.251 found: line 1
# Host 95.179.129.251 found: line 2
# Host 95.179.129.251 found: line 3
/home/andrew/.ssh/known_hosts updated.
Original contents retained as /home/andrew/.ssh/known_hosts.old
```

`ssh-keygen -R` removes all entries for the specified host and automatically backs up the original file as `known_hosts.old`. The next `ssh` connection will prompt for fingerprint acceptance again as if it were a first connection.

---

## 8.4 What is Server Hardening?

**Server hardening** is the process of reducing the attack surface of a server by eliminating unnecessary exposure and enforcing stricter access controls. A freshly provisioned server has wide-open defaults that are unsuitable for production use.

The hardening steps performed immediately after first login, while still connected as root, are:

| Step | Action |
|------|--------|
| 1 | Change the default root password |
| 2 | Create a new non-root user account |
| 3 | Grant that non-root user sudo (root-equivalent) privileges |
| 4 | Disable direct root login via SSH |
| 5 | Log out as root and log back in as the non-root user |

These steps are covered in detail in the following chapter. The goal is to arrive at a state where:

- Root login over SSH is disabled entirely
- All administrative work is done via a named non-root user with `sudo` access
- The attack surface of the SSH daemon is significantly reduced

> **Why disable root login?** The `root` account is a known, fixed target. Every automated brute-force bot that scans the internet for open SSH ports attempts to log in as `root`. Disabling root SSH login forces attackers to first guess a valid username — a significant additional barrier.

---

## 8.5 Summary of Commands

```bash
# Connect to server for the first time as root
ssh root@server_ip_address

# Accept fingerprint on first connection: type 'yes'

# Log out of the server
exit

# View local SSH known hosts file
ls -l ~/.ssh/
cat ~/.ssh/known_hosts

# Remove a stale fingerprint for a rebuilt server
ssh-keygen -R server_ip_address
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 7 — Server Provisioning](./07-server-provisioning.md) | [↑ Top](#chapter-8--first-login-via-ssh--hardening-introduction) | [Chapter 9 — Server Hardening →](./09-server-hardening.md) |
