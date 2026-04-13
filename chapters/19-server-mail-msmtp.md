[← Back to Index](../index.md)

---

# Chapter 19 — Server Mail Without SMTP Plugins

*Configuring msmtp as a lightweight SMTP relay so WordPress can send transactional email through Gmail without any plugin — wiring the server's `mail()` function directly to a trusted relay.*

---

## Contents

- [19.1 Why Not Run a Full Mail Server?](#191-why-not-run-a-full-mail-server)
- [19.2 The Solution: msmtp + Gmail SMTP](#192-the-solution-msmtp--gmail-smtp)
- [19.3 Prerequisites: Gmail App Password](#193-prerequisites-gmail-app-password)
- [19.4 Install msmtp](#194-install-msmtp)
- [19.5 Configure msmtp for Command-Line Use](#195-configure-msmtp-for-command-line-use)
  - [Create the User Config File](#create-the-user-config-file)
  - [Create the msmtp Log File](#create-the-msmtp-log-file)
- [19.6 Test Command-Line Mail Sending](#196-test-command-line-mail-sending)
- [19.7 Configure msmtp for PHP (www-data)](#197-configure-msmtp-for-php-www-data)
  - [Copy the User Config to /etc/](#copy-the-user-config-to-etc)
  - [Update the Log File Path in /etc/msmtprc](#update-the-log-file-path-in-etcmsmtprc)
  - [Create the PHP msmtp Log File](#create-the-php-msmtp-log-file)
- [19.8 Test PHP Mail Sending](#198-test-php-mail-sending)
- [19.9 Summary](#199-summary)

---

## 19.1 Why Not Run a Full Mail Server?

Setting up a proper MTA (Postfix, Sendmail, etc.) on a VPS requires configuring DKIM, SPF, DMARC, reverse DNS, and ongoing administration. It is complex, time-consuming, and easy to get wrong in a way that causes all outbound mail to land in spam. This is not a practical option for a single-site WordPress server.

---

## 19.2 The Solution: msmtp + Gmail SMTP

**msmtp** is a minimal SMTP client that hands outbound mail off to an external SMTP server. Using Gmail's SMTP server as the relay is free, reliable, and keeps mail out of spam because Google's sending reputation does the heavy lifting.

**Gmail SMTP limits (free tier):**
- Up to 100 recipients per message sent via SMTP
- Up to 100 emails per day

For most WordPress sites this is more than adequate. If a site grows beyond 100 emails per day, the msmtp configuration can be updated to point at a paid transactional mail provider (smtp2go, Sendgrid, Mailgun, Mailjet) — the config change is minimal.

> **Note on sender address:** For a `mail@yourdomain.com` from address (rather than a Gmail address), a dedicated third-party mail service is needed. Running a full mail server on a VPS just for a branded sender address is not recommended.

---

## 19.3 Prerequisites: Gmail App Password

Gmail no longer allows plain account passwords for SMTP access. A **Google App Password** is required:

1. Log in to [https://myaccount.google.com/](https://myaccount.google.com/)
2. Navigate to **Security → 2-Step Verification** (must be enabled first)
3. Scroll to **App passwords**, create a new one (e.g. named "server-msmtp"), and copy the generated 16-character password

This app password is what goes into the msmtp config file — not the regular Gmail password.

---

## 19.4 Install msmtp

Update the package list and install msmtp along with `msmtp-mta`, which makes msmtp act as a drop-in replacement for the system's `sendmail` binary. This is what allows PHP's `mail()` function to route through msmtp transparently.

```bash
sudo apt update
sudo apt install msmtp msmtp-mta
```

During installation a prompt asks whether to enable AppArmor support for msmtp. Select **No** — the AppArmor profile covers common use cases but has known edge cases that produce confusing permission denied errors.

---

## 19.5 Configure msmtp for Command-Line Use

Two separate config files are required — one for command-line use by the admin user, and one for PHP running as `www-data`:

| File | Location | Used by |
|---|---|---|
| `.msmtprc` | `~/` (home directory) | Command-line mail sending |
| `msmtprc` | `/etc/` | PHP / `www-data` user |

### Create the User Config File

Navigate to the home directory and create the config:

```bash
cd
nano .msmtprc
```

Populate it with the following, replacing the placeholder values with your Gmail address and app password:

```
defaults
# TLS DIRECTIVES
tls on
tls_starttls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
# LOGGING
logfile ~/.msmtp.log

# SET ACCOUNT NAME
account MAIL
# HOST INFORMATION
host smtp.gmail.com
port 587
auth on
user your.address@gmail.com
password your-16-char-app-password
from your.address@gmail.com

# SET DEFAULT ACCOUNT NAME
account default : MAIL
```

Save and exit. Then lock down the permissions — msmtp will refuse to run if the config file is world-readable:

```bash
sudo chmod 600 .msmtprc
```

### Create the msmtp Log File

```bash
cd
touch .msmtp.log
sudo chown $USER:$USER .msmtp.log
sudo chmod 660 .msmtp.log
```

---

## 19.6 Test Command-Line Mail Sending

Send a test email directly from the terminal:

```bash
msmtp recipient@example.com
```

Press **Enter**, type a short test message body, press **Enter** again, then press **Ctrl+D** to terminate msmtp and dispatch the email. Check the inbox to confirm delivery.

Verify the send was logged successfully:

```bash
cat .msmtp.log
```

A successful log entry looks like:

```
Jul 11 19:44:20 host=smtp.gmail.com tls=on auth=on user=... smtpstatus=250 ... exitcode=EX_OK
```

---

## 19.7 Configure msmtp for PHP (`www-data`)

PHP-FPM runs as `www-data`, which has no home directory and cannot read `~/.msmtprc`. A separate config file is placed in `/etc/` and owned by `www-data`.

### Copy the User Config to `/etc/`

```bash
sudo cp /home/$USER/.msmtprc /etc/msmtprc
cd /etc
sudo chown www-data:www-data msmtprc
sudo chmod 660 msmtprc
```

### Update the Log File Path in `/etc/msmtprc`

The home-directory log path (`~/.msmtp.log`) is not valid for `www-data`. Open the `/etc/` copy and change only the `logfile` directive:

```bash
sudo nano /etc/msmtprc
```

Change:

```
logfile ~/.msmtp.log
```

To:

```
logfile /var/log/msmtp.log
```

Everything else in the file stays the same as `~/.msmtprc`.

### Create the PHP msmtp Log File

```bash
cd /var/log/
sudo touch msmtp.log
sudo chown www-data:adm msmtp.log
sudo chmod 640 msmtp.log
```

---

## 19.8 Test PHP Mail Sending

The PHP `mail()` function must be tested running as `www-data` — the same user WordPress will use. To allow `www-data` to access the test script in the home directory, the home directory needs to be briefly world-executable:

```bash
cd /home
sudo chmod 755 $USER/
```

Create a test PHP script in the home directory:

```bash
cd
nano php_mail_test.php
```

Contents:

```php
<?php
    ini_set( 'display_errors', 1 );
    error_reporting( E_ALL );
    $from = "your.address@gmail.com";   // Must match 'from' in /etc/msmtprc
    $to = "recipient@example.com";
    $subject = "PHP Mail Test";
    $message = "Testing PHP Mail functionality";
    $headers = "From:" . $from;
    mail($to, $subject, $message, $headers);
    echo "Test email sent";
?>
```

Run it as `www-data`:

```bash
sudo -u www-data php php_mail_test.php
```

If the output is `Test email sent` with no PHP errors, and the email arrives in the inbox, the setup is working. Check `/var/log/msmtp.log` as well to confirm the PHP send is logged there.

Once confirmed, restore the home directory permissions:

```bash
cd /home
sudo chmod 750 $USER/
```

---

## 19.9 Summary

| Component | File | Purpose |
|---|---|---|
| msmtp user config | `~/.msmtprc` | Command-line mail via admin user |
| msmtp PHP config | `/etc/msmtprc` | PHP `mail()` via `www-data` |
| User mail log | `~/.msmtp.log` | Tracks command-line sends |
| PHP mail log | `/var/log/msmtp.log` | Tracks PHP/WordPress sends |

WordPress will now send transactional email through msmtp and Gmail without requiring any SMTP plugin. To switch to a third-party provider or use a branded sender address, update the `host`, `port`, `user`, `password`, and `from` fields in `/etc/msmtprc` with the provider's SMTP credentials.

---

| | | |
|:---|:---:|---:|
| [← Chapter 18 — Installing Nginx, MariaDB & PHP](./18-installing-nginx-mariadb-php.md) | [↑ Top](#chapter-19--server-mail-without-smtp-plugins) | [Chapter 20 — Nginx Configuration Files →](./20-nginx-configuration-files.md) |
