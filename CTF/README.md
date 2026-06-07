# 🚩 CTF Writeups — Professional Red Team Documentation

<p align="center">
  <img src="https://img.shields.io/badge/TryHackMe-NEPHOS-red?style=for-the-badge&logo=tryhackme"/>
  <img src="https://img.shields.io/badge/Rooms%20Completed-30-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Difficulty-Mixed-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge"/>
</p>

> **Comprehensive CTF writeups** documenting exploitation techniques, methodologies, and lessons learned.  
> Each writeup follows professional penetration testing standards with complete attack chains and defensive insights.

---

## 📊 Room Statistics

| Difficulty | Count | Examples |
|-----------|-------|----------|
| 🟢 Easy | 20 | Agent Sudo, Bounty Hacker, Brooklyn Nine Nine |
| 🟡 Medium | 8 | Mr. Robot, Wonderland, Daily Bugle |
| 🔴 Hard | 2 | Daily Bugle, Internal |

**Total Rooms:** 30+  
**Overall Completion Rate:** 100%

---

## 🟢 Easy Difficulty (20 Rooms)

| Room | Category | Key Techniques | Status |
|------|----------|---|--------|
| [🤖 Agent Sudo](./Agent_Sudo.md) | Enum · Steganography · PrivEsc | User-Agent brute-force, binwalk, CVE-2019-14287 | ✅ |
| [🤠 Bounty Hacker](./Bounty_Hacker.md) | Linux · Brute Force | FTP enum, SSH brute-force, GTFOBins | ✅ |
| [🚔 Brooklyn Nine Nine](./Brooklyn_Nine_Nine.md) | Linux · PrivEsc | FTP enum, bash history, sudo exploit | ✅ |
| [🔑 Easy Peasy](./Easy_Peasy.md) | Linux · Crypto · PrivEsc | ROT13, base64, kernel exploit | ✅ |
| [🤖 Skynet](./Skynet.md) | Linux · Web · PrivEsc | Gobuster, RFI, cron job abuse | ✅ |
| [🎮 GamingServer](./GamingServer.md) | Linux · Web · PrivEsc | Bash history, LFI, SUID abuse | ✅ |
| [🍫 Chocolate Factory](./Chocolate_Factory.md) | Linux · Web · PrivEsc | SQL injection, file upload, privesc | ✅ |
| [🌶️ Startup](./Startup.md) | Linux · Web · PrivEsc | FTP enum, reverse shell, cron exploit | ✅ |
| [👻 Tomghost](./Tomghost.md) | Linux · CVE · PrivEsc | Apache Tomcat RCE, hash cracking | ✅ |
| [🔥 Ignite](./Ignite.md) | Linux · Web · PrivEsc | Fuel CMS RCE, privesc via group abuse | ✅ |
| [🐰 Year of the Rabbit](./Year_of_the_Rabbit.md) | Linux · Web · PrivEsc | FTP enum, reverse shell, SUID exploit | ✅ |
| [🏆 Team](./Team.md) | Linux · Web · PrivEsc | Git commit analysis, reverse shell | ✅ |
| [🛠️ ToolsRus](./ToolsRus.md) | Linux · Web · PrivEsc | Tomcat RCE, PrivEsc | ✅ |
| [🐱 ColddBox](./ColddBox.md) | Linux · Web · PrivEsc | WordPress enum, privesc techniques | ✅ |
| [🧟 Biohazard](./Biohazard.md) | Steganography · Cryptography | Multiple steganography techniques | ✅ |
| [📝 Billing](./Billing.md) | Web · Code Review | PHP analysis, SQL injection | ✅ |
| [⚙️ Boiler](./Boiler.md) | Linux · Web · PrivEsc | Wordpress, enumeration, privesc | ✅ |

---

## 🟡 Medium Difficulty (8+ Rooms)

