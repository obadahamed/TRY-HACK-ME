# File Upload Vulnerabilities — From Upload to RCE
### Web Exploitation Write-Up | Week 3
**Author:** XENOS  
**Platform:** TryHackMe  
**Date:** 2026-03-19

---

## What is a File Upload Vulnerability?

File upload functionality is one of the most common features in modern web applications — profile pictures, attachments, documents. But when an application fails to properly validate what is being uploaded, that same functionality becomes a direct path to Remote Code Execution (RCE).

The core problem isn't that uploads exist. The problem is trust: applications that rely on client-side validation, extension blacklists, or MIME type checks without server-side enforcement are making assumptions an attacker will never respect.

This write-up documents a full bypass chain — six layers of increasing restriction, each broken with a different technique.

---

## The Bypass Chain

Each task in this lab introduced a new layer of defense. Each one fell.

---

### Task 1 — Direct File Overwrite

**The barrier:** None. No validation whatsoever.

**The approach:** Uploaded a file with the same name as an existing server-side file. The server accepted it and overwrote the original without complaint.

**Why it works:** When there is no access control on the upload destination and no uniqueness enforcement, an attacker controls what lives on the server's filesystem.

**Lesson:** Upload functionality without any restriction is not a feature — it's an open door.

---

### Task 2 — Direct PHP Shell Upload

**The barrier:** None on file type. The server executed uploaded scripts directly.

**The approach:** Uploaded a basic PHP webshell:

```php
<?php echo system($_GET['cmd']); ?>
```

Navigated to the uploaded file's URL and passed system commands via the `cmd` parameter:

```
http://target/uploads/shell.php?cmd=whoami
http://target/uploads/shell.php?cmd=cat /etc/passwd
```

**Why it works:** If the server runs PHP and there is no restriction on uploading `.php` files, any uploaded script becomes executable server-side code.

**Lesson:** File execution permissions on the upload directory are as critical as the upload validation itself.

---

### Task 3 — Client-Side JavaScript Filter Bypass

**The barrier:** A JavaScript filter running in the browser rejecting non-image extensions before the request was sent.

**The approach:** Client-side validation only exists in the browser — it never reaches the server. Two ways to bypass it:

1. Disable JavaScript in the browser entirely
2. Intercept the request in **Burp Suite** after bypassing the frontend check, then modify the filename from `shell.jpg` to `shell.php` before forwarding

The server received a `.php` file and executed it without question.

**Why it works:** The server had no independent validation. It trusted that the client had already filtered the input — a fundamentally broken assumption.

**Lesson:** Client-side validation is a UX feature, not a security control. Any validation that only exists in the browser can be bypassed in seconds.

---

### Task 4 — Server-Side Blacklist Bypass via `.phar`

**The barrier:** A server-side blacklist blocking `.php` file extensions.

**The approach:** PHP supports multiple valid extensions beyond `.php`. The blacklist only accounted for the obvious one. Uploading the same webshell with a `.phar` extension bypassed the filter entirely — and the server still executed it as PHP.

```
shell.phar
```

**Why it works:** Blacklists are inherently incomplete. PHP alone supports `.php`, `.php3`, `.php4`, `.php5`, `.phtml`, `.phar`. A blacklist must anticipate every possible bypass — a whitelist only needs to define what is allowed.

**Lesson:** Blacklists lose. Whitelists win. If the application doesn't explicitly define what is permitted and reject everything else, the filter will eventually be bypassed.

---

### Task 5 — Magic Bytes Bypass

**The barrier:** Server-side MIME type validation reading the file's magic bytes — the first few bytes of a file that identify its format — rather than trusting the extension or Content-Type header.

**The approach:** Prepended the GIF magic bytes signature to the PHP webshell:

```
GIF89a
<?php echo system($_GET['cmd']); ?>
```

The server read `GIF89a` at the start of the file, identified it as a GIF image, and accepted the upload. The embedded PHP code executed when the file was accessed.

**Why it works:** Magic byte checks verify file format, not file safety. A file can be a valid GIF and also contain executable PHP. These two facts are not mutually exclusive.

**Lesson:** No single validation method is sufficient. A robust implementation validates extension, MIME type, magic bytes, and — most importantly — ensures the upload directory cannot execute scripts.

---

### Task 6 — Node.js Reverse Shell

**The barrier:** The server ran Node.js instead of PHP, requiring a language-appropriate payload.

**The approach:** Adapted the attack to the target's runtime. Crafted a Node.js reverse shell payload, set up a listener with netcat:

```bash
nc -lvnp 4444
```

Uploaded the payload, triggered execution, and received an interactive shell on the listener.

**Why it works:** The vulnerability class is the same regardless of the server-side language. The payload must simply match the runtime. Identifying the technology stack before crafting a payload is a fundamental recon step.

**Lesson:** File upload vulnerabilities are not PHP-specific. Any server-side language that can be uploaded and executed is a viable attack vector.

---

## Tooling

| Tool | Purpose |
|---|---|
| **Burp Suite Repeater** | Intercepting and modifying requests to bypass client-side filters |
| **Burp Suite Proxy** | Capturing upload requests before they reach the server |
| **netcat** | Listening for incoming reverse shell connections |
| **curl** | Triggering uploaded files and testing execution |

---

## The Attack Pattern — Summarized

```
Identify upload functionality
        ↓
Attempt direct malicious upload
        ↓
Identify the filter type (client-side / extension / MIME / magic bytes)
        ↓
Select the appropriate bypass technique
        ↓
Upload → Navigate to file → Execute → Shell
```

---

## Defensive Recommendations

For completeness — what a properly secured file upload implementation looks like:

- **Whitelist** allowed extensions, never blacklist
- Validate MIME type **server-side**, independent of the Content-Type header
- Rename uploaded files server-side — never preserve the original filename
- Store uploaded files **outside the web root** so they cannot be accessed via URL
- Ensure the upload directory has **no execution permissions**
- Scan file contents with a dedicated library, not just extension or header checks

---

## Key Takeaway

File upload vulnerabilities are a direct path from a standard user interaction to full server compromise. Every layer of validation in this lab was bypassable — not because the techniques are complex, but because each filter was applied in isolation, trusting that one check was enough.

Defense requires layering. Attack requires finding the one check that was skipped.

The bypass chain in this lab is a good mental model to internalize: when one technique fails, understand *why* it failed, identify *what* the server is actually checking, and adapt accordingly.

---

*Written by XENOS — obadahamed.github.io*
