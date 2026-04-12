[← Back to Index](../index.md)

---

# Chapter 3 — Linux File System

*The Linux file system hierarchy, naming conventions, and navigation commands used throughout the server setup. Understanding the directory structure is essential — all configuration files, web roots, and log files live in specific, predictable locations.*

---

## Contents

- [3.1 File and Directory Naming Rules](#31-file-and-directory-naming-rules)
- [3.2 Directory Layout](#32-directory-layout)
- [3.3 Key Directories](#33-key-directories)
- [3.4 Navigating the File System](#34-navigating-the-file-system)
  - [Print Working Directory](#print-working-directory)
  - [Change Directory](#change-directory)
- [3.5 Absolute vs Relative Paths](#35-absolute-vs-relative-paths)

---

## 3.1 File and Directory Naming Rules

| Rule | Detail |
|------|--------|
| **Case sensitive** | `nginx.conf`, `Nginx.conf`, and `NGINX.CONF` are three different files |
| **Max length** | 255 characters per name |
| **Avoid special characters** | `* ? / : \| " ' < > [ ]` — these have special meaning in the shell |
| **No spaces** | Use underscores instead — `sshd_config.d` not `sshd config.d` |
| **Hidden files** | Any file or directory starting with `.` is hidden from `ls` by default |
| **Extensions** | Not required on Linux, but `.conf` is the convention for configuration files |

---

## 3.2 Directory Layout

The Linux file system is a single tree rooted at `/`. Everything — disks, devices, config files, web roots — lives somewhere under it. The directories most relevant to this project are:

```
/
├── etc/                        # System-wide configuration files
│   ├── nginx/
│   │   ├── nginx.conf          # Main Nginx configuration
│   │   ├── sites-available/    # Virtual host config files (inactive)
│   │   └── sites-enabled/      # Symlinks to active virtual hosts
│   ├── php/
│   │   └── 8.3/
│   │       └── fpm/            # PHP-FPM configuration
│   └── mysql/                  # MariaDB configuration
├── var/                        # Variable data (grows at runtime)
│   ├── log/                    # System and application log files
│   └── www/                    # Web root — WordPress sites live here
├── home/                       # Home directories for non-root users (~)
│   └── username/
└── root/                       # Home directory for the root user
```

---

## 3.3 Key Directories

| Directory | Purpose |
|-----------|---------|
| `/etc` | All system configuration files. Every service installed (Nginx, PHP, MariaDB, SSH) has a subdirectory here. This is where almost all editing happens. |
| `/etc/nginx/sites-available/` | Nginx virtual host config files are created here. A file here does **not** activate a site on its own. |
| `/etc/nginx/sites-enabled/` | Contains symlinks pointing to files in `sites-available`. Nginx only reads enabled sites. |
| `/etc/php/8.3/fpm/` | PHP-FPM pool and `php.ini` configuration for PHP 8.3. |
| `/var/www/` | Web root directory. Each WordPress site gets its own subdirectory here, e.g. `/var/www/example.com/public_html/`. |
| `/var/log/` | Log files for all services. Useful for debugging — `nginx/error.log`, `php8.3-fpm.log`, `mysql/error.log` all live here. |
| `/home/username/` | Home directory of the non-root server user. Represented as `~` in the prompt. |

---

## 3.4 Navigating the File System

### Print Working Directory

`pwd` prints the full absolute path of your current location. Useful for confirming where you are before editing files.

```bash
pwd
# Output: /home/ubuntu
```

### Change Directory

```bash
# Go to home directory (no matter where you are)
cd

# Go to the filesystem root
cd /

# Go to a specific directory using absolute path
cd /etc/nginx

# Go to a subdirectory relative to current location
cd sites-available

# Go up one directory level
cd ..

# Go up two directory levels
cd ../../

# Go to the previous directory (toggle between two locations)
cd -
```

**Terminal demo — navigating through the file system:**

```bash
andrew@hp:~$ pwd
/home/andrew

andrew@hp:~$ cd /etc
andrew@hp:/etc$ pwd
/etc

andrew@hp:/etc$ cd /var/log
andrew@hp:/var/log$ pwd
/var/log

andrew@hp:/var/log$ cd
andrew@hp:~$ pwd
/home/andrew

# Relative path — only works if you're already in /var
andrew@hp:~$ cd /var
andrew@hp:/var$ cd log
andrew@hp:/var/log$ pwd
/var/log

# Toggle between last two directories
andrew@hp:/var/log$ cd -
/home/andrew
andrew@hp:~$ cd -
/var/log

# Go up one level
andrew@hp:/var/log$ cd ..
andrew@hp:/var$

# Go up two levels from a nested directory
andrew@hp:/var/log/installer$ cd ../../
andrew@hp:/var$
```

> **Note:** `cd var` (relative) only works if a directory named `var` exists in your current location. `cd /var` (absolute) works from anywhere. Always prefer absolute paths when writing scripts to avoid location-dependent failures.

---

## 3.5 Absolute vs Relative Paths

There are two ways to specify the location of any file or directory on the system.

### Absolute Path

Starts with `/` and specifies the full path from the root directory. Works from any location in the file system.

```bash
# These are absolute paths — valid from anywhere
cd /etc/nginx
cd /var/www/example.com/public_html
nano /etc/nginx/sites-available/example.com.conf
nano /etc/php/8.3/fpm/php.ini
```

### Relative Path

Does **not** start with `/`. Specifies the location relative to the current working directory. The path changes meaning depending on where you are.

```bash
# If currently in /etc — these are equivalent:
cd nginx            # relative: moves to /etc/nginx
cd /etc/nginx       # absolute: same result, works from anywhere

# If currently in /etc/nginx — to reach /etc/php/8.3/fpm/php.ini:
nano ../../php/8.3/fpm/php.ini    # relative path using ..
nano /etc/php/8.3/fpm/php.ini     # absolute path — clearer and safer
```

**Project-relevant path reference:**

| Absolute Path | What it points to |
|---------------|-------------------|
| `/etc/nginx/nginx.conf` | Main Nginx config |
| `/etc/php/8.3/fpm/php.ini` | PHP runtime settings |
| `/etc/ssh/sshd_config.d/` | SSH daemon drop-in config directory |
| `/var/www/example.com/public_html/` | WordPress site web root |
| `/var/log/nginx/error.log` | Nginx error log |

Throughout this project, **absolute paths are used exclusively** in all commands and configuration files. Relative paths are fine interactively but are unreliable in scripts and configurations where the working directory cannot be guaranteed.

---

| | | |
|:---|:---:|---:|
| [← Chapter 2 — Linux Essentials](./02-linux-essentials.md) | [↑ Top](#chapter-3--linux-file-system) | [Chapter 4 — Users, Groups & Permissions →](./04-users-groups-permissions.md) |
