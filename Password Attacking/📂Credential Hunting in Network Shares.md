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

# 🔌 2. Manual SMB Hunting — Download First, Search Locally

> [!important]  
> **Before reaching for specialized tools, remember that you can often connect directly to an SMB share, download its contents, and perform local searches with standard Linux tools.**
> 
> This is especially useful when:
> 
> - You have valid SMB credentials.
>     
> - You can access a specific share.
>     
> - You want to inspect **many files repeatedly**.
>     
> - You want to use powerful local tools such as `grep`, `find`, `file`, `strings`, and `ripgrep`.
>     

The basic workflow is:

```text
             🌐 SMB SERVER
                  │
                  ▼
          🔐 Authenticate
                  │
                  ▼
            📂 SMB SHARE
                  │
                  ▼
          📥 Download Files
             mget *
                  │
                  ▼
          💻 Search Locally
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      grep      find      strings
        │         │         │
        └─────────┼─────────┘
                  ▼
          🔑 Find Credentials
                  │
                  ▼
          🧪 Manually Validate
```

---

## 💻 2.1 Connect to an SMB Share with `smbclient`

From Linux:

```bash
smbclient //10.129.234.121/IT -U mendres
```

You will be prompted for the password.

You can also specify the domain:

```bash
smbclient //10.129.234.121/IT -U 'INLANEFREIGHT.LOCAL/mendres'
```

Once connected, you receive an SMB prompt:

```text
smb: \>
```

Useful commands inside `smbclient`:

```text
ls
pwd
cd <directory>
lcd <local-directory>
get <file>
mget <files>
recurse ON
prompt OFF
```

### 🧠 Important Commands

|Command|Purpose|
|---|---|
|`ls`|List files/directories in the current SMB directory|
|`pwd`|Show the current remote SMB directory|
|`cd <directory>`|Change the remote SMB directory|
|`lcd <directory>`|Change the local download directory|
|`get <file>`|Download one file|
|`mget <files>`|Download multiple files|
|`recurse ON`|Enable recursive directory traversal|
|`prompt OFF`|Disable confirmation prompts for multiple downloads|

---

## 📥 2.2 Download Everything with `mget *`

If you want to pull the accessible contents of the current share:

```text
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

### 🧠 What Happens?

```text
recurse ON
    ↓
Allow smbclient to traverse directories

prompt OFF
    ↓
Don't ask "download this file?" for every file

mget *
    ↓
Download matching files/directories
```

This gives you a local copy that you can search using normal Linux utilities.

> [!warning]  
> `mget *` can download a **large amount of data**. In a real assessment, consider the share size, scope, available disk space, and time before recursively downloading everything.
> 
> If the share is huge, it is often better to identify valuable directories/files first.

---

## 📂 2.3 Download to a Dedicated Directory

Create a local directory first:

```bash
mkdir -p ~/smb/IT
cd ~/smb/IT
```

Then connect:

```bash
smbclient //10.129.234.121/IT -U mendres
```

Inside the SMB session:

```text
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

Now the downloaded files are available locally:

```text
~/smb/IT/
├── scripts/
├── configs/
├── backups/
├── documents/
└── ...
```

---

# 🔎 3. Search the Downloaded Share Locally

Once the files are downloaded, you no longer need to repeatedly query SMB for every search.

You can use Linux tools against the local copy.

## 🔤 3.1 Search File Contents with `grep`

Basic recursive search:

```bash
grep -Rni "password" ~/smb/IT/
```

Useful options:

```text
-R    Recursive
-n    Show line numbers
-i    Case-insensitive
```

Search multiple credential-related keywords:

```bash
grep -RniE 'password|passwd|passw|username|user|credential|cred|token|secret|apikey|api_key|private_key' ~/smb/IT/
```

This is often much faster than manually opening every downloaded file.

---

## 🎯 3.2 Search for Domain User References

For a Windows domain:

```bash
grep -Rni 'INLANEFREIGHT\\' ~/smb/IT/
```

You may discover references such as:

```text
INLANEFREIGHT\administrator
INLANEFREIGHT\jbader
INLANEFREIGHT\mendres
```

> [!tip]  
> Adapt your searches to the target environment. Domain names, usernames, application names, languages, project names, and company-specific terminology can all become useful search terms.

---

# 📄 4. Search by File Type

Content searches become more effective when combined with file-type filtering.

Find configuration files:

```bash
find ~/smb/IT -type f \( -iname "*.ini" -o -iname "*.cfg" -o -iname "*.conf" -o -iname "*.config" -o -iname "*.env" \)
```

Find PowerShell and batch scripts:

```bash
find ~/smb/IT -type f \( -iname "*.ps1" -o -iname "*.bat" -o -iname "*.cmd" \)
```

Find spreadsheets and documents:

```bash
find ~/smb/IT -type f \( -iname "*.xlsx" -o -iname "*.xls" -o -iname "*.docx" -o -iname "*.txt" \)
```

