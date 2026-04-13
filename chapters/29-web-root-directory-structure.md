[← Back to Index](../index.md)

---

# Chapter 29 — Web Root File and Directory Structure

*Establishing the directory structure that will house each WordPress site, covering terminology, the `tree` utility, `mkdir -p`, `rmdir`, `rm -rf`, and `mv` — plus creating the actual site document root.*

---

## Contents

- [29.1 Directory Terminology](#291-directory-terminology)
- [29.2 Step 1 — Update the System and Install tree](#292-step-1--update-the-system-and-install-tree)
- [29.3 Step 2 — Create the First Site Directory Structure (Step by Step)](#293-step-2--create-the-first-site-directory-structure-step-by-step)
- [29.4 Step 3 — Create Directories in One Command with mkdir -p](#294-step-3--create-directories-in-one-command-with-mkdir--p)
- [29.5 Step 4 — Navigating Deep Directory Trees](#295-step-4--navigating-deep-directory-trees)
- [29.6 Step 5 — Removing Directories](#296-step-5--removing-directories)
  - [rmdir — Remove Empty Directories Only](#rmdir--remove-empty-directories-only)
  - [rm -rf — Force-Remove Directories and All Contents](#rm--rf--force-remove-directories-and-all-contents)
  - [Wildcard Removal](#wildcard-removal)
- [29.7 Step 6 — Renaming Directories with mv](#297-step-6--renaming-directories-with-mv)
- [29.8 Step 7 — Creating the Real Site Directory](#298-step-7--creating-the-real-site-directory)
- [29.9 Command Reference Summary](#299-command-reference-summary)

---

## 29.1 Directory Terminology

Understanding the difference between the web root and the document root is important because both terms appear in Nginx configuration and are frequently confused.

**Distribution web root (`/var/www/`)** — The top-level directory where all website files live on a Debian/Ubuntu system. This is the standard location used by both Nginx and Apache on Debian-based distributions. All sites hosted on this server will have their own subdirectory here.

**Document root (`/var/www/example.com/public_html/`)** — The specific directory within a site's folder that Nginx points to when serving requests. This is the directory that is exposed to the internet. Keeping it as a subdirectory of the domain folder (rather than the domain folder itself) is deliberate — it means files can be placed alongside `public_html/` (such as logs, SSL certificates, or configuration) that are completely outside the web-accessible path.

The full path structure for each site follows this pattern:

```
/var/www/
└── example.com/          ← site root (not web-accessible)
    └── public_html/      ← document root (web-accessible)
```

---

## 29.2 Step 1 — Update the System and Install `tree`

Before starting, run a system update to ensure all packages are current, then install the `tree` utility:

```bash
sudo apt update
sudo apt upgrade
sudo apt install tree
```

`tree` prints a visual directory hierarchy directly in the terminal, which makes it easy to confirm the structure is correct at every step without having to chain multiple `ls` commands.

---

## 29.3 Step 2 — Create the First Site Directory Structure (Step by Step)

Navigate to the web root and create the directory for the first site:

```bash
cd /var/www/
sudo mkdir example.com
ls
```

`ls` confirms `example.com` now sits alongside the default Nginx `html/` directory.

Enter the site directory and create the `public_html` subdirectory:

```bash
cd example.com/
sudo mkdir public_html/
ls
```

Verify the structure is correct:

```bash
cd public_html/
ls
cd ../../
tree
```

`tree` output confirms the structure:

```
├── example.com
│   └── public_html
└── html
    └── index.nginx-debian.html

4 directories, 1 file
```

---

## 29.4 Step 3 — Create Directories in One Command with `mkdir -p`

Creating the domain directory and `public_html` subdirectory in two separate steps works, but `mkdir -p` (the `p` flag stands for **parents**) creates the entire path in a single command, including any intermediate directories that do not yet exist:

```bash
sudo mkdir -p example.net/public_html/
tree
```

`tree` now shows both sites:

```
├── example.com
│   └── public_html
├── example.net
│   └── public_html
└── html
    └── index.nginx-debian.html

6 directories, 1 file
```

`mkdir -p` is the preferred method for creating site directories. It is idempotent — running it again when the directory already exists produces no error and makes no change, which is important when scripting server setup tasks.

---

## 29.5 Step 4 — Navigating Deep Directory Trees

When inside a deeply nested directory, `cd` with multiple `../` segments navigates back up multiple levels at once:

```bash
cd /var/www/a/b/c/d/e
cd ../../../../../
```

Each `../` moves up one level. Five `../` from depth five returns to `/var/www/`. This is used frequently when working inside site directory structures.

---

## 29.6 Step 5 — Removing Directories

### `rmdir` — Remove Empty Directories Only

`rmdir` removes a directory only if it is completely empty. Attempting to remove a directory that contains anything fails immediately with an error:

```bash
sudo rmdir a/
# rmdir: failed to remove 'a/': Directory not empty
```

`rmdir` is the safe choice when the intent is to remove what is known to be an empty directory — the error protects against accidentally deleting something with contents.

### `rm -rf` — Force-Remove Directories and All Contents

`rm -rf` removes a directory and everything inside it, recursively, without prompting:

```bash
sudo rm -rf a/
ls
```

The entire directory tree is gone entirely after a single command.

> **Warning:** `rm -rf` on a directory with content is irreversible. There is no recycle bin or undo. On a production server this command must be used with full awareness of what is being deleted. Always double-check the path before pressing Enter.

### Wildcard Removal

`rm -rf` also accepts wildcards:

```bash
sudo rm -rf example.*
ls
```

Wildcard patterns should be used with extra caution on a server — confirming what the pattern matches with `ls example.*` before running `rm -rf example.*` is a safe habit.

---

## 29.7 Step 6 — Renaming Directories with `mv`

`mv` (move) renames a directory by moving it to a new name within the same location:

```bash
sudo mv example.gro/ example.org/
ls
```

The directory that was created with a typo (`example.gro`) is renamed to the correct name (`example.org`) without needing to recreate it or move any of its contents. `mv` works the same way for both empty and non-empty directories.

---

## 29.8 Step 7 — Creating the Real Site Directory

With the practice commands complete and the `/var/www/` directory cleaned up, the actual site directory for this project is created:

```bash
sudo mkdir -p expertwp.help/public_html/
tree
```

```
├── expertwp.help
│   └── public_html
└── html
    └── index.nginx-debian.html

4 directories, 1 file
```

Navigate into `public_html/` to confirm it is empty and accessible, then return home:

```bash
cd expertwp.help/public_html/
ls
cd
```

The document root at `/var/www/expertwp.help/public_html/` is now ready to receive site files and to be referenced in the Nginx server block configuration in the next chapter.

---

## 29.9 Command Reference Summary

| Command | Purpose |
|---------|---------|
| `sudo apt install tree` | Install the tree directory visualiser |
| `tree` | Print a visual directory tree from the current location |
| `sudo mkdir domain.com` | Create a single directory |
| `sudo mkdir public_html/` | Create a subdirectory |
| `sudo mkdir -p domain.com/public_html/` | Create full path in one command (preferred) |
| `sudo rmdir directory/` | Remove an empty directory only |
| `sudo rm -rf directory/` | Force-remove a directory and all its contents |
| `sudo mv old_name/ new_name/` | Rename a directory |
| `cd ../../` | Navigate up two directory levels |

> **Automation note:** In a bash script, `mkdir -p /var/www/${DOMAIN}/public_html/` is the correct form for creating site directories. The `-p` flag ensures the script is safe to run on an already-provisioned server — existing directories are left untouched rather than causing the script to exit with an error.

---

| | | |
|:---|:---:|---:|
| [← Chapter 28 — Optimise PHP: OPcache & Open File Limit](./28-optimize-php-opcache-open-file-limit.md) | [↑ Top](#chapter-29--web-root-file-and-directory-structure) | [Chapter 30 — Nginx Server Blocks, Browser Caching & FastCGI →](./30-nginx-server-blocks-browser-caching-fastcgi.md) |
