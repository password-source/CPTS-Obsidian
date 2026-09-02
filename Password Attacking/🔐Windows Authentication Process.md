# 🔐 1. Windows Authentication Process

> [!info]  
> **Windows authentication** is not handled by a single component. It is a chain of processes and authentication packages responsible for collecting credentials, validating them, creating a logon session, and enforcing security policies.
> 
> The most important components to understand for CPTS are:
> 
> **WinLogon → LogonUI → Credential Provider → LSASS → Authentication Package → SAM / Active Directory**

```text
                    👤 USER
                      │
                      ▼
                🖥️ WinLogon
                      │
                      ▼
                 🪟 LogonUI
                      │
                      ▼
             🔑 Credential Provider
                      │
                      │ Credentials
                      ▼
                  🛡️ LSASS
                      │
             ┌────────┴─────────┐
             ▼                  ▼
         🧬 NTLM            🛡️ Kerberos
             │                  │
             ▼                  ▼
        🗄️ SAM              🏢 Active Directory
             │                  │
             ▼                  ▼
       Local Account        Domain Account
```

> [!important]  
> **Mental model:**  
> Think of **LSASS as the authentication gatekeeper**. WinLogon collects the credentials, but LSASS and its authentication packages perform the actual security checks.

---

# 🧠 2. Local Security Authority (LSA)

The **Local Security Authority (LSA)** is a protected Windows subsystem responsible for enforcing security policies and managing authentication.

It handles areas such as:

- 👤 User authentication
    
- 🔐 Local logon security
    
- 🛂 Access control and permission checks
    
- 🆔 Translation between usernames and **Security Identifiers (SIDs)**
    
- 📜 Generation and forwarding of security audit events
    
- 🔑 Management of authentication packages
    

On a **Domain Controller (DC)**, these security policies and accounts apply to the domain and are backed by **Active Directory**.

```text
                🛡️ LSA
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   Authentication Access    Auditing
       │         Control       │
       ▼                       ▼
  NTLM/Kerberos          Windows Event Log
```

---

# 🖥️ 3. WinLogon

**WinLogon** is a trusted Windows system process responsible for managing security-related interactions with the user.

Its responsibilities include:

- 🚪 Launching **LogonUI**
    
- 🔑 Collecting/handling login credentials through credential providers
    
- 🔄 Handling password changes
    
- 🔒 Locking the workstation
    
- 🔓 Unlocking the workstation
    

At login:

```text
Keyboard Input
      │
      ▼
  Win32k.sys
      │ RPC
      ▼
   WinLogon
      │
      ▼
   LogonUI
      │
      ▼
Credential Provider
      │
      ▼
   Credentials
      │
      ▼
    LSASS
```

> [!important]  
> **WinLogon is not the component that ultimately validates the password.**
> 
> Its important role is to securely receive the login request and pass the collected credentials to **LSASS** for authentication.

### 🔑 Credential Providers

WinLogon relies on **Credential Providers** to obtain account credentials.

Credential providers are:

- Implemented as **COM objects**
    
- Implemented through DLLs
    
- Responsible for providing credential information to WinLogon
    

Examples of credential input can include:

```text
👤 Username
🔑 Password
🔢 PIN
🔐 Smart card / other authentication methods
```

---

# 🪟 4. LogonUI

`LogonUI.exe` provides the graphical interface through which users interact with the Windows login process.

The simplified flow is:

```text
WinLogon
   │
   └── launches
          │
          ▼
      LogonUI
          │
          └── uses
                │
                ▼
       Credential Provider
                │
                ▼
          User Credentials
```

The collected credentials are then passed through **WinLogon → LSASS** for authentication.

---

# 🛡️ 5. LSASS — Local Security Authority Subsystem Service

**LSASS** stands for:

> **Local Security Authority Subsystem Service**

Its executable is:

```text
%SystemRoot%\System32\Lsass.exe
```

LSASS is one of the most important processes to understand during Windows credential attacks.

### 🎯 Main Responsibilities

LSASS:

- 🔐 Governs authentication processes
    
- 🛡️ Enforces local security policies
    
- 👤 Authenticates users
    
- 🧩 Loads and manages authentication/security packages
    
- 📜 Forwards security audit events to the Windows Event Log
    
- 🛂 Participates in access-control decisions
    

