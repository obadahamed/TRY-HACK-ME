# 🔺 Privilege Escalation — Notes & Techniques

![Category](https://img.shields.io/badge/Category-Privilege%20Escalation-red?style=flat-square)
![Focus](https://img.shields.io/badge/Focus-Linux%20%7C%20Windows-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

> Comprehensive notes and techniques for Linux and Windows privilege escalation.  
> Built from TryHackMe labs, CTF experience, and the Jr Penetration Tester path.

---

## 📁 Contents

| File | Description |
|------|-------------|
| [🐧 Linux Privilege Escalation](./Linux%20Privilege%20Escalation.md) | Full Linux privesc techniques with commands |
| [🐧 Common-Linux-PrivEsc](./Common-Linux-PrivEsc.md) | Common-Linux-PrivEsc ways |

---

## 🎯 Key Concepts Covered

### 🐧 Linux
- SUID / SGID binaries
- Sudo misconfigurations (`sudo -l`)
- Cron job abuse
- Weak file permissions
- PATH hijacking
- Capabilities
- `/etc/passwd` writable

### 🪟 Windows *(coming soon)*
- SeImpersonatePrivilege (Potato attacks)
- Unquoted service paths
- Weak registry permissions
- AlwaysInstallElevated
- DLL hijacking
- Stored credentials

---

## ⚡ Quick Reference — Linux

```bash
# 1. Basic enumeration
whoami && id
sudo -l
cat /etc/passwd | grep -v nologin

# 2. SUID binaries
find / -perm -4000 2>/dev/null

# 3. Writable directories
find / -writable 2>/dev/null | grep -v proc

# 4. Cron jobs
cat /etc/crontab
ls -la /etc/cron*

# 5. Running processes
ps aux | grep root
```

---

## 🔗 Useful Resources

- [GTFOBins](https://gtfobins.github.io/) — SUID/sudo/cron exploit one-liners
- [HackTricks — Linux Privesc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [TryHackMe — Linux Privesc](https://tryhackme.com/room/linprivesc)

---

## 📫 Connect

[![GitHub](https://img.shields.io/badge/GitHub-obadahamed-181717?style=flat-square&logo=github)](https://github.com/obadahamed)
[![TryHackMe](https://img.shields.io/badge/TryHackMe-NEPHOS-red?style=flat-square&logo=tryhackme)](https://tryhackme.com/p/NEPHOS)
