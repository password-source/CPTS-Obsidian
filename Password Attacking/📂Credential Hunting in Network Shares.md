# 📂 1. Credential Hunting in Network Shares

> [!info]  
> **Goal:** Search network shares for accidentally exposed credentials, secrets, configuration files, and other sensitive information.
> 
> Corporate environments commonly use **SMB/network shares** to allow employees and teams to store and exchange files. These shares can become a **goldmine for attackers** when sensitive information is stored with overly broad access permissions.

```text
🏢 Corporate Environment
        │
        ▼
   📂 SMB Shares
        │
        ├── IT files
        ├── Scripts
        ├── Configurations
        ├── Documents
        └── Backups
              │
              ▼
        🔎 Credential Hunting
              │
              ▼
       🔑 Credentials / Secrets
              │
              ▼
      🎯 Further Access
```

---

# 🧠 2. What Makes Network Shares Valuable?

A network share may contain files that were never intended to be accessible to everyone.

Common examples include:

- 🔑 Plaintext passwords
    
- 🪪 Usernames
    
- 🎫 API tokens
    
- 🔐 Private keys
    
- ⚙️ Configuration files
    
- 📜 PowerShell scripts
    
- 🖥️ Batch scripts
    
- 📊 Spreadsheets
    
- 📝 Deployment files
    
- 💾 Backups
    
- 🏗️ Infrastructure information
    

> [!important]  
> **The goal isn't simply to enumerate shares.**
> 
> The real objective is to determine:
> 
> **Which accessible shares contain information that can lead to credentials, privilege escalation, or further access?**

---

# 🔎 3. Common Credential Patterns

Before using specialized tools, start by understanding what you're searching for.

## 📝 3.1 Useful Keywords

Search file contents for terms such as:

```text
passw
user
token
key
secret
```

These keywords can reveal credentials embedded inside configuration files, scripts, documentation, or other files.

---

## 📄 3.2 Interesting File Extensions

Prioritize files commonly associated with configuration or stored credentials:

```text
.ini
.cfg
.env
.xlsx
.ps1
.bat
```

For example:

```text
database.cfg
production.env
backup.ps1
passwords.xlsx
deploy.bat
```

> [!tip]  
> Don't restrict yourself to files with obvious names such as `password.txt`. Credentials are frequently buried in **configuration files and scripts**.

---

## 🏷️ 3.3 Interesting File Names

Look for filenames containing:

```text
config
user
passw
cred
initial
```

Examples:

```text
config.ini
initial_setup.txt
user_config.xml
credentials.cfg
password_backup.xlsx
```

---

# 🌐 4. Domain-Specific Searches

When targeting a Windows domain, domain names can be useful search terms.

For example, for:

```text
INLANEFREIGHT.LOCAL
```

search for:

```text
INLANEFREIGHT\
```

This may locate files containing domain-qualified usernames such as:

```text
INLANEFREIGHT\administrator
INLANEFREIGHT\jbader
```

> [!important]  
> **Localize your search keywords.**
> 
> If the target organization is German, for example, searching only for `User` may miss relevant files using `Benutzer`.
> 
> Adapt keyword searches to the target's **language and environment**.

---

# 🎯 5. Choose Shares Strategically

Scanning every available share isn't always efficient.

Imagine:

```text
10 shares × 10,000 files
        ↓
     100,000 files
        ↓
   🔎 Huge search
```

Instead, prioritize shares based on their likely value.

### 🥇 High-Value Targets

```text
IT
Finance
HR
Infrastructure
Development
Deployment
Backups
```

### 🥉 Lower-Priority Targets

```text
Company Photos
Marketing Images
Public Media
General Documents
```

> [!tip]  
> **Think like an attacker:** an IT share is more likely to contain scripts, configurations, credentials, and deployment information than a share containing company photographs.

---

# 💻 6. Start Simple Before Automating

Before using specialized tools, basic command-line searches can sometimes be enough.

A general Windows approach is:

```powershell
Get-ChildItem -Recurse -Include *.ext \\Server\Share | Select-String -Pattern ...
```

### 🧠 Methodology

```text
📂 Identify interesting share
        ↓
📄 Identify interesting extensions
        ↓
🔎 Search file contents
        ↓
🔑 Identify credentials/secrets
        ↓
🎯 Validate findings
```

When the environment becomes large, automation becomes much more useful.

---

# 🪟 7. Hunting from Windows

The primary tools covered here are:

```text
🛠️ Snaffler
🛠️ PowerHuntShares
```

---

# 🔍 8. Snaffler

