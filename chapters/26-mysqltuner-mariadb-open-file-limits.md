[← Back to Index](../index.md)

---

# Chapter 26 — MySQLTuner & MariaDB: Raising the Open File Limit

*Running MySQLTuner to get performance recommendations for a running MariaDB instance, and raising the MariaDB open file limit via a systemd drop-in configuration file.*

---

## Contents

- [26.1 What is MySQLTuner?](#261-what-is-mysqltuner)
- [26.2 Part 1 — Running MySQLTuner](#262-part-1--running-mysqltuner)
  - [Step 1 — Clean Up the Home Directory](#step-1--clean-up-the-home-directory)
  - [Step 2 — Create a Directory and Download MySQLTuner](#step-2--create-a-directory-and-download-mysqltuner)
  - [Step 3 — Make the Script Executable and Run It](#step-3--make-the-script-executable-and-run-it)
- [26.3 Part 2 — Raising the MariaDB Open File Limit](#263-part-2--raising-the-mariadb-open-file-limit)
  - [Step 1 — Find the MariaDB Process ID and Inspect Current Limits](#step-1--find-the-mariadb-process-id-and-inspect-current-limits)
  - [Step 2 — Create the systemd Override Directory](#step-2--create-the-systemd-override-directory)
  - [Step 3 — Create the Limits Drop-In File](#step-3--create-the-limits-drop-in-file)
  - [Step 4 — Reload systemd and Restart MariaDB](#step-4--reload-systemd-and-restart-mariadb)
  - [Step 5 — Verify the New Limit is Active](#step-5--verify-the-new-limit-is-active)
- [26.4 Summary](#264-summary)

---

## 26.1 What is MySQLTuner?

MySQLTuner is a Perl script that connects to your MariaDB/MySQL server, reads the current runtime variables and status counters, and outputs a prioritised list of recommendations — things like buffer pool sizes, log file sizes, query cache settings, and more. It does not make changes automatically; it only reports what should be tuned.

> **Important:** MySQLTuner is not a tool to run continuously. It needs the server to have been running under real load for a meaningful period before its recommendations are useful. Run it every **60–90 days** as a periodic health check, not as a monitoring agent.

---

## 26.2 Part 1 — Running MySQLTuner

### Step 1 — Clean Up the Home Directory

Before setting up the MySQLTuner working directory, remove any leftover test files from previous steps to keep the home directory clean:

```bash
ls
rm fail2ban.deb
rm php_mail_test.php
ls
```

### Step 2 — Create a Directory and Download MySQLTuner

A dedicated directory keeps the MySQLTuner script organised and makes it easy to re-run or update later:

```bash
mkdir MySQLTuner
cd MySQLTuner/
```

Download the script directly from its official source using `wget`. The `-O` flag saves it with the explicit filename `mysqltuner.pl` rather than whatever name the redirect chain would produce:

```bash
wget http://mysqltuner.pl/ -O mysqltuner.pl
```

`wget` follows the HTTP 301 redirects automatically and saves the Perl script. Confirm the download completed successfully:

```bash
ls -l
```

Expected output:

```
total 260
-rw-rw-r-- 1 andrew andrew 264268 Jul 16 19:48 mysqltuner.pl
```

### Step 3 — Make the Script Executable and Run It

The downloaded file has no execute permission by default. `chmod +x` grants it:

```bash
sudo chmod +x mysqltuner.pl
ls -l
```

After `chmod`, the permission string changes from `-rw-rw-r--` to `-rwxrwxr-x`, confirming it is now executable.

Run MySQLTuner with `sudo` so it can read the MariaDB status without entering credentials manually:

```bash
sudo ./mysqltuner.pl
```

MySQLTuner connects to the local MariaDB socket, reads variables and status counters, and prints a colour-coded report with its recommendations. Review the output carefully and use the recommendations as a guide for further tuning of `50-server.cnf`.

> **Note:** MySQLTuner recommendations are server-specific. The directives tuned in Chapter 25 (InnoDB buffer pool, log file size, etc.) should be cross-referenced against what MySQLTuner reports for your specific workload and RAM.

---

## 26.3 Part 2 — Raising the MariaDB Open File Limit

### The Problem

Under sustained load, MariaDB can exhaust the number of file descriptors the OS allows a single process to hold open. When that happens, the server logs a `"too many open files"` error and begins refusing connections or failing to open table files. The fix is to raise the `LimitNOFILE` value that systemd enforces for the MariaDB service.

The workflow: **check the current limit → raise it → verify the new limit is active.**

### Step 1 — Find the MariaDB Process ID and Inspect Current Limits

Use `ps aux` filtered by `mysql` (MariaDB runs as the `mysql` user) to find the process ID:

```bash
ps aux | grep mysql
```

From the output, identify the PID in the second column of the `mysql` row. Then read the kernel-enforced limits for that process directly from `/proc`:

```bash
cat /proc/<PID>/limits
```

Look at the **Max open files** row:

```
Limit                  Soft Limit   Hard Limit   Units
...
Max open files         32768        32768        files
...
```

The default is `32768`. On a busy WordPress server with many simultaneous connections and open table files, this can be exhausted.

### Step 2 — Create the systemd Override Directory

```bash
cd /etc/systemd/system/
ls
```

There is no `mariadb.service.d/` directory by default, so create one:

```bash
sudo mkdir mariadb.service.d/
cd mariadb.service.d/
```

Drop-in directories (the `.d/` pattern) are the correct way to override individual systemd unit file directives without modifying the original package-managed unit file. Any `.conf` file placed inside this directory is merged with the main unit on startup.

### Step 3 — Create the Limits Drop-In File

```bash
sudo nano limits.conf
```

Add the following content to the file:

```ini
[Service]
LimitNOFILE=40000
```

`LimitNOFILE` maps directly to the `RLIMIT_NOFILE` kernel resource limit. Setting it to `40000` raises both the soft and hard limits that MariaDB's process will inherit when systemd starts it. Save and exit nano (`Ctrl+X`, then `Y`, then `Enter`).

### Step 4 — Reload systemd and Restart MariaDB

After creating or modifying any systemd unit or drop-in file, the daemon must re-read its configuration before the change takes effect:

```bash
sudo systemctl daemon-reload
```

Then restart MariaDB using the alias set up in Chapter 25:

```bash
mariare
```

Which expands to `sudo systemctl restart mariadb`.

### Step 5 — Verify the New Limit is Active

Find the new MariaDB PID (it changes on restart):

```bash
ps aux | grep mysql
```

Read the limits for the new PID:

```bash
cat /proc/<NEW_PID>/limits
```

Confirm the **Max open files** row now shows the updated value:

```
Limit                  Soft Limit   Hard Limit   Units
...
Max open files         40000        40000        files
...
```

Both the soft and hard limits are now `40000`, confirming the drop-in configuration was picked up correctly by systemd.

---

## 26.4 Summary

| Task | Key file / command |
|------|-------------------|
| Download MySQLTuner | `wget http://mysqltuner.pl/ -O mysqltuner.pl` |
| Make executable | `sudo chmod +x mysqltuner.pl` |
| Run MySQLTuner | `sudo ./mysqltuner.pl` |
| MariaDB drop-in dir | `/etc/systemd/system/mariadb.service.d/` |
| Limit override file | `limits.conf` → `LimitNOFILE=40000` |
| Reload & restart | `sudo systemctl daemon-reload && sudo systemctl restart mariadb` |
| Verify new limit | `ps aux \| grep mysql` → `cat /proc/<PID>/limits` |

> **Automation note:** For a bash script, the `daemon-reload` and `mariare` (restart) commands must run as separate steps after writing the drop-in file. The PID verification step can be scripted with `pgrep -o mysql` to avoid having to extract it manually from `ps aux` output.

---

| | | |
|:---|:---:|---:|
| [← Chapter 25 — Optimise MariaDB](./25-optimize-mariadb.md) | [↑ Top](#chapter-26--mysqltuner--mariadb-raising-the-open-file-limit) | [Chapter 27 — Harden & Optimise PHP →](./27-harden-and-optimize-php.md) |
