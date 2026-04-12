# Contributing

Thank you for taking the time to contribute. This project is a practical, hands-on guide to building a production WordPress server. Contributions that improve accuracy, clarity, or coverage are welcome.

---

## What Counts as a Good Contribution

- **Corrections** — a command is wrong, a path has changed, a package name is different on a newer Ubuntu release
- **Clarifications** — a step is ambiguous or missing context that would help someone following along
- **Version updates** — software versions (PHP, MariaDB, Certbot, Fail2Ban, etc.) have been updated and the instructions need to reflect that
- **New sections** — a related topic that fits naturally within the project's scope and follows the same style
- **Typos and formatting** — small fixes are always welcome

## What Is Out of Scope

- Switching to a different stack (Apache, Caddy, PostgreSQL, etc.)
- Content that is vendor-specific and cannot be reproduced by anyone following the guide
- Sections that assume managed hosting — this guide targets self-managed VPS environments

---

## How to Contribute

### 1. Open an Issue First (for significant changes)

Before writing a new section or restructuring existing content, open an issue to describe what you want to change and why. This avoids wasted effort if the direction doesn't fit the project.

For small fixes (typos, broken commands, outdated package names), you can skip straight to a pull request.

### 2. Fork & Clone

```bash
git clone https://github.com/your-username/repo-name.git
cd repo-name
```

### 3. Create a Branch

Use a descriptive branch name:

```bash
git checkout -b fix/fail2ban-jail-config
git checkout -b section/redis-object-cache
git checkout -b update/php-84-compatibility
```

### 4. Make Your Changes

Follow the style guidelines below. Preview your Markdown locally before pushing.

### 5. Open a Pull Request

- Write a clear title and description explaining what changed and why
- Reference any related issue with `Closes #123` or `Related to #123`
- Keep each PR focused on a single topic — separate concerns into separate PRs

---

## Style Guidelines

### Markdown

- Use ATX-style headings (`#`, `##`, `###`)
- Use fenced code blocks with a language hint: ` ```bash `, ` ```nginx `, ` ```ini `, ` ```php `
- Use tables for structured comparisons (directive definitions, file paths, options)
- Avoid inline HTML

### Documentation Structure

Each section file follows this structure:

```
# Section Title

## Overview
Brief description of what the section covers and why it matters.

---

## Step 1 — Action Title
What the step does, why it is done.

```bash
command here
```

Expected output (if relevant):
```
output here
```

Explanation of what the output confirms.

---

## Step N — ...
```

- Section files are numbered (e.g. `35_php_fpm_pool_isolation.md`)
- Steps use `## Step N — Title` format
- Every command block is followed by an explanation of what it does or what the output means
- Configuration file contents are shown in full — no partial excerpts without clear context

### Commands

- Prefix privileged commands with `sudo`
- Use `nano` as the editor (consistent with the rest of the guide)
- Always show expected output for commands where output confirms success
- Avoid shell one-liners that sacrifice readability for brevity

### Paths & Names

- Use `/etc/nginx/sites-available/example.com.conf` as the placeholder domain — not `yourdomain.com` or `mydomain.com`
- Keep file paths consistent with the rest of the guide
- When referencing config keys, use `inline code` formatting

---

## Commit Messages

Use the imperative mood and be specific:

```
Fix incorrect socket path in PHP-FPM pool example
Add step to verify open file limits after MariaDB restart
Update Certbot install command for Ubuntu 24.04
```

Avoid vague messages like `fix stuff` or `update docs`.

---

## Questions

If you are unsure whether something is in scope or how to approach a change, open an issue and ask. Contributions are reviewed and responded to as time allows.
