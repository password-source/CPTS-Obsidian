# 🔐 1. Credential Hunting in Windows

> [!info]  
> **Credential Hunting** is the process of searching a compromised Windows system for **stored credentials** such as passwords, usernames, API keys, connection strings, configuration secrets, and other authentication material.
> 
> Once we obtain access to a Windows machine through **GUI or CLI**, credential hunting should become part of our enumeration workflow.
> 
> 🎯 **Goal:** Find credentials that can provide **additional access, privilege escalation, lateral movement, or access to sensitive services.**

---

# 🧠 2. The Core Mental Model

Do **not** randomly search the entire filesystem and hope to find a password.

Instead, ask:

> **"How is this computer being used, and where would credentials be required for those activities?"**

For example, if we compromise an **IT administrator's workstation**, we should expect to find things related to:

```text
IT Administrator Workstation
        │
        ├── 🖥️ Remote administration
        │      └── RDP / SSH / WinSCP
        │
        ├── 🌐 Web browsers
        │      └── Saved website credentials
        │
        ├── 🔐 Password managers
        │      └── KeePass databases
        │
        ├── ⚙️ Administration tools
        │      └── OpenVPN / WinSCP / etc.
        │
        ├── 📜 Scripts
        │      └── Hardcoded passwords
        │
        └── 📁 Configuration files
               └── Connection strings / credentials
```

> [!tip]  
> **Think like the user, not like the filesystem.**
> 
> The more you understand the target's role, the more targeted your credential hunt becomes.

---

# 🔎 3. Search-Centric Credential Hunting

Modern Windows and applications provide extensive search functionality.

This allows us to search for **likely credential-related terms** instead of manually opening thousands of files.

### 🔑 Useful Search Terms

Keep these keywords in your credential-hunting cheat sheet:

```text
password
passphrase
key
username
user account
creds
users
passkeys
configuration
dbcredential
dbpassword
pwd
login
credentials
```

### 🧠 Why These Terms Matter

Credentials are frequently stored in:

- Configuration files
    
- Scripts
    
- Application databases
    
- Documentation
    
- Source code
    
- User-created files
    
- Password managers
    
- Browser credential stores
    
- Administration tools
    

For example:

```text
config.xml
web.config
backup.ps1
deploy.ps1
database.config
unattend.xml
passwords.txt
```

---

# 🪟 4. Windows Search

If we have access to the **GUI**, Windows Search can be useful for quickly locating files and settings containing credential-related terms.

Windows Search can search across:

- ⚙️ Windows settings
    
- 📁 Files
    
- 📦 Applications
    
- 🔎 File contents in supported locations
    

For example, searching for:

```text
pass
```

may return files, applications, or Windows settings related to passwords.

> [!important]  
> Windows Search is particularly useful during an initial GUI-based assessment because it requires very little tooling and can quickly reveal user-created files or documentation.

---

# 🧰 5. LaZagne