> [!important]  
> **CPTS relevance:** LSASS is a major credential target because authentication material and security-related secrets can exist in its process memory.

```text
                 🛡️ LSASS
                    │
       ┌────────────┼─────────────┐
       ▼            ▼             ▼
   Security     Authentication   Auditing
    Policy          Packages
       │        ┌────┴────┐          │
       │        ▼         ▼          ▼
       │       NTLM    Kerberos   Event Log
       │
       ▼
 Access Control
```

---

# 🧬 6. Windows Authentication Packages

Authentication packages are components, typically implemented as **DLLs**, that perform authentication-related operations.

The major packages/components relevant to CPTS are:

|**Authentication PackagesDescription**||
|---|---|
|`Lsasrv.dll`|The LSA Server service both enforces security policies and acts as the security package manager for the LSA. The LSA contains the Negotiate function, which selects either the NTLM or Kerberos protocol after determining which protocol is to be successful.|
|`Msv1_0.dll`|Authentication package for local machine logons that don't require custom authentication.|
|`Samsrv.dll`|The Security Accounts Manager (SAM) stores local security accounts, enforces locally stored policies, and supports APIs.|
|`Kerberos.dll`|Security package loaded by the LSA for Kerberos-based authentication on a machine.|
|`Netlogon.dll`|Network-based logon service.|
|`Ntdsa.dll`|Directory System Agent (DSA) that manages the Active Directory database (ntds.dit), processes LDAP queries, and handles replication between domain controllers. Only loaded on Domain Controllers.|

### 🧠 What to Remember

```text
Lsasrv.dll
   │
   └── LSA security package management
             │
             ├── NTLM → Msv1_0.dll
             │
             └── Kerberos → Kerberos.dll

Samsrv.dll
   │
   └── Local SAM accounts

Ntdsa.dll
   │
   └── Active Directory / ntds.dit
```

> [!tip]  
> For CPTS, the most important distinction is:
> 
> **Local authentication → SAM**
> 
> **Domain authentication → Active Directory / NTDS**
> 
> **Authentication processing → LSASS**

---

# 🔄 7. Complete Interactive Logon Flow

A local interactive logon involves several components working together.

```text
👤 USER
  │
  │ Enters credentials
  ▼
🖥️ WinLogon
  │
  │ Launches
  ▼
🪟 LogonUI
  │
  │ Uses
  ▼
🔑 Credential Provider
  │
  │ Supplies credentials
  ▼
🛡️ LSASS
  │
  │ Selects/uses authentication package
  ▼
┌───────────────────────┐
│ Authentication Package│
│                       │
│ NTLM / Kerberos       │
└───────────┬───────────┘
            │
       ┌────┴─────┐
       ▼          ▼
    🗄️ SAM      🏢 AD
    Local       Domain
    Account     Account
```

> [!success]  
> **CPTS mental model:**
> 
> `WinLogon` = manages the login interaction  
> `LogonUI` = displays the login interface  
> `Credential Provider` = obtains credentials  
> `LSASS` = authentication/security gatekeeper  
> `Authentication Package` = performs the authentication logic  
> `SAM` = local account database  
> `NTDS.dit` = domain account database

---

# 🗃️ 8. SAM — Security Account Manager

The **Security Account Manager (SAM)** is the local Windows database responsible for storing local user account information.

Its file is located at:

```text
%SystemRoot%\system32\config\SAM
```

It is mounted in the Registry under:

```text
HKLM\SAM
```

### 🔐 What SAM Stores

Local account passwords are stored as **password hashes**, historically including:

- `LM` hashes
    
- `NTLM` hashes
    

The hashes are cryptographically protected rather than storing plaintext passwords.

> [!warning]  
> Accessing the SAM database requires **SYSTEM-level privileges**.

### 🧠 SAM Mental Model

```text
🖥️ Windows Computer
       │
       ▼
     SAM
       │
       ├── 👤 Local users
       ├── 🔑 Password hashes
       └── 🛡️ Local security information
```

---

# 🏠 9. Workgroup vs Domain Authentication

Windows systems can operate in either a **workgroup** or a **domain** environment.

## 🏠 Workgroup

In a workgroup:

```text
             🖥️ Windows Host
                    │
                    ▼
                  🗄️ SAM
                    │
             ┌──────┴──────┐
             ▼             ▼
          User A        User B
```

Each computer manages its own local users through its local SAM database.