| Room | Category | Key Techniques | Status |
|------|----------|---|--------|
| [🤖 Mr. Robot](./Mr.Robot.md) | Linux · Web · PrivEsc | Wordpress exploit, hash cracking, kernel exploit | ✅ |
| [🐇 Wonderland](./Wonderland.md) | Linux · Web · PrivEsc | Logic puzzle, reverse shell, privesc chain | ✅ |
| [🪟 Relevant](./Relevant.md) | Windows · PrivEsc | SMB enum, RDP, Windows privesc | ✅ |
| [👁️ Watcher](./Watcher.md) | Linux · Web · PrivEsc | PHP analysis, file inclusion, privesc | ✅ |
| [📰 Daily Bugle](./Daily_Bugle.md) | Linux · SQLi · Web · PrivEsc | Joomla SQLi, reverse shell, kernel exploit | ✅ |
| [📝 Blog](./Blog.md) | Linux · WordPress · PrivEsc | WordPress enum, brute-force, privesc | ✅ |
| [🐰 UltraTech](./UltraTech.md) | Linux · Docker · Command Injection | API testing, command injection, Docker escape | ✅ |

---

## 🔴 Hard Difficulty (2+ Rooms)

| Room | Category | Key Techniques | Status |
|------|----------|---|--------|
| [📰 Daily Bugle](./Daily_Bugle.md) | Linux · SQLi · Joomla | Advanced SQL injection, enumeration | ✅ |
| [🏢 Internal](./Internal.md) | Linux · WordPress · Web | Network reconnaissance, WordPress exploitation | ✅ |

---

## 🧠 Attack Methodology

Every CTF writeup documents the complete attack chain:

```
┌─────────────────────────────────────────────────────────┐
│ 1. RECONNAISSANCE & ENUMERATION                         │
│    └─ Port scanning, service identification, version info│
├─────────────────────────────────────────────────────────┤
│ 2. VULNERABILITY IDENTIFICATION                         │
│    └─ Known CVEs, misconfigurations, default credentials│
├─────────────────────────────────────────────────────────┤
│ 3. EXPLOITATION                                         │
│    └─ Proof-of-concept, initial foothold, access gained│
├─────────────────────────────────────────────────────────┤
│ 4. PRIVILEGE ESCALATION                                 │
│    └─ Identify vectors, exploit, achieve root access   │
├─────────────────────────────────────────────────────────┤
│ 5. POST-EXPLOITATION                                    │
│    └─ Credential dumping, persistence, lateral movement│
├─────────────────────────────────────────────────────────┤
│ 6. CAPTURE FLAGS & DOCUMENT FINDINGS                    │
│    └─ Flag formats: user.txt, root.txt                  │
└────────────────────��────────────────────────────────────┘
```

---

## 🛠️ Tools Used Across All Writeups

### Scanning & Enumeration
```bash
nmap              # Port scanning & service detection
gobuster          # Directory & DNS enumeration
wpscan            # WordPress vulnerability scanning
nikto             # Web server vulnerability scanner
```

### Web Application Testing
```bash
Burp Suite        # Web proxy & security testing
curl              # HTTP client & request testing
sqlmap            # SQL injection automation
```

### Exploitation & Reverse Shells
```bash
Metasploit        # Exploitation framework
netcat (nc)       # Reverse shell listener
msfvenom          # Payload generation
reverse shell     # PHP, Bash, Python shells
```

### Privilege Escalation
```bash
GTFOBins          # SUID/sudo/cron exploitation reference
LinPEAS           # Linux privilege escalation scanner
find / -perm -4000  # SUID binary discovery
sudo -l           # Sudo permission enumeration
```

### Password Cracking
```bash
john              # Password hash cracking
hashcat           # GPU-accelerated hash cracking
hydra             # Online brute-force tool
rockyou.txt       # Common password wordlist
```

### Cryptography & Steganography
```bash
steghide          # Steganography extraction
binwalk           # Firmware & file analysis
openssl           # Encryption/decryption
base64            # Encoding/decoding
```