Then inspect particularly interesting files.

---

# 🧪 5. Useful Local Search Techniques

## 🔍 5.1 Find Files by Name

```bash
find ~/smb/IT -type f \( -iname "*password*" -o -iname "*passwd*" -o -iname "*credential*" -o -iname "*cred*" -o -iname "*config*" -o -iname "*secret*" \)
```

---

## 🧬 5.2 Search for Secrets with `grep`

```bash
grep -RniE 'passw|passwd|credential|token|secret|api[_-]?key|username|login' ~/smb/IT/
```

---

## 📝 5.3 Search Text Files While Ignoring Binary Files

```bash
grep -RniI -E 'passw|passwd|credential|token|secret' ~/smb/IT/
```

`-I` tells `grep` to ignore binary files.

---

## 🧱 5.4 Inspect Unknown Files with `file`

When you find something interesting:

```bash
file ~/smb/IT/path/to/file
```

This helps determine whether the file is:

```text
📄 Text
📦 Archive
📊 Spreadsheet
🖼️ Image
💾 Database
⚙️ Executable
```

---

## 🔤 5.5 Search Strings in Binary Files

Some potentially useful information may be embedded inside binaries or unusual file formats.

```bash
strings <file> | grep -iE 'password|passwd|token|secret|user'
```

> [!important]  
> `grep` is excellent for text-based content, but not every interesting file is plain text. `file` and `strings` can help identify or extract readable information from less obvious files.

---

# 🧠 6. Manual SMB Hunting Mental Model

The key idea is:

```text
🔐 AUTHENTICATE
      ↓
📂 ACCESS SHARE
      ↓
📥 DOWNLOAD
      ↓
🔎 SEARCH LOCALLY
      ↓
📄 IDENTIFY INTERESTING FILE
      ↓
🔑 EXTRACT SECRET
      ↓
🧪 VALIDATE
      ↓
🎯 REUSE / ESCALATE
```

This gives you an important alternative to immediately using automated tools.

> [!success]  
> **CPTS mindset:** Specialized tools automate the process, but you should understand the underlying workflow first.
> 
> **SMB → files → search → credentials → validate → access**

---

# 🧠 7. What Makes Network Shares Valuable?

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

# 🔎 8. Common Credential Patterns

Before using specialized tools, start by understanding what you're searching for.

## 📝 8.1 Useful Keywords

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

## 📄 8.2 Interesting File Extensions

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

## 🏷️ 8.3 Interesting File Names

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

# 🌐 9. Domain-Specific Searches

When targeting a Windows domain, domain names can be useful search terms.

For example:

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

# 🎯 10. Choose Shares Strategically

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

# 💻 11. Start Simple Before Automating

Before using specialized tools, basic command-line searches can sometimes be enough.

A general Windows approach is:

```powershell
Get-ChildItem -Recurse -Include *.ext \\Server\Share | Select-String -Pattern ...
```

From Linux, you can use the manual **SMB → download → grep** approach:

```bash
smbclient //SERVER/SHARE -U USER
```

Then:

```text
recurse ON
prompt OFF
mget *
```

Followed by:

```bash
grep -RniE 'password|passwd|user|credential|token|secret' ./SHARE/
```

### 🧠 Methodology

```text
📂 Identify interesting share
        ↓
🔐 Authenticate
        ↓
📥 Download accessible content
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

# 🪟 12. Hunting from Windows

The primary tools covered here are:

```text
🛠️ Snaffler
🛠️ PowerHuntShares
```

---

# 🔍 13. Snaffler

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

## 🧠 13.1 What Snaffler Does

When no specific computer or path targets are supplied, Snaffler can perform discovery automatically:

```text
Snaffler
   │
   ├── 🔎 DFS Discovery
   ├── 🏢 Domain Computer Discovery
   ├── 📂 Share Discovery
   ├── 🔐 Permission Checks
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

# 🚦 14. Understanding Snaffler Results

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

# ⚙️ 15. Snaffler Search Refinement

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

# 🕵️ 16. PowerHuntShares

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

# 💻 17. Running PowerHuntShares

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

# 🐧 18. Hunting from Linux

When you don't have access to a domain-joined Windows machine, Linux can still be used to remotely search SMB shares.

The main tools covered here are:

```text
🛠️ smbclient
🛠️ MANSPIDER
🛠️ NetExec
```

Think of them as different levels of automation:

```text
                 Linux SMB Hunting
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      smbclient      MANSPIDER      NetExec
          │             │             │
      Manual          Automated     Automated
      download        content       SMB
      + grep          search        spidering
```

---

# 🕷️ 19. MANSPIDER

**MANSPIDER** allows you to search SMB shares remotely from Linux.

It is recommended to use the official Docker container to avoid dependency issues.

---

## 💻 19.1 Basic MANSPIDER Search

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
    
