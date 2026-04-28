# 🚩 TryHackMe — CTF Write-Ups

<p align="center">
  <img src="https://img.shields.io/badge/TryHackMe-OBADA-red?style=for-the-badge&logo=tryhackme"/>
  <img src="https://img.shields.io/badge/Rooms%20Completed-19-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Hard-1-critical?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Medium-5-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Easy-13-success?style=for-the-badge"/>
</p>

> Structured write-ups for TryHackMe CTF rooms — documenting methodology, tools, techniques, and lessons learned.  
> Written from the perspective of a self-taught penetration tester following a **52-week Red Team roadmap**.

---

## 📋 Completed Rooms

| Room | Difficulty | Category | Status |
|------|-----------|----------|--------|
| [🤖 Mr. Robot](./Mr.Robot.md) | 🟡 Medium | Linux · Web · PrivEsc | ✅ |
| [👾 Agent Sudo](./Agent_Sudo.md) | 🟢 Easy | Steganography · PrivEsc | ✅ |
| [🤠 Bounty Hacker](./Bounty_Hacker.md) | 🟢 Easy | Linux · Brute Force | ✅ |
| [🚔 Brooklyn Nine Nine](./Brooklyn_Nine_Nine.md) | 🟢 Easy | Linux · PrivEsc | ✅ |
| [🔑 Easy Peasy](./Easy_Peasy.md) | 🟢 Easy | Linux · PrivEsc | ✅ |
| [🤖 Skynet](./Skynet.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [🎮 GamingServer](./GamingServer.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [🍫 Chocolate Factory](./Chocolate_Factory.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [🐇 Wonderland](./Wonderland.md) | 🟡 Medium | Linux · Web · PrivEsc | ✅ |
| [🌶️ Startup](./Startup.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [👻 Tomghost](./Tomghost.md) | 🟢 Easy | Linux · CVE · PrivEsc | ✅ |
| [🔥 Ignite](./Ignite.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [🪟 Relevant](./Relevant.md) | 🟡 Medium | Windows · PrivEsc | ✅ |
| [📰 Daily Bugle](./Daily_Bugle.md) | 🔴 Hard | Linux · SQLi · Web · PrivEsc | ✅ |
| [🐰 Year of the Rabbit](./Year_of_the_Rabbit.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [👁️ Watcher](./Watcher.md) | 🟡 Medium | Linux · Web · PrivEsc | ✅ |
| [🏆 Team](./Team.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [🛠️ ToolsRus](./ToolsRus.md) | 🟢 Easy | Linux · Web · PrivEsc | ✅ |
| [📝 Blog](./Blog.md) | 🟡 Medium | Linux · WordPress · PrivEsc | ✅ |
| [Internal](./Internal.md)| 🔴 Hard | Linux  · Web · PrivEsc . tunuulling . WordPress| ✅ |
| (UltraTech)(./UltraTech.md) | 🟡 Medium | Linux · Docker · PrivEsc . Command Injection  | ✅ |

---

## 🧠 Methodology

Every room follows the same structured attack chain:

```
1. Recon       →  nmap scan, service & version enumeration
2. Enumerate   →  dig deeper into the attack surface
3. Exploit     →  gain initial foothold
4. Escalate    →  identify and abuse PrivEsc vector
5. Document    →  write up the full chain with lessons learned
```

---

## 🛠️ Tools Used

| Category | Tools |
|----------|-------|
| Scanning | `nmap`, `gobuster`, `wpscan` |
| Exploitation | `Metasploit`, `Burp Suite`, custom scripts |
| Cracking | `john`, `hashcat`, `gpg2john`, `rockyou.txt` |
| Steganography | `steghide`, `binwalk`, `strings` |
| Shells | `netcat`, PHP reverse shell, `pty` spawn |
| PrivEsc | `GTFOBins`, `sudo -l`, `getcap`, `find -perm -u=s` |

---

## 📊 Progress

```
Easy    █████████████  13 rooms
Medium  █████░░░░░░░░   5 rooms
Hard    █░░░░░░░░░░░░   1 room
─────────────────────
Total                  19 rooms
```

---

*Write-ups authored by [XENOS](https://obadahamed.github.io) · 52-Week Red Team Roadmap*
