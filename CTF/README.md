# 🚩 TryHackMe — CTF Write-Ups

![Profile](https://img.shields.io/badge/TryHackMe-OBADA-red?style=flat-square&logo=tryhackme)
![Rooms](https://img.shields.io/badge/Rooms%20Completed-14-blue?style=flat-square)
![Difficulty](https://img.shields.io/badge/Hard-1-critical?style=flat-square)
![Difficulty](https://img.shields.io/badge/Medium-3-orange?style=flat-square)
![Difficulty](https://img.shields.io/badge/Easy-10-success?style=flat-square)

> Structured write-ups for TryHackMe CTF rooms — documenting methodology, tools, techniques, and lessons learned.  
> Written from the perspective of a self-taught penetration tester following a 52-week Red Team roadmap.

---

## 📋 Completed Rooms

| Room | Difficulty | Category | Status |
|------|-----------|----------|--------|
| [🤖 Mr. Robot](./Mr.Robot.md) | 🟡 Medium | Linux / Web / PrivEsc | ✅ Solved |
| [👾 Agent Sudo](./Agent_Sudo.md) | 🟢 Easy | Steganography / PrivEsc | ✅ Solved |
| [🤠 Bounty Hacker](./Bounty_Hacker.md) | 🟢 Easy | Linux / Brute Force | ✅ Solved |
| [🚔 Brooklyn Nine Nine](./Brooklyn_Nine_Nine.md) | 🟢 Easy | Linux / PrivEsc | ✅ Solved |
| [🔑 Easy Peasy](./Easy_Peasy.md) | 🟢 Easy | Linux / PrivEsc | ✅ Solved |
| [🤖 Skynet](./Skynet.md) | 🟢 Easy | Linux / Web / PrivEsc | ✅ Solved |
| [🎮 GamingServer](./GamingServer.md) | 🟢 Easy | Linux / Web / PrivEsc | ✅ Solved |
| [🍫 Chocolate Factory](./Chocolate_Factory.md) | 🟢 Easy | Linux / Web / PrivEsc | ✅ Solved |
| [🐇 Wonderland](./Wonderland.md) | 🟡 Medium | Linux / Web / PrivEsc | ✅ Solved |
| [🌶️ Startup](./Startup.md) | 🟢 Easy | Linux / Web / PrivEsc | ✅ Solved |
| [👻 Tomghost](./Tomghost.md) | 🟢 Easy | Linux / CVE / PrivEsc | ✅ Solved |
| [🔥 Ignite](./Ignite.md) | 🟢 Easy | Linux / Web / PrivEsc | ✅ Solved |
| [🪟 Relevant](./Relevant.md) | 🟡 Medium | Windows / PrivEsc | ✅ Solved |
| [📰 Daily Bugle](./Daily_Bugle.md) | 🔴 Hard | Linux / SQLi / Web / PrivEsc | ✅ Solved |
| [🐇 Year of the Rabbit](./Year_of_the_Rabbit.md) | 🟢 Easy | Linux / Web / PrivEsc | ✅ Solved |
| [👾 Watcher](./Watcher.md) | 🟡 Medium | linux / PrivEsc / web | ✅ Solved |



---

## 🧠 Methodology

Every room follows the same structured process:

```
1. Recon       →  nmap scan, service enumeration
2. Enumerate   →  dig deeper into attack surface
3. Exploit     →  gain initial foothold
4. Escalate    →  identify and exploit PrivEsc vector
5. Document    →  write up the full chain
```

---

## 🛠️ Tools Used

| Category | Tools |
|----------|-------|
| Scanning | `nmap`, `gobuster` |
| Exploitation | `Metasploit`, `Burp Suite`, custom scripts |
| Cracking | `john`, `hashcat`, `gpg2john` |
| Steganography | `steghide`, `binwalk`, `strings` |
| Shells | `netcat`, PHP reverse shell |
| PrivEsc | GTFOBins, `sudo -l`, `getcap`, `find` |

---

## 📊 Progress

```
Easy    ███████████ 11/11
Medium  ████░░░░░░  4/4
Hard    █░░░░░░░░░  1/1
```

---

*Writeups authored by [XENOS](https://obadahamed.github.io) | 52-Week Red Team Roadmap*