[Snaffler GitHub repository](https://github.com/SnaffCon/Snaffler?utm_source=chatgpt.com)

**Snaffler** is a C# program designed to run on a **domain-joined Windows machine** and automatically:

- Discover accessible network shares
    
- Search those shares
    
- Identify interesting files
    
- Detect potentially sensitive information
    

A basic search:

```cmd
c:\Users\Public>Snaffler.exe -s
```

---

## 🧠 8.1 What Snaffler Does

When no specific computer or path targets are supplied, Snaffler can perform discovery automatically:

```text
Snaffler
   │
   ├── 🔎 DFS Discovery
   │
   ├── 🏢 Domain Computer Discovery
   │
   ├── 📂 Share Discovery
   │
   ├── 🔐 Permission Checks
   │
   └── 📄 Interesting File Detection
```

The example output shows Snaffler querying Active Directory, discovering computers, and identifying readable shares such as:

```text
ADMIN$
C$
Company
Finance
HR
IT
Marketing
NETLOGON
Sales
SYSVOL
```

It then identifies potentially interesting files, including:

```text
unattend.xml
```

with an `AdministratorPassword` field.

---

# 🚦 9. Understanding Snaffler Results

Snaffler categorizes findings according to its detection rules.

For example:

```text
{Red}<KeepPassOrKeyInCode|...>
```

indicates a potentially high-value credential/key-related finding.

The example identifies:

```text
\\DC01.inlanefreight.local\ADMIN$\Panther\unattend.xml
```

and:

```text
\\DC01.inlanefreight.local\C$\Windows\Panther\unattend.xml
```

> [!important]  
> `unattend.xml` is particularly interesting during Windows credential hunting because automated Windows installation/deployment files can contain account configuration and potentially sensitive authentication information.

---

# ⚙️ 10. Snaffler Search Refinement

Snaffler can produce a **large amount of output**.

Many matches may be **false positives**, meaning a match looks interesting according to a search rule but doesn't actually contain usable credentials.

Two useful options:

### `-u`

Retrieves users from Active Directory and searches files for references to those users.

```text
-u
```

### `-i` / `-n`

Used to specify which shares should be included in the search.

```text
-i
-n
```

> [!tip]  
> Use these options to reduce the amount of irrelevant output and focus your search on the users/shares that matter.

---

# 🕵️ 11. PowerHuntShares

[PowerHuntShares GitHub repository](https://github.com/NetSPI/PowerHuntShares?utm_source=chatgpt.com)

**PowerHuntShares** is a PowerShell-based tool for discovering and analyzing SMB shares.

Unlike Snaffler, it **doesn't necessarily require execution from a domain-joined machine**.

One major advantage is its reporting functionality:

```text
PowerHuntShares
       │
       ├── 🔎 Discover shares
       ├── 🔐 Analyze permissions
       ├── 📄 Analyze files
       └── 📊 Generate HTML report
```

---

# 💻 12. Running PowerHuntShares

Basic scan:

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public
```

The tool automates several discovery tasks:

```text
1. Determine current computer's domain
2. Enumerate domain computers
3. Check if computers respond to ping
4. Find computers with TCP/445 accessible
5. Enumerate SMB shares
6. Enumerate SMB permissions
7. Identify potentially excessive privileges
8. Identify read/write access
9. Identify high-risk shares
10. Identify common share owners/names/directories
11. Generate last-written / last-accessed timelines
12. Generate HTML summary + CSV files
```

The tool output explicitly notes that scans **can take hours in large environments**.

> [!warning]  
> Large environments can produce a significant amount of data and take substantial time. Consider your scope and prioritize valuable targets.

---

# 🐧 13. Hunting from Linux

When you don't have access to a domain-joined Windows machine, Linux can still be used to remotely search SMB shares.

The main tools covered here are:

```text
🛠️ MANSPIDER
🛠️ NetExec
```

---

# 🕷️ 14. MANSPIDER

[MANSPIDER GitHub repository](https://github.com/blacklanternsecurity/MANSPIDER?utm_source=chatgpt.com)

**MANSPIDER** allows you to search SMB shares remotely from Linux.

It is recommended to use the official Docker container to avoid dependency issues.

---

## 💻 14.1 Basic MANSPIDER Search

Example:

```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider 10.129.234.121 -c 'passw' -u 'mendres' -p 'Inlanefreight2025!'
```

### 🧩 Breakdown

```text
docker run --rm
       ↓
Run temporary Docker container

-v ./manspider:/root/.manspider
       ↓
Persist MANSPIDER data/loot

blacklanternsecurity/manspider
       ↓
MANSPIDER Docker image

10.129.234.121
       ↓
Target SMB server

-c 'passw'
       ↓
Search file contents for "passw"

-u 'mendres'
       ↓
SMB username

-p 'Inlanefreight2025!'
       ↓
SMB password
```

The example output indicates that MANSPIDER:

- Skips files larger than 10 MB
    
- Uses 5 threads
    
- Searches file contents for `passw`
    
- Downloads matching files into its loot directory
    
- Successfully authenticates to the target SMB server.
    

> [!important]  
> MANSPIDER is particularly useful when you want to **search remote SMB shares directly from Linux** rather than first obtaining a domain-joined Windows machine.

---

# 🧰 15. NetExec — Spidering SMB Shares

[NetExec Wiki — SMB Share Spidering](https://www.netexec.wiki/smb-protocol/spidering-shares?utm_source=chatgpt.com)

NetExec can also search network shares.

The relevant functionality is the:

```text
--spider
```

option.

---

## 💻 15.1 Search a Share for `passw`

```bash
nxc smb 10.129.234.121 -u mendres -p 'Inlanefreight2025!' --spider IT --content --pattern "passw"
```

### 🧩 Command Structure

```text
nxc smb
   │
   └── SMB protocol

10.129.234.121
   │
   └── Target server

-u mendres
   │
   └── Username

-p 'Inlanefreight2025!'
   │
   └── Password

--spider IT
   │
   └── Search the IT share

--content
   │
   └── Search file contents

--pattern "passw"
   │
   └── Search for "passw"
```

The example first confirms SMB connectivity and authentication:

```text
SMB  10.129.234.121  445  DC01
[+] inlanefreight.local\mendres:Inlanefreight2025!
[*] Started spidering
[*] Spidering .
```

---

# ⚔️ 16. Windows vs Linux Share Hunting

|Tool|Platform|Domain-Joined Required?|Main Strength|
|---|---|--:|---|
|**Snaffler**|Windows|✅ Yes|Automated share/file discovery from a domain-joined host|
|**PowerHuntShares**|Windows / PowerShell|❌ Not necessarily|Share discovery + permissions + HTML reporting|
|**MANSPIDER**|Linux|❌ No|Remote SMB content searching|
|**NetExec**|Linux / Windows|❌ No|SMB enumeration and share spidering|

> [!tip]  
> **CPTS shortcut:**
> 
> **Windows domain-joined host → Snaffler**
> 
> **Windows PowerShell → PowerHuntShares**
> 
> **Linux → MANSPIDER / NetExec**

---

# 🧠 17. The Complete Credential-Hunting Strategy

```text
             📂 NETWORK SHARES
                    │
                    ▼
          🔎 ENUMERATE SMB SHARES
                    │
                    ▼
        🎯 PRIORITIZE VALUABLE SHARES
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       IT / HR /            Photos /
       Finance /            Media
       Dev / Backup            │
          │                  ⬇️ Lower Priority
          ▼
      📄 Search Files
          │
          ├── Filename
          ├── Extension
          └── Content
                │
                ▼
       🔑 Credentials / Secrets
                │
                ▼
       🧪 Manually Validate
                │
                ▼
       🎯 Further Enumeration
       / Authentication /
       Privilege Escalation
```

---

# 🔥 18. High-Value Search Cheat Sheet

### 🔤 Content Keywords

```text
passw
user
token
key
secret
INLANEFREIGHT\
```

### 📄 File Extensions

```text
.ini
.cfg
.env
.xlsx
.ps1
.bat
```

### 🏷️ Filename Keywords

```text
config
user
passw
cred
initial
```

### 🎯 High-Value Shares

```text
IT
Finance
HR
Development
Infrastructure
Deployment
Backups
```

---

# ⚡ 19. Tool Cheat Sheet

|Tool|Command / Technique|Purpose|
|---|---|---|
|**Snaffler**|`Snaffler.exe -s`|Automatically discover shares and interesting files|
|**Snaffler**|`-u`|Search for references to AD users|
|**Snaffler**|`-i`, `-n`|Refine which shares are searched|
|**PowerHuntShares**|`Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public`|Discover and analyze SMB shares|
|**MANSPIDER**|`docker run ... blacklanternsecurity/manspider ... -c 'passw'`|Search remote SMB shares from Linux|
|**NetExec**|`nxc smb <IP> ... --spider IT --content --pattern "passw"`|Search SMB share contents|

---

# 🧠 20. CPTS Mental Model

> [!important]  
> **Don't think: "Find an SMB share."**
> 
> Think:
> 
> **"Find an accessible SMB share → prioritize valuable shares → search filenames/content → identify secrets → manually validate → use valid credentials for further enumeration."**

```text
📂 SHARE
  ↓
🔐 ACCESS
  ↓
🎯 PRIORITIZE
  ↓
🔎 SEARCH
  ↓
📄 FIND INTERESTING FILE
  ↓
🔑 EXTRACT SECRET
  ↓
🧪 VALIDATE
  ↓
🎯 REUSE / ESCALATE
```

## 🏁 21. Key Takeaways

> [!success]
> 
> - Network shares are often an overlooked source of **credentials and sensitive information**.
>     
> - Start with simple searches before moving to automated tooling.
>     
> - Search **content, filenames, and extensions** rather than relying on obvious password filenames.
>     
> - **IT-related shares** are generally more valuable targets than low-value media shares.
>     
> - **Snaffler** is designed for automated share/file hunting from a domain-joined Windows machine.
>     
> - **PowerHuntShares** performs SMB share discovery and produces useful **HTML/CSV reports**.
>     
> - **MANSPIDER** allows remote SMB content searching from Linux.
>     
> - **NetExec** can spider SMB shares using `--spider`, including searching file contents with `--content --pattern`.
>     
> - Automated tools produce many results, so **manual validation is essential** because false positives are common.
>     
> - The most important skill is not knowing one tool—it is knowing **what to search for and which shares are most likely to contain valuable information**.
>