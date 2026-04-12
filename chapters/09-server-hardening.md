[← Back to Index](../index.md)

---

# Chapter 9 — Server Hardening

*The complete hardening process carried out immediately after first root login — moving from wide-open defaults to a locked-down server where root SSH access is disabled and all administration runs through a named non-root user.*

> All commands in this chapter are run while logged in as **root** over SSH.

---

## Contents

- [9.1 Understanding the Shell Prompt](#91-understanding-the-shell-prompt)
- [9.2 Step 1 — Change the Root Password](#92-step-1--change-the-root-password)
- [9.3 Step 2 — Create a New Non-Root User](#93-step-2--create-a-new-non-root-user)
- [9.4 Step 3 — Remove Any Pre-existing Default Users](#94-step-3--remove-any-pre-existing-default-users)
- [9.5 Step 4 — Grant the New User sudo Privileges](#95-step-4--grant-the-new-user-sudo-privileges)
- [9.6 Step 5 — Set a Password for the Non-Root User](#96-step-5--set-a-password-for-the-non-root-user)
- [9.7 Step 6 — Disable Root Login via SSH](#97-step-6--disable-root-login-via-ssh)
- [9.8 Step 7 — Exit Root and Log in as the Non-Root User](#98-step-7--exit-root-and-log-in-as-the-non-root-user)
- [9.9 Summary of All Hardening Commands](#99-summary-of-all-hardening-commands)

---

## 9.1 Understanding the Shell Prompt

The terminal prompt format is `user@hostname:directory#` (or `$` for non-root users). This makes it immediately clear which user is active and where in the filesystem you are:

```
root@2404:~#       # logged in as root on the server
andrew@hp:~$       # logged in as 'andrew' on the local machine
andrew@2404:~$     # logged in as 'andrew' on the server
```

The `#` symbol indicates a root shell. The `$` symbol indicates a non-root shell.

---

## 9.2 Step 1 — Change the Root Password

The root password set by Vultr at provisioning time is auto-generated and should be replaced immediately with a strong password of your own choosing.

```bash
passwd
```

You will be prompted to enter and confirm the new password. No characters are shown while typing:

```
New password:
Retype new password:
passwd: password updated successfully
```

---

## 9.3 Step 2 — Create a New Non-Root User

Running day-to-day server administration as root is dangerous — a single mistaken command can cause irreversible damage with no safeguards. A dedicated non-root user account is created for all ongoing work.

```bash
adduser username
```

Replace `username` with the name you want to use. The `adduser` command is interactive and handles everything — creating the user, assigning a UID/GID, creating the home directory at `/home/username`, and copying skeleton files from `/etc/skel`:

```
info: Adding user 'andrew' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group 'andrew' (1002) ...
info: Adding new user 'andrew' (1002) with group 'andrew (1002)' ...
info: Creating home directory '/home/andrew' ...
info: Copying files from '/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for andrew
Enter the new value, or press ENTER for the default
        Full Name []:
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] y
info: Adding new user 'andrew' to supplemental / extra groups 'users' ...
info: Adding user 'andrew' to group 'users' ...
```

Press Enter through all the optional user information fields — these are not needed for a server account. Type `y` when asked to confirm.

---

## 9.4 Step 3 — Remove Any Pre-existing Default Users

Some cloud providers pre-create default user accounts (such as `ubuntu` or `linuxuser`) on freshly provisioned instances. These accounts are unnecessary and should be removed before setting up your own user.

### Check what users exist in /home:

```bash
cd /home
ls -l
```

Example output showing pre-existing accounts that need to be removed:

```
drwxr-x--- 2 andrew    andrew    4096 Jun 29 18:19 andrew
drwxr-x--- 3 linuxuser linuxuser 4096 Jun 29 17:44 linuxuser
drwxr-x--- 4 ubuntu    ubuntu    4096 Jun 14 17:29 ubuntu
```

### Remove each unwanted user and their home directory:

```bash
deluser linuxuser --remove-home
deluser ubuntu --remove-home
```

The `--remove-home` flag deletes the user's home directory along with the account. Verify only your user remains:

```bash
ls -l /home
```

```
drwxr-x--- 2 andrew andrew 4096 Jun 29 18:19 andrew
```

---

## 9.5 Step 4 — Grant the New User sudo Privileges

The new user needs the ability to run commands as root using `sudo`. This is done by editing the `/etc/sudoers` file — the file that controls who can use `sudo` and with what permissions.

### Set nano as the Default Editor for visudo

`visudo` is the dedicated command for safely editing the sudoers file. It validates the syntax before saving, which prevents you from locking yourself out with a broken configuration. Set it to nano first:

```bash
update-alternatives --config editor
```

A menu of available editors is shown:

```
There are 4 choices for the alternative editor (providing /usr/bin/editor).

  Selection    Path                 Priority   Status
------------------------------------------------------------
* 0            /bin/nano             40        auto mode
  1            /bin/ed             -100        manual mode
  2            /bin/nano             40        manual mode
  3            /usr/bin/vim.basic    30        manual mode
  4            /usr/bin/vim.tiny     15        manual mode

Press <enter> to keep the current choice[*], or type selection number:
```

Press Enter to keep nano (already auto mode), or type `2` and press Enter to explicitly select it.

### Open the sudoers file with visudo:

```bash
visudo
```

This opens `/etc/sudoers` in nano. Navigate to the **User privilege specification** section and add your new user on a new line directly below the root entry:

```
# User privilege specification
root    ALL=(ALL:ALL) ALL
andrew  ALL=(ALL:ALL) ALL
```

The `ALL=(ALL:ALL) ALL` directive means: on all hosts, as any user and any group, run any command. This gives the user full sudo access equivalent to root.

Save with `Ctrl+O`, confirm the filename, then exit with `Ctrl+X`.

> **Why visudo and not nano directly?** `visudo` locks the file during editing to prevent simultaneous edits and validates the syntax before writing. A syntax error in `/etc/sudoers` can make `sudo` completely unusable — visudo prevents this.

---

## 9.6 Step 5 — Set a Password for the Non-Root User (if needed)

If you need to change or set the non-root user's password from root, use `passwd` with the username:

```bash
passwd andrew
```

```
New password:
Retype new password:
passwd: password updated successfully
```

---

## 9.7 Step 6 — Disable Root Login via SSH

With a working non-root sudo user in place, root SSH login is now disabled. This is the most important hardening step — it eliminates the single largest brute-force attack vector on any Linux server.

### The SSH Configuration Layout in Ubuntu 24.04

On Ubuntu 24.04, the SSH daemon configuration is split across two locations:

```
/etc/ssh/
├── sshd_config              # main SSH daemon config file
└── sshd_config.d/
    └── 50-cloud-init.conf   # cloud provider overrides (Vultr injects this)
```

The `Include /etc/ssh/sshd_config.d/*.conf` directive at the top of `sshd_config` means any `.conf` file in the `sshd_config.d/` directory is automatically loaded and its directives override those in the main file. The cloud-init file is where `PermitRootLogin` must be changed.

### Edit the main sshd_config:

```bash
nano /etc/ssh/sshd_config
```

Find and change:

```
PermitRootLogin yes
```

to:

```
PermitRootLogin no
```

Save and exit.

### Edit the cloud-init drop-in config:

```bash
nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

The file contains:

```
PasswordAuthentication yes
PermitRootLogin yes
```

Change `PermitRootLogin yes` to `PermitRootLogin no`:

```
PasswordAuthentication yes
PermitRootLogin no
```

Save with `Ctrl+O` and exit with `Ctrl+X`. Verify the change:

```bash
cat /etc/ssh/sshd_config.d/50-cloud-init.conf
```

```
PasswordAuthentication yes
PermitRootLogin no
```

### Restart the SSH daemon to apply the changes:

```bash
systemctl restart ssh
```

> **Critical:** Do not close your current root session until you have confirmed the non-root user can log in successfully. If something goes wrong, you still have the active root session to fix it.

---

## 9.8 Step 7 — Exit Root and Log in as the Non-Root User

Log out of the root session:

```bash
exit
```

```
logout
Connection to 95.179.129.251 closed.
```

### Verify root login is now blocked:

```bash
ssh root@95.179.129.251
```

```
root@95.179.129.251's password:
Permission denied, please try again.
root@95.179.129.251's password:
Permission denied, please try again.
```

Root login is confirmed disabled.

### Log in as the non-root user:

```bash
ssh andrew@95.179.129.251
```

```
andrew@95.179.129.251's password:
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-35-generic x86_64)
...
andrew@2404:~$
```

The `$` prompt confirms you are now logged in as the non-root user. From this point forward, all server administration is done as this user, prefixing privileged commands with `sudo`.

---

## 9.9 Summary of All Hardening Commands

```bash
# 1. Change the root password
passwd

# 2. Create a new non-root user
adduser username

# 3. Check for and remove any pre-existing default users
cd /home && ls -l
deluser linuxuser --remove-home
deluser ubuntu --remove-home

# 4. Set nano as default editor, then grant sudo privileges
update-alternatives --config editor
visudo
# Add under root: username ALL=(ALL:ALL) ALL

# 5. Change non-root user password from root if needed
passwd username

# 6. Disable root SSH login (both files)
nano /etc/ssh/sshd_config
# Change: PermitRootLogin yes → PermitRootLogin no

nano /etc/ssh/sshd_config.d/50-cloud-init.conf
# Change: PermitRootLogin yes → PermitRootLogin no

systemctl restart ssh

# 7. Exit root and verify
exit
ssh root@server_ip        # should be denied
ssh username@server_ip    # should succeed
```

---

| | | |
|:---|:---:|---:|
| [← Chapter 8 — First Login & Hardening Intro](./08-first-login-and-hardening-intro.md) | [↑ Top](#chapter-9--server-hardening) | [Chapter 10 — sudo, Updates & SSH Keys →](./10-hardening-sudo-updates-ssh-keys.md) |