- Successfully authenticates to the target SMB server
    

> [!important]  
> MANSPIDER is particularly useful when you want to **search remote SMB shares directly from Linux** rather than first obtaining a domain-joined Windows machine.

---

# 🧰 20. NetExec — Spidering SMB Shares

NetExec can also search network shares.

The relevant functionality is the:

```text
--spider
```

option.

---

## 💻 20.1 Search a Share for `passw`

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

# ⚔️ 21. Manual vs Automated Share Hunting

|Method|Platform|Domain-Joined Required?|Main Strength|
|---|---|---|---|
|**smbclient + grep**|Linux|❌ No|Manual download + powerful local searching|
|**Snaffler**|Windows|✅ Yes|Automated share/file discovery from a domain-joined host|
|**PowerHuntShares**|Windows / PowerShell|❌ Not necessarily|Share discovery + permissions + HTML reporting|
|**MANSPIDER**|Linux|❌ No|Remote SMB content searching|
|**NetExec**|Linux / Windows|❌ No|SMB enumeration and share spidering|

> [!tip]  
> **CPTS shortcut:**
> 
> **Need manual control → `smbclient` + `grep`**
> 
> **Windows domain-joined host → Snaffler**
> 
> **Windows PowerShell → PowerHuntShares**
> 
> **Linux automated search → MANSPIDER / NetExec**

---

# 🧠 22. The Complete Credential-Hunting Strategy

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
      🔐 AUTHENTICATE
          │
          ▼
      📥 DOWNLOAD
          │
          ▼
      📄 SEARCH FILES
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

# 🔥 23. High-Value Search Cheat Sheet

### 🔤 Content Keywords

```text
passw
passwd
user
username
credential
cred
token
key
secret
password
INLANEFREIGHT\
```

### 📄 File Extensions

```text
.ini
.cfg
.conf
.config
.env
.xlsx
.ps1
.bat
.cmd
.txt
.xml
```

### 🏷️ Filename Keywords

```text
config
user
passw
cred
credential
secret
initial
backup
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

# ⚡ 24. Command Cheat Sheet

|Tool Command / Technique Purpose|||
|---|---|---|
|**Snaffler**|`Snaffler.exe -s`|Automatically discover shares and interesting files|
|**Snaffler**|`-u`|Search for references to AD users|
|**Snaffler**|`-i`, `-n`|Refine which shares are searched|
|**PowerHuntShares**|`Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public`|Discover and analyze SMB shares|
|**MANSPIDER**|`docker run ... blacklanternsecurity/manspider ... -c 'passw'`|Search remote SMB shares from Linux|
|**NetExec**|`nxc smb <IP> ... --spider IT --content --pattern "passw"`|Search SMB share contents|

### 🐧 Manual Linux Workflow

```bash
# Connect
smbclient //10.129.234.121/IT -U mendres

# Inside smbclient
recurse ON
prompt OFF
mget *
```

Then search locally:

```bash
grep -RniE 'password|passwd|passw|username|credential|cred|token|secret' ~/smb/IT/
```

Find interesting files:

```bash
find ~/smb/IT -type f \( -iname "*password*" -o -iname "*cred*" -o -iname "*config*" -o -iname "*secret*" \)
```

Inspect unknown files:

```bash
file <file>
strings <file> | grep -iE 'password|passwd|token|secret|user'
```

---

# 🧠 25. CPTS Mental Model

> [!important]  
> **Don't think: "Find an SMB share."**
> 
> Think:
> 
> **"Find an accessible SMB share → prioritize valuable shares → download or spider its contents → search filenames/content → identify secrets → manually validate → use valid credentials for further enumeration."**

```text
📂 SHARE
  ↓
🔐 ACCESS
  ↓
🎯 PRIORITIZE
  ↓
📥 DOWNLOAD / SPIDER
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

---

# 🏁 26. Key Takeaways

> [!success]
> 
> - Network shares are often an overlooked source of **credentials and sensitive information**.
>     
> - You don't always need specialized tooling: **`smbclient` → `mget *` → local `grep`** is a powerful manual workflow.
>     
> - Use `recurse ON` + `prompt OFF` + `mget *` when you intentionally want to recursively download accessible share contents.
>     
> - Don't blindly download huge shares; consider scope, disk space, and time.
>     
> - Search **content, filenames, and extensions** rather than relying on obvious password filenames.
>     
> - Combine keywords such as `passw`, `cred`, `token`, `secret`, and domain-qualified usernames.
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
> - **CPTS core workflow:**
>     
> 
> ```text
> SMB ACCESS
>     ↓
> SHARE ENUMERATION
>     ↓
> PRIORITIZE
>     ↓
> DOWNLOAD / SPIDER
>     ↓
> SEARCH
>     ↓
> FIND SECRET
>     ↓
> VALIDATE
>     ↓
> FURTHER ACCESS
> ```