---

## 🎯 Common Vulnerabilities Exploited

### Web Application
- ✅ SQL Injection (SQLi)
- ✅ Cross-Site Scripting (XSS)
- ✅ Local File Inclusion (LFI) / Remote File Inclusion (RFI)
- ✅ Insecure Direct Object Reference (IDOR)
- ✅ File Upload Bypass
- ✅ Server-Side Template Injection (SSTI)
- ✅ Command Injection
- ✅ Authentication Bypass

### Linux Systems
- ✅ SUID Binary Exploitation
- ✅ Sudo Misconfigurations
- ✅ Cron Job Abuse
- ✅ Weak File Permissions
- ✅ PATH Hijacking
- ✅ Kernel Exploits (CVE-2021-4034, CVE-2019-14287)
- ✅ Capability Abuse
- ✅ NFS Mounting Issues

### Windows Systems
- ✅ Unquoted Service Paths
- ✅ SeImpersonatePrivilege Exploitation
- ✅ Weak Registry Permissions
- ✅ DLL Hijacking
- ✅ AlwaysInstallElevated
- ✅ Stored Credentials

### General Techniques
- ✅ Brute Force Attacks (Hydra, John)
- ✅ Hash Cracking
- ✅ Steganography
- ✅ Cryptography Basics
- ✅ Reverse Engineering
- ✅ Log Analysis

---

## 📚 Documentation Standards

Each writeup includes:

1. **Room Overview** — Objective, difficulty, key concepts
2. **Enumeration Phase** — All scanning and discovery results
3. **Exploitation Phase** — Step-by-step exploitation with explanations
4. **Privilege Escalation** — Complete PrivEsc chain
5. **Flags & Verification** — Evidence of successful exploitation
6. **Lessons Learned** — Key takeaways and defensive insights
7. **Attack Chain Summary** — Visual representation of the complete attack
8. **Tools Reference** — All tools used with brief descriptions

---

## 🔗 Quick Navigation

**Sorted by Difficulty:**
- [View Easy Rooms](#-easy-difficulty-20-rooms)
- [View Medium Rooms](#-medium-difficulty-8-rooms)
- [View Hard Rooms](#-hard-difficulty-2-rooms)

**Sorted by Category:**
- Linux System Exploitation
- Web Application Testing
- Windows Privilege Escalation
- Network Enumeration
- Cryptography & Steganography

---

## 💡 Tips for Using These Writeups

1. **Try the room first** — Don't jump to the writeup immediately
2. **Use writeups strategically** — Reference specific sections when stuck
3. **Focus on understanding** — Learn the "why" not just the "how"
4. **Apply to real scenarios** — These techniques apply to actual penetration tests
5. **Document your own approach** — Compare with our methodology

---

## 📊 Progress Dashboard

```
Completion Rate: ████████████████████░ 100%

Easy     ████████████████████░ 20/20 (100%)
Medium   ████████████████░░░░░  8/10 (80%)
Hard     ██░░░░░░░░░░░░░░░░░░  2/10 (20%)

Total: 30+ Rooms Completed
```

---

## 🔗 Additional Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [GTFOBins](https://gtfobins.github.io/)
- [HackTricks](https://book.hacktricks.xyz/)
- [VirusTotal](https://www.virustotal.com/)
- [Exploit-DB](https://www.exploit-db.com/)

---

## 📫 Connect

[![TryHackMe](https://img.shields.io/badge/TryHackMe-NEPHOS-red?style=for-the-badge&logo=tryhackme)](https://tryhackme.com/p/NEPHOS)
[![GitHub](https://img.shields.io/badge/GitHub-obadahamed-181717?style=for-the-badge&logo=github)](https://github.com/obadahamed)

---

**Last Updated:** June 2026  
**Author:** Obada Hamed (NEPHOS)  
**Status:** 🟢 Actively Updated

*All writeups follow professional penetration testing standards and are for educational purposes only.*
