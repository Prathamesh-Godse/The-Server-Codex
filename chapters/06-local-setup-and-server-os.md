[← Back to Index](../index.md)

---

# Chapter 6 — Local Machine Setup & Server OS Choice

*What you need on your local machine before touching the server, and why Ubuntu 24.04 LTS is the right choice for the server OS.*

---

## Contents

- [6.1 Local Machine vs. Remote Server](#61-local-machine-vs-remote-server)
- [6.2 Why Ubuntu 24.04 LTS as the Server OS](#62-why-ubuntu-2404-lts-as-the-server-os)
- [6.3 Local Software Requirements](#63-local-software-requirements)
  - [Web Browser](#web-browser)
  - [Text Editor](#text-editor)
  - [Terminal Emulator](#terminal-emulator)
- [6.4 Summary](#64-summary)

---

## 6.1 Local Machine vs. Remote Server

The local operating system is irrelevant to the server setup. All interaction with the server happens remotely over SSH from a terminal. The server itself always runs Linux regardless of what the local machine runs.

| | Local Machine | Remote Server |
|---|---|---|
| **OS** | Windows, macOS, or Linux | Ubuntu 24.04 LTS |
| **Role** | Workstation — used to connect to and manage the server | Hosts Nginx, MariaDB, PHP, WordPress |
| **Access method** | Terminal / Git Bash → SSH | Receives SSH connections |

---

## 6.2 Why Ubuntu 24.04 LTS as the Server OS

Ubuntu 24.04 LTS (Noble Numbat) was chosen as the server distribution for this project for the following reasons:

- **Market dominance** — powers 40%+ of all Linux servers in production
- **Large community** — extensive documentation, Stack Overflow answers, and forum support readily available
- **Fast, secure, and stable** — well-tested LTS release, hardened defaults
- **Up-to-date packages** — ships with newer versions of Nginx, PHP, and MariaDB compared to older LTS releases
- **Long-term support** — up to 12 years of security updates (5 years standard + extended via Ubuntu Pro), making it suitable for production deployments without frequent OS migrations

> LTS (Long-Term Support) releases are the correct choice for servers. Non-LTS Ubuntu releases only receive 9 months of support and should not be used for production infrastructure.

---

## 6.3 Local Software Requirements

Three pieces of software are needed on the local machine to work with the server.

### Web Browser

Any modern browser works — Chrome, Firefox, Edge, or Safari. The browser is used to access the WordPress admin panel, verify the site is live, and read documentation.

### Text Editor

A local text editor is useful for drafting configuration file content and scripts before copying them to the server. Any editor works — VS Code, Sublime Text, Notepad++, or similar.

### Terminal Emulator

The terminal is the primary interface for all server interaction. SSH connections, running commands, and editing files on the server are all done through the terminal.

On **Linux or macOS**, the built-in terminal application is sufficient.

On **Windows**, use one of the following:

- **Git Bash** — a Unix-like terminal bundled with Git for Windows; provides `ssh`, `scp`, and all standard Unix commands
- **Windows Terminal** with WSL (Windows Subsystem for Linux)
- **PowerShell** — works for SSH on modern Windows versions

> For consistency across platforms, all SSH commands in this project are written in standard Unix syntax, which works identically in Git Bash, macOS Terminal, and Linux Terminal.

---

## 6.4 Summary

| Requirement | Detail |
|---|---|
| Server OS | Ubuntu 24.04 LTS |
| Local OS | Any — Windows, macOS, or Linux |
| Browser | Any modern browser |
| Text editor | Any (VS Code recommended) |
| Terminal | Linux/macOS Terminal, or Git Bash on Windows |

---

| | | |
|:---|:---:|---:|
| [← Chapter 5 — Essential Linux Skills](./05-essential-linux-skills.md) | [↑ Top](#chapter-6--local-machine-setup--server-os-choice) | [Chapter 7 — Server Provisioning →](./07-server-provisioning.md) |
