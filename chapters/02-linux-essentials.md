[← Back to Index](../index.md)

---

# Chapter 2 — Linux Essentials

*Core Linux concepts and commands used throughout the server setup — distributions, the shell, SSH, terminal usage, and essential navigation.*

---

## Contents

- [2.1 Linux Distributions](#21-linux-distributions)
- [2.2 The Shell](#22-the-shell)
- [2.3 Secure Shell (SSH)](#23-secure-shell-ssh)
- [2.4 The Terminal Emulator](#24-the-terminal-emulator)
- [2.5 Reading the Terminal Prompt](#25-reading-the-terminal-prompt)
- [2.6 Navigating with ls](#26-navigating-with-ls)

---

## 2.1 Linux Distributions

A Linux distribution (distro) is a complete operating system built around the Linux kernel, bundled with a package manager, init system, and default software. Common server distros include Ubuntu, Debian, and Red Hat — each with its own trade-offs. This project uses **Ubuntu** because of its wide VPS support, large community, and predictable LTS release cycle.

**Ubuntu versioning:**

| Type | Cadence | Example |
|------|---------|---------|
| Major LTS release | Every 2 years | 20.04, 22.04, 24.04 |
| Minor point release | As needed (bug/security fixes) | 22.04.1, 22.04.2 |

This project uses **Ubuntu 24.04 LTS (Noble Numbat)**, supported until 2029 with standard security updates.

---

## 2.2 The Shell

The shell is the command interpreter that sits between the user and the operating system kernel. It accepts text commands, executes them, and returns output. On Ubuntu servers the default shell is **Bash** (Bourne Again Shell).

There are two ways to interact with a shell:

- **Locally** — through a terminal emulator on a desktop machine
- **Remotely** — over an encrypted SSH connection to a server (the primary method used in this project)

---

## 2.3 Secure Shell (SSH)

SSH is a cryptographic network protocol used to securely access and administer remote servers over an untrusted network. All data transmitted between the client and the server is encrypted.

To connect to a remote server:

```bash
ssh username@server_ip_address
```

For example:

```bash
ssh ubuntu@192.168.1.100
```

SSH key-based authentication is used throughout this project instead of password authentication. This involves generating an RSA or ED25519 key pair — the public key is placed on the server, and the private key stays on the local machine.

```bash
# Generate an SSH key pair on your local machine
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy the public key to the server
ssh-copy-id username@server_ip_address
```

Once the key is in place, connecting to the server no longer requires a password.

---

## 2.4 The Terminal Emulator

A terminal emulator is a software application that provides a terminal window within a graphical environment. It bridges the user interface and the shell, allowing text-based commands to be entered and executed.

Common terminal emulators by platform:

| Platform | Examples |
|----------|---------|
| Linux | GNOME Terminal, Konsole, Alacritty |
| macOS | Terminal, iTerm2 |
| Windows | Windows Terminal, PuTTY |

For remote server work, the terminal emulator is used locally to open an SSH session into the server. All server-side commands run on the remote machine — the terminal is just the window.

---

## 2.5 Reading the Terminal Prompt

When logged into a server, the shell displays a prompt before every command. Understanding the prompt helps confirm which user and machine you are operating on — important when switching between root and non-root contexts.

```
root@my2404server:~#
```

| Part | Meaning |
|------|---------|
| `root` | Currently logged-in username |
| `@` | Separator |
| `my2404server` | Hostname of the server |
| `:` | Separator |
| `~` | Current directory (`~` means home directory) |
| `#` | Root user indicator |
| `$` | Non-root user indicator |

Example prompts:

```bash
# Root user — full system privileges
root@my2404server:~#

# Non-root user — limited privileges
ubuntu@my2404server:~$
```

> Throughout this project, most work is done as a non-root user with `sudo` for privileged operations. Avoid running everything as root directly.

---

## 2.6 Navigating with `ls`

`ls` lists the contents of a directory. It is one of the most frequently used commands when navigating a server and verifying file and directory states after configuration changes.

**Basic usage:**

```bash
# List files in current directory
ls

# List including hidden files (files starting with .)
ls -a

# Long format — shows permissions, owner, size, and modification date
ls -l

# Long format including hidden files
ls -la

# Same as above (flag order is interchangeable)
ls -al

# Long format, all files, sorted in reverse order
ls -alr

# Recursive listing — descends into subdirectories
ls -alR
```

**Example output of `ls -la`:**

```
total 44
drwxr-x--- 6 andrew andrew 4096 Jun 18 19:15 .
drwxr-xr-x 3 root   root   4096 Jun  5  2023 ..
-rw------- 1 andrew andrew 4155 Jun 18 19:43 .bash_history
-rw-r--r-- 1 andrew andrew  220 Jun  5  2023 .bash_logout
-rw-r--r-- 1 andrew andrew 3771 Jun  5  2023 .bashrc
drwx------ 2 andrew andrew 4096 Jun  5  2023 .cache
drwx------ 3 andrew andrew 4096 Jul  9  2023 .config
drwxrwxr-x 3 andrew andrew 4096 Jul  9  2023 .local
-rw-r--r-- 1 andrew andrew  807 Jun  5  2023 .profile
drwx------ 2 andrew andrew 4096 Jun 18 19:14 .ssh
```

**Reading the long listing columns:**

| Column | Meaning |
|--------|---------|
| `drwxr-x---` | File type and permissions |
| `6` | Number of hard links |
| `andrew` | Owner (user) |
| `andrew` | Owner (group) |
| `4096` | Size in bytes |
| `Jun 18 19:15` | Last modified date and time |
| `.` | File or directory name |

**Permission string breakdown:**

```
d  rwx  r-x  ---
│   │    │    │
│   │    │    └── Other: no permissions
│   │    └─────── Group: read + execute
│   └──────────── Owner: read + write + execute
└──────────────── d = directory, - = file
```

**Combining flags:**

Multiple flags can be combined in any order — `ls -la`, `ls -al`, `ls -laR`, and `ls -aRl` all work equivalently. The order of flags does not affect the result.

---

| | | |
|:---|:---:|---:|
| [← Chapter 1 — Project Overview](./01-nginx-wordpress-server.md) | [↑ Top](#chapter-2--linux-essentials) | [Chapter 3 — Linux File System →](./03-linux-file-system.md) |