[LaZagne GitHub repository](https://github.com/AlessandroZ/LaZagne?utm_source=chatgpt.com)

**LaZagne** is a credential-recovery tool designed to search for credentials stored by various applications.

Instead of manually investigating every application, LaZagne uses different **modules**, with each module targeting particular software or credential locations.

---

## 📊 5.1 LaZagne Modules

|ModuleDescription||
|---|---|
|browsers|Extracts passwords from various browsers including Chromium, Firefox, Microsoft Edge, and Opera|
|chats|Extracts passwords from various chat applications including Skype|
|mails|Searches through mailboxes for passwords including Outlook and Thunderbird|
|memory|Dumps passwords from memory, targeting KeePass and LSASS|
|sysadmin|Extracts passwords from the configuration files of various sysadmin tools like OpenVPN and WinSCP|
|windows|Extracts Windows-specific credentials targeting LSA secrets, Credential Manager, and more|
|wifi|Dumps WiFi credentials|

> [!important]  
> The **browser** module is particularly interesting because browsers commonly provide built-in password storage.

---

# 🌐 6. Browser Credential Hunting

Popular browsers include:

- Google Chrome
    
- Microsoft Edge
    
- Firefox
    
- Opera
    

Browsers may store credentials for websites and applications.

However, these credentials are generally **encrypted**, rather than simply sitting in plaintext.

```text
Browser
   │
   ├── Saved username
   ├── Saved password
   └── Encrypted credential database
                │
                ↓
        Browser-specific
        decryption required
```

Tools exist that can assist with recovering browser-stored credentials, including:

[firefox_decrypt](https://github.com/unode/firefox_decrypt?utm_source=chatgpt.com)

[decrypt-chrome-passwords](https://github.com/ohyicong/decrypt-chrome-passwords?utm_source=chatgpt.com)

LaZagne supports **35 different browsers on Windows**.

> [!warning]  
> "Encrypted" does **not** necessarily mean "inaccessible." If an attacker obtains the necessary user context and decryption material, stored browser credentials may be recoverable.

---

# 🛠️ 7. Running LaZagne

It is useful to keep a **standalone copy** of LaZagne on the attack machine so it can be transferred to the target when needed.

[LaZagne Releases](https://github.com/AlessandroZ/LaZagne/releases/?utm_source=chatgpt.com)

For example, `LaZagne.exe` can be transferred to a Windows target through an established RDP session.

If using `xfreerdp`, files can be copied into the RDP session.

### ▶️ Execute All Modules

Once `LaZagne.exe` is on the target:

```cmd
C:\Users\bob\Desktop> start LaZagne.exe all
```

### What `all` Means

```text
LaZagne.exe
     │
     └── all
          │
          ├── 🌐 Browsers
          ├── 💬 Chats
          ├── 📧 Mail
          ├── 🧠 Memory
          ├── ⚙️ Sysadmin tools
          ├── 🪟 Windows credentials
          └── 📶 WiFi
```

The `all` option tells LaZagne to execute **all included modules**.

---

# 🔬 8. Verbose Mode: `-vv`

We can use:

```text
-vv
```

to obtain more detailed output about what LaZagne is doing in the background.

This is useful for **understanding the tool** and seeing which applications and credential stores it attempts to access.

> [!tip]  
> During learning, verbose output is valuable because it helps you understand **where credentials are actually being searched for**, rather than treating LaZagne as a black box.

---

# 💥 9. Example LaZagne Output

A successful credential discovery may look like:

```text
|====================================================================|
|                                                                    |
|                        The LaZagne Project                         |
|                                                                    |
|                          ! BANG BANG !                             |
|                                                                    |
|====================================================================|


########## User: bob ##########

------------------- Winscp passwords -----------------

[+] Password found !!!
URL: 10.129.202.51
Login: admin
Password: SteveisReallyCool123
Port: 22
```

### 🧠 What We Learned

The output gives us several useful pieces of information:

```text
Application → WinSCP
Target      → 10.129.202.51
Username    → admin
Password    → SteveisReallyCool123
Port        → 22
```

This could potentially provide access to another system/service.

> [!success]  
> **Credential hunting is not only about finding passwords.**
> 
> Always record the **username + password + target + service + port + application** because the credential may be useful somewhere other than the system where you found it.

---

# 🔍 10. `findstr` — Search Files for Credential Patterns

Windows provides the built-in `findstr` utility for searching for patterns across files.

A useful credential-hunting command is:

```cmd
C:\> findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

### 🧩 Important Options

```text
findstr
  │
  ├── /S  → Search subdirectories
  ├── /I  → Case-insensitive search
  ├── /M  → Print only filenames containing a match
  └── /C: → Search for the specified literal string
```

So:

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

essentially means:

> 🔎 Search recursively and case-insensitively for the literal term `password`, and return the filenames containing matches in the specified file types.

### 📁 File Types Being Searched

```text
*.txt       → Text files
*.ini       → Initialization/configuration files
*.cfg       → Configuration files
*.config    → Configuration files
*.xml       → XML files
*.git       → Git-related files
*.ps1       → PowerShell scripts
*.yml       → YAML configuration files
```

> [!important]  
> Scripts and configuration files are especially valuable because administrators and developers sometimes accidentally **hardcode credentials** into them.

---

# 🗺️ 11. Credential Hunting Depends on the System's Role

There is no single perfect credential-hunting command.

Your approach should change depending on what type of Windows machine you compromise.

### 🖥️ Windows Desktop

Potential targets:

```text
🌐 Browser passwords
🔐 Credential Manager
🔑 KeePass
📁 User documents
📜 PowerShell scripts
📡 WiFi credentials
🛠️ Admin tools
```

### 🖥️ Windows Server

Potential targets:

```text
⚙️ Application configuration
🌐 web.config
📜 Deployment scripts
🗄️ Database credentials
📂 IT shares
🏢 SYSVOL
🔐 Service credentials
👤 User/computer descriptions
```

### 🧠 Key Principle

```text
          What is this machine?
                   │
                   ↓
          What is it used for?
                   │
                   ↓
       What applications/services?
                   │
                   ↓
       Where do those require creds?
                   │
                   ↓
       🎯 Search those locations first
```

---

# 📍 12. Additional Credential Locations

Credential hunting should extend beyond the local user's obvious files.

## 🏢 Active Directory / SYSVOL

Look for:

- Passwords in **Group Policy** within the `SYSVOL` share
    
- Passwords in scripts stored in `SYSVOL`
    

## 📂 IT Shares

Search for:

- Passwords inside scripts
    
- Configuration files
    
- Administrative documentation
    
- Backup files
    

## 🌐 `web.config`

On development machines and IT shares, inspect:

```text
web.config
```

These files may contain application/database connection information.

## 🛠️ `unattend.xml`

Search for:

```text
unattend.xml
```

Automated Windows installation/configuration files may contain credentials or other sensitive configuration information.

## 👤 AD User / Computer Descriptions

Check:

```text
User description
Computer description
```

Administrators sometimes place useful information in description fields.

## 🔐 KeePass

Look for KeePass database files.

Typical workflow:

```text
Find KeePass database
        ↓
Obtain/guess/crack master password
        ↓
Open database
        ↓
Recover stored credentials
```

> [!warning]  
> Finding a KeePass database does **not** automatically mean you can read its contents. The database is protected by a master password.

## 📁 User Systems and Shares

Search user-accessible systems and network shares for files with names such as:

```text
pass.txt
passwords.docx
passwords.xlsx
```

Also consider collaborative storage such as **SharePoint**.

---

# 🎯 13. Credential Hunting Workflow

A practical CPTS workflow:

```text
                 🖥️ INITIAL ACCESS
                        │
                        ↓
              🧭 IDENTIFY THE USER
                        │
                        ↓
              🧠 UNDERSTAND THE ROLE
                        │
              ┌─────────┴─────────┐
              ↓                   ↓
        🖥️ GUI ACCESS        💻 CLI ACCESS
              │                   │
              └─────────┬─────────┘
                        ↓
                 🔎 SEARCH TERMS
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
     🪟 Windows      🧰 LaZagne    🔍 findstr
       Search
          │             │             │
          └─────────────┼─────────────┘
                        ↓
               📁 CONFIG / SCRIPTS
                        │
                        ↓
               🌐 BROWSERS / APPS
                        │
                        ↓
                🔐 PASSWORD STORES
                        │
                        ↓
              🏢 SHARES / SYSVOL
                        │
                        ↓
                👤 USER / COMPUTER
                   DESCRIPTIONS
                        │
                        ↓
                💎 VALID CREDENTIAL
                        │
                        ↓
        🔓 PRIVESC / LATERAL MOVEMENT /
             ADDITIONAL ACCESS
```

---

# 🧠 14. High-Value Mental Model

Think of credential hunting as answering **three questions**:

### 1️⃣ Who is using the machine?

```text
Normal user?
Developer?
IT administrator?
Domain administrator?
Service account?
```

### 2️⃣ What do they use?

```text
Browser?
WinSCP?
OpenVPN?
PowerShell?
KeePass?
RDP?
Database tools?
```

### 3️⃣ Where would those applications store credentials?

```text
Browser DB
Config file
Script
Credential Manager
LSA secrets
KeePass database
Application database
Network share
```

This transforms credential hunting from:

```text
❌ Randomly search everything
```

into:

```text
✅ User/Role
      ↓
   Applications
      ↓
Credential storage locations
      ↓
    Credentials
      ↓
 Additional access
```

---

# 📊 15. Credential Hunting Cheat Sheet

|Target|What to Look For|
|---|---|
|🪟 Windows Search|Files/settings containing credential-related keywords|
|🌐 Browsers|Saved website credentials|
|🧰 LaZagne|Credentials stored by supported applications|
|🔍 `findstr`|Passwords and other secrets inside files|
|📜 Scripts|Hardcoded passwords and connection information|
|⚙️ Configuration files|Database/application credentials|
|🏢 SYSVOL|Group Policy and scripts containing passwords|
|📂 IT shares|Scripts, configuration files, documentation|
|🌐 `web.config`|Application/database credentials|
|🛠️ `unattend.xml`|Installation/configuration credentials|
|👤 AD descriptions|Passwords or useful account information|
|🔐 KeePass|Password databases|
|📁 User files|`pass.txt`, `passwords.docx`, `passwords.xlsx`, etc.|
|☁️ SharePoint|User-created documents containing credentials|

---

# ⚡ 16. CPTS Quick Reference

### 🔎 Search Keywords

```text
password
passphrase
key
username
user account
creds
users
passkeys
configuration
dbcredential
dbpassword
pwd
login
credentials
```

### 🧰 LaZagne

```cmd
start LaZagne.exe all
```

Verbose:

```cmd
start LaZagne.exe all -vv
```

### 🔍 `findstr`

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

### 📍 High-Value Locations

```text
SYSVOL
IT shares
web.config
unattend.xml
KeePass databases
Browser credential stores
Credential Manager
User documents
Scripts
Configuration files
AD user descriptions
AD computer descriptions
SharePoint
```

---

# ⚠️ 17. Important CPTS Lessons

> [!important]  
> **1. Context beats random searching.**
> 
> Identify the machine's purpose and the user's role first.

> [!important]  
> **2. Credentials are often stored outside dedicated password stores.**
> 
> Scripts, configuration files, documents, shares, and descriptions can all contain secrets.

> [!warning]  
> **3. Don't assume encrypted means impossible to recover.**
> 
> Browser and application credentials may be encrypted but can sometimes be recovered when the necessary user context/decryption material is available.

> [!tip]  
> **4. Search broadly, but prioritize intelligently.**
> 
> Start with high-probability locations based on the system's role, then expand outward.

> [!success]  
> **5. Every credential discovery can change the attack path.**
> 
> A password found on one workstation might provide:
> 
> `Lateral Movement → Server Access → Privilege Escalation → Domain Access`

---

# ✅ 18. Key Takeaways

```text
🎯 CREDENTIAL HUNTING
        │
        ├── 🧠 Understand the target/user
        │
        ├── 🔎 Search useful keywords
        │
        ├── 🪟 Use Windows Search
        │
        ├── 🧰 Use LaZagne
        │
        ├── 🔍 Use findstr
        │
        ├── 📁 Search scripts/configuration
        │
        ├── 🌐 Check browsers
        │
        ├── 🏢 Check SYSVOL / IT shares
        │
        ├── 🔐 Check KeePass / credential stores
        │
        └── 💎 Reuse discovered credentials
                 ↓
          🔓 MORE ACCESS
```

> [!success]  
> **CPTS Mental Model:**
> 
> **Initial Access → Understand User/Role → Identify Applications → Identify Credential Storage → Search → Recover Credentials → Test/Re-use → Expand Access**