### Example

If you have:

```text
PC01
PC02
PC03
```

each machine can have its own local accounts:

```text
PC01 → SAM
PC02 → SAM
PC03 → SAM
```

---

# 🏢 10. Domain Authentication

In an Active Directory domain:

```text
                 🖥️ Client
                    │
                    │ Logon Request
                    ▼
              🏢 Domain Controller
                    │
                    ▼
                 NTDS.dit
                    │
                    ▼
             👤 Domain Account
```

The **Domain Controller (DC)** validates domain credentials using the Active Directory database.

The primary database is:

```text
%SystemRoot%\ntds.dit
```

> [!important]  
> **Local account → SAM**
> 
> **Domain account → Domain Controller → Active Directory / `NTDS.dit`**

---

# 🔐 11. SYSKEY

Microsoft introduced **SYSKEY (`syskey.exe`)** in Windows NT 4.0 to improve protection against offline attacks against the SAM database.

When enabled, SYSKEY partially encrypted the SAM database on disk using a system-generated key.

```text
SAM
 │
 │ Password hashes
 ▼
🔐 SYSKEY protection
 │
 ▼
Encrypted/protected data on disk
```

> [!warning]  
> SYSKEY is primarily relevant as **historical Windows credential-storage context**. Modern Windows security mechanisms have evolved significantly beyond this legacy feature.

---

# 🔑 12. Credential Manager

**Credential Manager** is a built-in Windows feature that allows users to store credentials used for:

- 🌐 Network resources
    
- 🌍 Websites
    
- 🖥️ Applications
    
- 🔐 Other saved authentication targets
    

Saved credentials are stored per-user inside the user's **Credential Locker**.

A relevant location is:

```powershell
PS C:\Users\[Username]\AppData\Local\Microsoft\[Vault/Credentials]\
```

### 🧠 Credential Manager Mental Model

```text
👤 User
  │
  │ Saves credential
  ▼
🔐 Credential Manager
  │
  ▼
🔒 Credential Locker
  │
  └── Encrypted credentials
```

> [!important]  
> Credential Manager is different from the **SAM**:
> 
> **SAM** → local Windows account authentication data  
> **Credential Manager** → saved credentials used for accessing resources, applications, websites, etc.

During credential attacks, saved Credential Manager secrets can become valuable targets because they may provide reusable credentials for additional access.

---

# 🏢 13. NTDS.dit — Active Directory Database

In Windows domain environments, every writable Domain Controller normally maintains a copy of the Active Directory database:

```text
NTDS.dit
```

Location:

```text
%SystemRoot%\ntds.dit
```

The database is synchronized between Domain Controllers, with the exception of **Read-Only Domain Controllers (RODCs)**, which have different replication/security characteristics.

### 📦 Information Stored in NTDS.dit

`NTDS.dit` contains Active Directory information including:

- 👤 User accounts
    
- 🔑 Password hashes
    
- 👥 Group accounts
    
- 💻 Computer accounts
    
- 📜 Group Policy Objects
    
- 🏢 Other Active Directory directory data
    

```text
                 🏢 Domain Controller
                         │
                         ▼
                    NTDS.dit
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   👤 Users          👥 Groups       💻 Computers
        │
        ▼
 🔑 Password hashes
```

> [!important]  
> **`NTDS.dit` is one of the highest-value credential sources in an Active Directory environment.**
> 
> If an attacker can obtain and extract credential material from it, they may recover password hashes and other domain information that can enable further attacks.

---

# ⚔️ 14. Credential Storage — Pentesting Perspective

Understanding where credentials live tells you **what to target after gaining access**.

```text
                 🎯 Windows Credential Sources
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
       🗄️ SAM       🔐 Credential       🏢 NTDS.dit
                         Manager
          │                │                │
          ▼                ▼                ▼
    Local account     Saved credentials   Domain accounts
       hashes          / secrets             + hashes
```

### 🎯 Attacker's Perspective

```text
Initial Access
     │
     ▼
🖥️ Compromise Windows Host
     │
     ├──► 🔎 Inspect local credentials
     │          │
     │          └──► SAM
     │
     ├──► 🔎 Inspect saved credentials
     │          │
     │          └──► Credential Manager
     │
     └──► 🏢 If DC / sufficient privileges
                │
                └──► NTDS.dit
                         │
                         ▼
                  🔑 Domain hashes
                         │
                         ▼
                 Further Access
```

