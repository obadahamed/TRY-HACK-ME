# 🧟 Biohazard — TryHackMe Write-Up

> — XENOS(Obada)

---

## 📋 Room Overview

| Field | Details |
|-------|---------|
| **Platform** | TryHackMe |
| **Room** | Biohazard |
| **Difficulty** | Medium |
| **Category** | CTF / Web + Steganography + Cryptography |
| **Author** | XENOS |

---

## 🗺️ Attack Chain Overview

```
Port Scan (Nmap)
      │
      ▼
Web Enumeration (Port 80)
      │
      ├──► Hidden Directories → Base64 / Base32 / ROT13 / Vigenère Ciphers
      │
      ├──► Flag Collection (Emblem, Lockpick, Gold, Shield, Blue Jewel, Crests x4)
      │
      ▼
FTP Login (Port 21) — Credentials from decoded crests
      │
      ├──► Steganography (Steghide / Strings / Binwalk) on JPG images
      │
      ├──► GPG Decryption → helmet_key
      │
      ▼
SSH Login (Port 22) — umbrella_guest / T_virus_rules
      │
      ├──► Hidden directory .jailcell → MO Disk 2 key
      │
      ├──► Vigenère (auto-solve) → weasker password
      │
      ▼
Privilege Escalation → sudo su → root
```

---

## 🔍 Phase 1 — Reconnaissance

### Nmap — Full Port Scan

```bash
nmap -p- --min-rate 5000 <TARGET_IP>
```

```bash
nmap -sV -sC -p 21,22,80 <TARGET_IP>
```

**Open Ports:**

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd |
| 22 | SSH | OpenSSH |
| 80 | HTTP | Apache |

> **Why this step matters:** A full port scan (`-p-`) ensures we don't miss non-standard ports. The version scan (`-sV`) reveals exact service versions — critical for identifying known CVEs. Scripts (`-sC`) run default NSE checks like anonymous FTP login, which here was **disabled**.

---

## 🌐 Phase 2 — Web Enumeration (Port 80)

### Methodology

Rather than running Gobuster blindly, this room rewards **manual source-code reading** — almost every directory is hidden inside HTML comments or encoded strings embedded in the page.

### Directory Discovery Chain

```
/            → source code → link to /mansionmain/
/mansionmain/ → source code comment → /diningRoom/
/diningRoom/  → Base64 string → /teaRoom/
/teaRoom/     → note → /artRoom/  (map of all rooms)
```

> **Theory — Why developers hide paths in comments:**
> Lazy developers often leave debug comments, staging paths, or backup file references in HTML source. A real-world attacker always reads the source before running automated scanners — automated tools miss inline hints.

---

## 🔐 Phase 3 — Cryptography Chain

This room is a sequence of encoding/cipher challenges. Each room unlocks the next.

---

### 3.1 — Base64

**Where:** `/diningRoom/` source code comment

```bash
echo "SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=" | base64 -d
# Output: How about the /teaRoom/
```

**Theory:** Base64 encodes binary data as ASCII text using a 64-character alphabet (`A-Z`, `a-z`, `0-9`, `+`, `/`). It is **encoding, not encryption** — no key needed. The `=` padding at the end is the giveaway.

---

### 3.2 — Base32

**Where:** Bar Room note

```bash
echo "KRQWK3TPNRQXIZLTOQQGK3TF..." | base32 -d
```

**Theory:** Base32 uses a 32-character alphabet and produces output with `=` padding. Commonly confused with Base64 — the key tell is that Base32 output is **all uppercase** and longer than Base64 for the same input.

---

### 3.3 — ROT13

**Where:** Dining Room 2F source code comment

```
Lbh trg gur oyhr trz ol chfuvat gur fgnghf gb gur ybjre sybbe...
```

```bash
echo "Lbh trg gur oyhr trz..." | tr 'A-Za-z' 'N-ZA-Mn-za-m'
# Output: You get the blue gem by pushing the status to the lower floor...
```

