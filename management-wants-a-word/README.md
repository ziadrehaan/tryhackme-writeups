#Hacker Holidays 2026 · Day 14

**Category:** Forensics · Windows · Cryptography  
**Difficulty:** Hard  
**Points:** 120  
**Flag:** `THM{..._..._..._...}` *(redacted)*  

> *"It was always her. It was never a bug; it was the business model."*

---

## 📌 Overview
This room is a Windows forensics chain, not a single exploit. You are given a KAPE triage dump of a laptop left behind by "Vera" in Room 214. The objective is to follow the artifact trail from browser secrets through DPAPI to a VeraCrypt container, and finally recover the flag from a hidden document.

### 🔗 The Chain:
```text
KAPE Triage ➔ Chrome Artifacts ➔ Saved Password ➔ VeraCrypt Container ➔ PDF Flag
🚀 Step-by-Step WalkthroughStep 1 — Understanding the Hint@0xMia's post gives two critical clues:"a browser will remember things for you that you never told anyone else 💀""why did Patch tell me this version number 1.26.29 idk what it means"Analysis:Browser memory ➔ Chrome stores saved passwords in Login Data.1.26.29 ➔ This is a VeraCrypt release version, not a Chrome version."not every hidden file needs a password cracker" ➔ The password is already stored somewhere in the triage; you just need to find it.Step 2 — Triage & Initial ReconAfter extracting the archive, the structure is a standard KAPE output:Plaintextmanagement-wants-a-word-forensics-hh-day-14/
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
│       │       │           └── Protect/           ← DPAPI master keys
│       │       └── Documents/
│       │           └── backup                     ← 100 MB container
│       └── Windows/
│           └── System32/
│               └── config/
│                   ├── SAM
│                   ├── SYSTEM
│                   └── ...
The presence of SAM, SYSTEM, and Vera's full profile tells us we can do offline credential recovery.Step 3 — Chrome History AnalysisOpen History in DB Browser for SQLite and browse the urls table:SQLSELECT id, url, title FROM urls;
Key findings:http://bytelotus.thm:8080/ — SecureVault Portalhttp://bytelotus.thm:8080/login — Login pageGoogle searches for "chrome cves", "how to exfiltrate data red teaming", "tryhackme"Vera had a local vault application running on bytelotus.thm:8080.Step 4 — Extracting the Saved PasswordOpen Login Data in DB Browser and browse the logins table:origin_urlusername_valuepassword_valuehttp://bytelotus.thm:8080/VeraSecretVaultBLOB (encrypted)The password_value is encrypted with Windows DPAPI and then AES-GCM via Chrome.Option A — ChromePass (Easiest)Download ChromePass from NirSoft.Launch it and go to File ➔ Advanced Options ➔ Select User Data Folder.Point it at the User Data folder (not Default, the parent directory).Result:PlaintextURL:      [http://bytelotus.thm:8080/](http://bytelotus.thm:8080/)
Username: VeraSecretVault
Password: Wh4t1sV3raD0inG0nTh1sH0st
Option B — Manual DPAPI + Python (Advanced)Bash# 1. Extract NT hash from SAM/SYSTEM
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# 2. Crack Vera's password with John/Hashcat
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt nt.john
# → minivera

# 3. Decrypt the DPAPI master key
impacket-dpapi masterkey -file ".../Protect/SID/c90719ef-..." \
  -sid "S-1-5-21-..." \
  -password minivera
Step 5 — Locating the VeraCrypt ContainerSearch the triage for large files or files named backup:PowerShellGet-ChildItem -Path "C:\...\management-wants-a-word-forensics-hh-day-14" `
  -Recurse -Filter "backup" -ErrorAction SilentlyContinue
Result:PlaintextDirectory: ...\KAPE\C\Users\vera\Documents

Mode        LastWriteTime     Length     Name
----        -------------     ------     ----
------      8/4/2026 2:15 PM  104857600  backup
At 100 MB, this is the VeraCrypt volume.Step 6 — Mounting the VeraCrypt ContainerWindows — VeraCrypt GUISelect File ➔ choose backup.Mount ➔ pick a drive letter (e.g., Z:).Enter Password: Wh4t1sV3raD0inG0nTh1sH0stLinux — cryptsetupBashsudo cryptsetup open --type tcrypt --veracrypt "/path/to/backup" veracontainer
sudo mkdir -p /mnt/veradata
sudo mount -o ro /dev/mapper/veracontainer /mnt/veradata
Step 7 — Finding the FlagInside the mounted volume:PlaintextZ:\ (or /mnt/veradata)
├── secret_financial_documents/
│   ├── important_invoice_byte_lotus.pdf
│   └── transactions.csv
└── ...
Open important_invoice_byte_lotus.pdf to retrieve the flag embedded in the invoice line-item description.🛠️ Tools UsedToolPurposeDB Browser for SQLiteInspect Chrome History and Login DataChromePass (NirSoft)Decrypt Chrome saved passwordsVeraCryptMount the encrypted containerImpacketsecretsdump + dpapi masterkey decryptionPowerShell / BashFile searching and triage navigation💡 Key TakeawaysForensics is a chain: One artifact leads to the next. There is no single "exploit".Browsers are goldmines: Saved passwords, history, and autofill often contain the keys to the kingdom.DPAPI security relies on user password: Crack the local NT hash, decrypt the master key, and Chrome falls open.Version numbers are clues: 1.26.29 was VeraCrypt, not Chrome.Documents hide secrets in plain sight: Always inspect PDFs, CSVs, and office files inside encrypted containers.