> [!warning]  
> Credential extraction should be understood as part of a broader attack chain:
> 
> **Compromise → Identify credential source → Extract → Crack/Pass/Reuse → Escalate or Move Laterally**

---

# 📊 15. Critical Credential Sources Cheat Sheet

|Component|Purpose|Credential/Account Scope|Important Location|
|---|---|---|---|
|**SAM**|Stores local Windows accounts|Local accounts|`%SystemRoot%\system32\config\SAM`|
|**Credential Manager**|Stores saved credentials|Per-user saved credentials|`C:\Users\[Username]\AppData\Local\Microsoft\[Vault/Credentials]\`|
|**NTDS.dit**|Active Directory database|Domain accounts|`%SystemRoot%\ntds.dit`|
|**LSASS**|Handles authentication/security operations|Active authentication/session material|`%SystemRoot%\System32\Lsass.exe`|

---

# 🧠 16. CPTS Authentication Mental Model

The easiest way to remember the entire architecture:

```text
                    👤 USER
                      │
                      ▼
               🖥️ WINLOGON
            "Manage the login"
                      │
                      ▼
                 🪟 LOGONUI
             "Show the login UI"
                      │
                      ▼
          🔑 CREDENTIAL PROVIDER
            "Get the credentials"
                      │
                      ▼
                  🛡️ LSASS
          "Security/authentication gate"
                      │
              ┌───────┴────────┐
              ▼                ▼
            NTLM            KERBEROS
              │                │
              └───────┬────────┘
                      ▼
              Authentication
                      │
             ┌────────┴────────┐
             ▼                 ▼
          🗄️ SAM             🏢 AD
         Local Auth        Domain Auth
                               │
                               ▼
                          📁 NTDS.dit
```

---

# ⚡ 17. CPTS Quick Reference

|🎯 Component|🧠 Remember It As|
|---|---|
|**WinLogon**|Manages secure login interactions|
|**LogonUI**|Displays the Windows login interface|
|**Credential Provider**|Collects/provides credentials|
|**LSASS**|Authentication & security gatekeeper|
|**Lsasrv.dll**|LSA server/security package manager|
|**Msv1_0.dll**|NTLM/local authentication package|
|**Kerberos.dll**|Kerberos security package|
|**Samsrv.dll**|SAM management|
|**Netlogon.dll**|Network-based logon service|
|**Ntdsa.dll**|Active Directory database/DSA functionality on DCs|
|**SAM**|Local account database|
|**Credential Manager**|Saved per-user credentials|
|**NTDS.dit**|Active Directory database|
|**LSASS.exe**|Windows authentication/security process|

---

# 🎯 18. Key Takeaways

> [!success]
> 
> - **WinLogon** manages security-related login interactions and launches `LogonUI`.
>     
> - **LogonUI** presents the login interface.
>     
> - **Credential Providers** obtain the user's authentication credentials.
>     
> - **LSASS** is the central authentication/security gatekeeper.
>     
> - **Authentication packages** such as NTLM and Kerberos perform authentication-related operations.
>     
> - **SAM** stores local Windows account information and password hashes.
>     
> - **Credential Manager** stores encrypted credentials saved by individual users.
>     
> - **NTDS.dit** stores Active Directory domain data, including user accounts and password hashes.
>     
> - **Local authentication** generally involves the local **SAM**.
>     
> - **Domain authentication** involves the **Domain Controller and Active Directory**.
>     
> - For CPTS credential attacks, understanding **where credentials are stored** is just as important as knowing **how to crack or reuse them**.
>     

```text
🔐 WINDOWS CREDENTIAL MAP

LOCAL HOST
   │
   ├── 🗄️ SAM
   │      └── Local account hashes
   │
   ├── 🔐 Credential Manager
   │      └── Saved user credentials
   │
   └── 🛡️ LSASS
          └── Authentication/session material

DOMAIN CONTROLLER
   │
   └── 🏢 NTDS.dit
          ├── Domain users
          ├── Password hashes
          ├── Groups
          ├── Computers
          └── AD data
```

> [!important]  
> **The CPTS takeaway:**  
> When you compromise a Windows system, don't only ask **"How do I get SYSTEM?"**
> 
> Also ask:
> 
> **"Where does this system keep authentication material, and what can that material give me next?"**