**Theory:** ROT13 is a Caesar cipher with a shift of 13. Because the English alphabet has 26 letters, applying ROT13 twice returns the original text. It is **symmetric** — the same operation decodes and encodes. Giveaway: letter frequency distribution is preserved, no numbers are shifted.

---

### 3.4 — Vigenère Cipher

**Where:** Bar Room → gold_emblem submission → encoded output

```
Cipher: xyhajlyy...
Key:    rebecca       (found via gold emblem → dining room interaction)
```

**Tool:** [CyberChef](https://gchq.github.io/CyberChef/) — Vigenère Decode operation

**Theory:** The Vigenère cipher is a polyalphabetic substitution cipher. Each character in the plaintext is shifted by the corresponding character in a repeating key. It was historically considered unbreakable but is now trivially solved with frequency analysis or a known key. The key here (`rebecca`) was embedded in the game's narrative — a reminder that in CTFs (and real engagements), **context is a clue**.

**Detection:** Unlike ROT13, Vigenère produces output where the frequency distribution of letters is flatter — this is the tell that the shift is not constant.

---

### 3.5 — Multi-Layer Decoding (Crests)

Each crest is encoded **multiple times** using different bases. The strategy:

> Decode iteratively — try Base64 first (look for `=` padding), take output as new input, try Base32/Base58 next.

| Crest | Encoding Layers | Decoded Segment |
|-------|----------------|----------------|
| 1 | Base64 → Base32 | `RlRQIHVzZXI6IG` |
| 2 | Base32 → Base58 | `h1bnRlciwgRlRQIHBh` |
| 3 | Base64 → Binary → Hex | `c3M6IHlvdV9jYW50X2h` |
| 4 | Base58 → Hex | `pZGVfZm9yZXZlcg==` |

**Combined + Base64 decode:**
```
FTP user: hunter
FTP pass: you_cant_hide_forever
```

---

## 📁 Phase 4 — FTP Enumeration (Port 21)

```bash
ftp <TARGET_IP>
# Login: hunter / you_cant_hide_forever

ftp> mget *
```

**Downloaded files:**
- `important.txt` — mentions helmet_key and `/hidden_closet/`
- `helmet_key.txt.gpg` — GPG-encrypted file
- `001-key.jpg`, `002-key.jpg`, `003-key.jpg`

---

## 🖼️ Phase 5 — Steganography

**Theory:** Steganography is the practice of hiding data inside another file (image, audio, video) such that its existence is not obvious. Unlike cryptography (which hides meaning), steganography hides the message's presence entirely. Common tools: **Steghide**, **Binwalk**, **Exiftool**, **Strings**.

---

### 001-key.jpg — Steghide

```bash
steghide extract -sf 001-key.jpg
# No passphrase needed
# Extracts: key-001.txt → cGxhbnQ0Ml9jYW
```

**Why this works:** Steghide hides data in the least-significant bits (LSB) of image pixels. The change in pixel value is imperceptible to the human eye but extractable with the tool.

---

### 002-key.jpg — Strings

```bash
strings 002-key.jpg | grep -v "^[a-zA-Z]"
# Found in EXIF comment: 5fYmVfZGVzdHJveV9
```

**Why this works:** EXIF metadata fields (especially `Comment`) are often overlooked. `strings` dumps all readable ASCII sequences from any binary — a fast first pass on any unknown file.

---

### 003-key.jpg — Binwalk

```bash
binwalk -e 003-key.jpg
# Extracts: key-003.txt → 3aXRoX3Zqb2x0
```

**Why this works:** Binwalk scans for embedded file signatures (magic bytes) within a file. Images can have entire files appended or nested inside them without affecting how the image renders.

---

### Combining the Keys

```
cGxhbnQ0Ml9jYW  +  5fYmVfZGVzdHJveV9  +  3aXRoX3Zqb2x0
```

```bash
echo "cGxhbnQ0Ml9jYW5fYmVfZGVzdHJveV93aXRoX3Zqb2x0" | base64 -d
# Output: plant42_can_be_destroy_with_vjolt
```

---

### GPG Decryption

```bash
gpg helmet_key.txt.gpg
# Passphrase: plant42_can_be_destroy_with_vjolt
```

> **Theory:** GPG (GNU Privacy Guard) implements the OpenPGP standard for symmetric and asymmetric encryption. A `.gpg` file encrypted symmetrically requires only a passphrase — no key pair exchange needed. The passphrase here was deliberately hidden across three steganographic layers, enforcing a complete enumeration workflow.

---

## 🔑 Phase 6 — SSH Login (Port 22)

**Credentials from study room + wolf medal:**
```bash
ssh umbrella_guest@<TARGET_IP>
# Password: T_virus_rules
```

### Enumeration

```bash
ls -la                   # Hidden directory .jailcell
cat .jailcell/chris.txt  # MO Disk 2 key: albert
ls /home                 # Users: hunter, weasker, umbrella_guest
```

---

## 🔓 Phase 7 — Vigenère (Auto-Solve) & Lateral Movement

**Encoded message from Hidden Closet:**
```
wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk
```

**Key source:** MO Disk 2 (`albert`) — found in `.jailcell/chris.txt`

**Tool:** [Boxentriq Vigenère Auto-Solver](https://www.boxentriq.com/code-breaking/vigenere-cipher)

**Decoded:**
```
albert weasker password: stars_members_are_my_guinea_pig
```

```bash
su weasker
# Password: stars_members_are_my_guinea_pig
```

---

## ⚡ Phase 8 — Privilege Escalation

```bash
sudo -l
# (ALL : ALL) ALL — weasker has unrestricted sudo

sudo su
# → root shell
```

**Theory — Misconfigured sudo:**
The `(ALL:ALL) ALL` sudo rule means the user can execute **any command as any user**, including root. This is the most dangerous sudo configuration. In a real environment, this would be flagged as a Critical finding — it completely nullifies any other access control on the system.

**Detection:**
- Always run `sudo -l` after gaining any shell
- Any entry without `NOPASSWD` still allows privilege escalation if you know the user's password

---

## 📊 Flags Summary

| Flag | Location | Method |
|------|---------|--------|
| Emblem flag | `/diningRoom/` | Direct interaction |
| Lockpick flag | `/teaRoom/` | Direct interaction |
| Gold emblem flag | Bar Room | Piano + emblem |
| Shield key | `/diningRoom/the_great_shield_key.html` | Vigenère + rebecca key |
| Blue jewel | `/diningRoom/sapphire.html` | ROT13 decode |
| Helmet key | `helmet_key.txt.gpg` | GPG + stego key |
| SSH creds | Study Room + Wolf Medal | Decompression |
| Root flag | `/root/` | sudo su |

---

## 🧠 Lessons Learned

1. **Read the source, always.** Automated scanners miss inline comments. Every hidden directory here was in the HTML — not in a wordlist.

2. **Encoding ≠ Encryption.** Base64/Base32/Base58 provide zero security. Recognizing them on sight (padding, character set, length) is a core skill.

3. **Steganography is a multi-tool discipline.** No single tool catches everything — Steghide, Strings, Binwalk, and Exiftool each revealed something different from the same set of images.

4. **Context is a key.** `rebecca` and `albert` weren't found by brute force — they were embedded in the room's narrative. In real-world engagements, information found in notes, emails, and comments is often the pivot point.

5. **sudo -l is always the first privesc check.** If a user has unrestricted sudo, the entire privilege model collapses in one command.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning & service detection |
| `CyberChef` | Multi-layer encoding/decoding |
| `Steghide` | LSB steganography extraction |
| `Binwalk` | Embedded file extraction |
| `Strings` | ASCII extraction from binaries |
| `Exiftool` | EXIF metadata inspection |
| `gpg` | Symmetric decryption |
| Boxentriq | Vigenère auto-solve |

---

<div align="center">

*Written by* **XENOS** | [GitHub](https://github.com/obadahamed) · [Portfolio](https://obadahamed.github.io)

</div>
