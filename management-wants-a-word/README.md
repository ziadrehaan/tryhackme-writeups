Hacker Holidays 2026 · Day 14
Category: Forensics · Windows · Cryptography
Difficulty: Hard
Points: 120
Flag: THM{..._..._..._...} (redacted)
"It was always her. It was never a bug; it was the business model."
Overview
This room is a Windows forensics chain, not a single exploit. You are given a KAPE triage dump of a laptop left behind by "Vera" in Room 214. The objective is to follow the artifact trail from browser secrets through DPAPI to a VeraCrypt container, and finally recover the flag from a hidden document.
The Chain:
plain
KAPE Triage → Chrome Artifacts → Saved Password → VeraCrypt Container → PDF Flag
Step 1 — Understanding the Hint
@0xMia's post gives two critical clues:
"a browser will remember things for you that you never told anyone else 💀"
"why did Patch tell me this version number 1.26.29 idk what it means"
Analysis:
Browser memory → Chrome stores saved passwords in Login Data.
1.26.29 → This is a VeraCrypt release version, not a Chrome version.
"not every hidden file needs a password cracker" → The password is already stored somewhere in the triage; you just need to find it.
Step 2 — Triage & Initial Recon
After extracting the archive, the structure is a standard KAPE output:
plain
management-wants-a-word-forensics-hh-day-14/
├── KAPE/
│   └── C/
│       ├── Users/
│       │   └── vera/
│       │       ├── AppData/
│       │       │   ├── Local/
│       │       │   │   └── Google/
│       │       │   │       └── Chrome For Testing/
│       │       │   │           └── User Data/
│       │       │   │               └── Default/
│       │       │   │                   ├── Login Data
│       │       │   │                   ├── History
│       │       │   │                   ├── Cookies
│       │       │   │                   └── ...
│       │       │   └── Roaming/
│       │       │       └── Microsoft/
│       │       │           └── Protect/          ← DPAPI master keys
│       │       └── Documents/
│       │           └── backup                      ← 100 MB container
│       └── Windows/
│           └── System32/
│               └── config/
│                   ├── SAM
│                   ├── SYSTEM
│                   └── ...
The presence of SAM, SYSTEM, and Vera's full profile tells us we can do offline credential recovery.
Step 3 — Chrome History Analysis
Open History in DB Browser for SQLite and browse the urls table:
sql
SELECT id, url, title FROM urls;
Screenshot — History Table:
Key findings:
http://bytelotus.thm:8080/ — SecureVault Portal
http://bytelotus.thm:8080/login — Login page
Google searches for "chrome cves", "how to exfiltrate data red teaming", "tryhackme"
Vera had a local vault application running on bytelotus.thm:8080.
Step 4 — Extracting the Saved Password
Open Login Data in DB Browser and browse the logins table:
Screenshot — Logins Table:
Table
origin_url	username_value	password_value
http://bytelotus.thm:8080/	VeraSecretVault	BLOB (encrypted)
The password_value is encrypted with Windows DPAPI and then AES-GCM via Chrome. To decrypt it you have two options.
Option A — ChromePass (Easiest)
Download ChromePass from NirSoft.
Launch it and go to File → Advanced Options → Select User Data Folder.
Point it at the User Data folder (not Default, the parent directory).
ChromePass will use the embedded DPAPI keys to decrypt the password automatically.
Result:
plain
URL:      http://bytelotus.thm:8080/
Username: VeraSecretVault
Password: Wh4t1sV3raD0inG0nTh1sH0st
Option B — Manual DPAPI + Python (Advanced)
If you prefer the command-line route:
bash
# 1. Extract NT hash from SAM/SYSTEM
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# 2. Crack Vera's password with John/Hashcat
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt nt.john
# → minivera

# 3. Decrypt the DPAPI master key
impacket-dpapi masterkey -file ".../Protect/SID/c90719ef-..." \
  -sid "S-1-5-21-..." \
  -password minivera

# 4. Use a Python script to decrypt Chrome's AES-GCM key from Local State,
#    then decrypt the password_value BLOB from Login Data.
Both paths lead to the same password:
plain
Wh4t1sV3raD0inG0nTh1sH0st
Step 5 — Locating the VeraCrypt Container
Search the triage for large files or files named backup:
powershell
Get-ChildItem -Path "C:\...\management-wants-a-word-forensics-hh-day-14" \
  -Recurse -Filter "backup" -ErrorAction SilentlyContinue
Result:
plain
Directory: ...\KAPE\C\Users\vera\Documents

Mode    LastWriteTime     Length Name
----    -------------     ------ ----
------  8/4/2026 2:15 PM  104857600 backup
At 100 MB, this is the VeraCrypt volume.
Step 6 — Mounting the VeraCrypt Container
Windows — VeraCrypt GUI
Select File → choose backup.
Mount → pick a drive letter (e.g., Z:).
Enter the password:
plain
Wh4t1sV3raD0inG0nTh1sH0st
Click OK. The volume mounts read/write.
Linux — cryptsetup
bash
sudo cryptsetup open --type tcrypt --veracrypt \
  "/path/to/backup" veracontainer

sudo mkdir -p /mnt/veradata
sudo mount -o ro /dev/mapper/veracontainer /mnt/veradata
Step 7 — Finding the Flag
Inside the mounted volume:
plain
Z:\ (or /mnt/veradata)
├── secret_financial_documents/
│   ├── important_invoice_byte_lotus.pdf
│   └── transactions.csv
└── ...
Open important_invoice_byte_lotus.pdf:
Screenshot — Invoice PDF:
The flag is embedded in the invoice line-item description.
Flag format: THM{...} (redacted in public writeups)
Attack Chain Summary
plain
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  KAPE Triage    │────▶│ Chrome Login Data │────▶│  SecureVault Pass  │
│  (Windows Dump) │     │  + History        │     │  Wh4t1sV3raD0in... │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
                                                          │
                                                          ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  PDF Invoice    │◀────│  VeraCrypt Vol    │◀────│  Mount Container    │
│  (THM{...})     │     │  secret_financial │     │  backup (100 MB)    │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
Key Takeaways
Forensics is a chain — One artifact leads to the next. There is no single "exploit".
Browsers are goldmines — Saved passwords, history, and autofill often contain the keys to the kingdom.
DPAPI security is user-password dependent — Crack the local NT hash, decrypt the master key, and Chrome falls open.
Version numbers are clues — 1.26.29 was VeraCrypt, not Chrome.
Documents hide secrets in plain sight — Always inspect PDFs, CSVs, and office files inside encrypted containers.
Tools Used
Table
Tool	Purpose
DB Browser for SQLite	Inspect Chrome History and Login Data
ChromePass (NirSoft)	Decrypt Chrome saved passwords
VeraCrypt	Mount the encrypted container
Impacket (optional)	secretsdump + dpapi masterkey decryption
PowerShell / Bash	File searching and triage navigation
References
TryHackMe — Management Wants a Word
VeraCrypt Downloads
NirSoft ChromePass
DB Browser for SQLite
