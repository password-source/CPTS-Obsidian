# 🌐 1. Interacting with Common Services

> [!info]  
> **Core idea:** During a penetration test, you will constantly encounter different network services. To attack or assess a service effectively, you need to understand:
> 
> - 🎯 What the service is used for
>     
> - 🔌 How to connect/interact with it
>     
> - 🛠️ Which tools are available
>     
> - 🔎 What information can be enumerated
>     
> - 🔑 Where credentials or sensitive information may be exposed
>     
> - ⚔️ What attacks become possible after authentication
>     
> 
> **Mental model:**
> 
> ```text
> 🎯 Identify Service
>       ↓
> 🧠 Understand Purpose
>       ↓
> 🔌 Connect
>       ↓
> 🔎 Enumerate
>       ↓
> 🔑 Find Credentials / Sensitive Data
>       ↓
> ⚔️ Exploit / Abuse
>       ↓
> 🧗 Escalate / Pivot
> ```

---

# 📂 2. File Share Services

File-sharing services allow systems and users to transfer or access files.

Common internal file-sharing services include:

```text
SMB
NFS
FTP
TFTP
SFTP
```

Modern environments may also use cloud storage:

```text
Dropbox
Google Drive
OneDrive
SharePoint
AWS S3
Azure Blob Storage
Google Cloud Storage
```

> [!important]  
> **CPTS mindset:** Whenever you discover a file-sharing service, think:
> 
> **"What files can I access, and what credentials or sensitive information might be stored there?"**

A file share may expose:

```text
📄 Documents
🔐 Passwords
🔑 Private keys
⚙️ Configuration files
📝 Scripts
💾 Backups
👤 User information
🗃️ Database files
```

Source:

---

# 🪟 3. SMB — Server Message Block

**SMB** is heavily used in Windows networks for file and resource sharing.

You may interact with SMB through:

```text
🖥️ Windows GUI
⌨️ Windows CMD
💻 PowerShell
🐧 Linux CLI
🛠️ Specialized tools
```

> [!tip]  
> When you see **TCP/445**, immediately think:
> 
> ```text
> SMB
>   ↓
> Shares
>   ↓
> Permissions
>   ↓
> Files
>   ↓
> Credentials / Sensitive Data
> ```
> 
> SMB is therefore both an **access mechanism** and a potentially valuable **credential-hunting location**.

---

# 🖥️ 4. SMB — Windows GUI

A Windows user can access a share using:

```text
\\<SERVER>\<SHARE>\
```

Example:

```text
\\192.168.220.129\Finance\
```

If the share permits anonymous access or your current credentials have sufficient permissions, Windows may open the share directly.

Otherwise, Windows will request credentials.

```text
Anonymous / authorized user
          ↓
     Share opens

No permission
          ↓
  Authentication prompt
```

> [!important]  
> **Authentication and authorization are different:**
> 
> - **Authentication:** Who are you?
>     
> - **Authorization:** What are you allowed to access?
>     
> 
> Successfully authenticating to SMB does **not** mean you can read every share.

Source:

---

# ⌨️ 5. SMB — Windows CMD

## 🔎 5.1 `dir`

`dir` lists files and directories.

```cmd
C:\htb> dir \\192.168.220.129\Finance\
```

Example output:

```text
Volume in drive \\192.168.220.129\Finance has no label.
Volume Serial Number is ABCD-EFAA

Directory of \\192.168.220.129\Finance

02/23/2022  11:35 AM    <DIR>          Contracts
               0 File(s)          4,096 bytes
               1 Dir(s)  15,207,469,056 bytes free
```

### 🧠 Why use it?

Use `dir` to quickly determine:

- Whether the share is accessible
    
- What directories exist
    
- What files are present
    
- Where interesting content may be located
    

---

# 💽 6. `net use` — Map an SMB Share

`net use` connects to a shared resource and can map it to a drive letter.

```cmd
C:\htb> net use n: \\192.168.220.129\Finance
```

Successful result:

```text
The command completed successfully.
```

The share is now available as:

```text
N:\
```

### 🧠 Mental Model

```text
\\SERVER\SHARE
       ↓
   net use
       ↓
      N:\
       ↓
Use it like a local drive
```

---

# 🔐 7. `net use` with Credentials

You can authenticate to a share using a username and password:

```cmd
C:\htb> net use n: \\192.168.220.129\Finance /user:plaintext Password123
```

> [!warning]  
> The password is supplied directly on the command line in this example. In real environments, avoid exposing credentials in shell history/process listings when possible.

---

# 🔎 8. Searching a Large SMB Share

Once mapped to `N:`, you can use normal Windows commands against it.

For example, count all files:

```cmd
C:\htb> dir n: /a-d /s /b | find /c ":\"
```

Example:

```text
29302
```

This means the share contains **29,302 files**.

### 🧠 Command Breakdown

|Option|Meaning|
|---|---|
|`dir`|Application|
|`n:`|Directory or drive to search|
|`/a-d`|`/a` is the attribute and `-d` means not directories|
|`/s`|Displays files in a specified directory and all subdirectories|
|`/b`|Uses bare format (no heading information or summary)|

The output is piped to:

```cmd
find /c ":\"
```

which counts the resulting file paths.

Source:

---

# 🎯 9. Search SMB by Filename

Instead of manually browsing thousands of files, search for interesting filenames.

Useful keywords include:

```text
cred
password
users
secrets
key
```

Also consider source-code extensions:

```text
.cs
.c
.go
.java
.php
.asp
.aspx
.html
```

Examples:

```cmd
C:\htb> dir n:\*cred* /s /b
```

Result:

```text
n:\Contracts\private\credentials.txt
```

Another:

```cmd
C:\htb> dir n:\*secret* /s /b
```

Result:

```text
n:\Contracts\private\secret.txt
```

> [!tip]  
> Searching by **filename** is often much faster than opening files manually.
> 
> Think:
> 
> ```text
> Huge Share
>     ↓
> Keyword Search
>     ↓
> Interesting Files
>     ↓
> Inspect Content
> ```

---

# 🔍 10. `findstr` — Search File Contents

`findstr` searches inside text files.

Example:

```cmd
c:\htb>findstr /s /i cred n:\*.*
```

Example output:

```text
n:\Contracts\private\secret.txt:file with all credentials
n:\Contracts\private\credentials.txt:admin:SecureCredentials!
```

### Important Options

```text
/s  → Search subdirectories
/i  → Case-insensitive search
```

> [!important]  
> `dir` searches **filenames**.
> 
> `findstr` searches **file contents**.
> 
> ```text
> dir
>  ↓
> "Which files look interesting?"
> 
> findstr
>  ↓
> "Which files actually contain interesting text?"
> ```

Source:

---

# 💙 11. SMB — PowerShell

PowerShell provides more powerful scripting and automation capabilities than traditional CMD.

You can use:

```powershell
Get-ChildItem
```

or its short alias:

```powershell
gci
```

to enumerate files.

Example:

```powershell
PS C:\htb> Get-ChildItem \\192.168.220.129\Finance\
```

---

# 💽 12. `New-PSDrive` — Map SMB in PowerShell

Instead of `net use`, PowerShell can create a mapped drive:

```powershell
PS C:\htb> New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem"
```

Example result:

```text
Name Used (GB) Free (GB) Provider      Root
---- --------- --------- --------      ----
N                         FileSystem    \\192.168.220.129\Finance
```

Now:

```powershell
N:
```

moves into the mapped share.

---

# 🔐 13. PowerShell — PSCredential

PowerShell uses a `PSCredential` object to represent credentials.

Example:

```powershell
$username = 'plaintext'
$password = 'Password123'
$secpassword = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secpassword
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem" -Credential $cred
```

### 🧠 Flow

```text
Username + Password
        ↓
Convert password
        ↓
SecureString
        ↓
PSCredential
        ↓
New-PSDrive
        ↓
Authenticated SMB Share
```

---

# 📊 14. PowerShell — Count Files

Once inside `N:`:

```powershell
PS N:\> (Get-ChildItem -File -Recurse | Measure-Object).Count
```

Result:

```text
29302
```

This recursively counts files.

---

# 🎯 15. PowerShell — Search by Filename

Use `-Include` to search for matching files:

```powershell
PS C:\htb> Get-ChildItem -Recurse -Path N:\ -Include *cred* -File
```

Example:

```text
Directory: N:\Contracts\private

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2/23/2022   4:36 PM             25 credentials.txt
```

---

# 🔎 16. `Select-String` — Search File Contents

PowerShell's `Select-String` is similar to:

```text
grep     → Linux
findstr  → Windows CMD
```

Example:

```powershell
PS C:\htb> Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List
```

Example output:

```text
N:\Contracts\private\secret.txt:1:file with all credentials
N:\Contracts\private\credentials.txt:1:admin:SecureCredentials!
```

> [!tip]  
> For CPTS, remember this comparison:
> 
> ```text
> Windows CMD
> ├── dir
> └── findstr
> 
> PowerShell
> ├── Get-ChildItem / gci
> └── Select-String
> 
> Linux
> ├── find
> └── grep
> ```

Source:

---

# 🐧 17. SMB — Linux

Linux can interact with SMB shares hosted by:

```text
Windows
Samba
```

A useful approach is to **mount the SMB share as a local directory**.

```text
SMB Server
     ↓
 CIFS mount
     ↓
/mnt/Finance
     ↓
find / grep / strings / other Linux tools
```

---

# 📂 18. Mount SMB with CIFS

Create a mount point:

```bash
sudo mkdir /mnt/Finance
```

Mount the share:

```bash
sudo mount -t cifs -o username=plaintext,password=Password123,domain=. //192.168.220.129/Finance /mnt/Finance
```

Now the remote SMB share behaves like a local directory:

```text
/mnt/Finance/
```

> [!important]  
> `cifs` is the Linux filesystem mechanism commonly used to mount SMB/CIFS shares.

---

# 🔐 19. SMB Credential File

Instead of putting the password directly into the mount command, use a credential file:

```bash
mount -t cifs //192.168.220.129/Finance /mnt/Finance -o credentials=/path/credentialfile
```

Credential file format:

```text
username=plaintext
password=Password123
domain=.
```

> [!warning]  
> Credential files contain sensitive authentication material. They should be protected with appropriate filesystem permissions.

---

# 📦 20. `cifs-utils`

Linux systems may require the `cifs-utils` package for SMB mounting.

Install it with:

```bash
sudo apt install cifs-utils
```

---

# 🔎 21. Search a Mounted SMB Share — Linux

Once mounted, use normal Linux tools.

### Search filenames

```bash
find /mnt/Finance/ -name *cred*
```

Example:

```text
/mnt/Finance/Contracts/private/credentials.txt
```

### Search file contents

```bash
grep -rn /mnt/Finance/ -ie cred
```

Example:

```text
/mnt/Finance/Contracts/private/credentials.txt:1:admin:SecureCredentials!
/mnt/Finance/Contracts/private/secret.txt:1:file with all credentials
```

> [!important]  
> Mounting SMB is powerful because you can then use the entire Linux filesystem/search ecosystem against the remote share:
> 
> ```text
> SMB
> ↓
> mount
> ↓
> Linux directory
> ↓
> find
> grep
> strings
> file
> awk
> sed
> scripts
> ```

Source:

---

# 🗂️ 22. Other File-Sharing Services

Other services may provide file access:

```text
FTP
TFTP
NFS
```

The exact connection method differs, but the **post-access methodology remains similar**:

```text
Connect
  ↓
Enumerate
  ↓
Access files
  ↓
Search files
  ↓
Identify sensitive information
  ↓
Reuse credentials / continue attack
```

> [!tip]  
> Don't become overly dependent on one tool.
> 
> Learn the **service concept** first, then learn the tools that interact with it.

---

# 📧 23. Email Services

Email commonly involves separate protocols for sending and receiving.

### Sending

```text
SMTP
```

### Receiving

```text
POP3
IMAP
```

Secure variants may use:

```text
SMTPS
IMAPS
```

TLS can also be negotiated using **STARTTLS**.

---

# 📨 24. Email Client — Evolution

Evolution can be used to interact with mail servers.

Install:

```bash
sudo apt-get install evolution
```

If Evolution produces:

```text
bwrap: Can't create file at ...
```

the source recommends:

```bash
export WEBKIT_FORCE_SANDBOX=0 && evolution
```

When configuring a mail server, you may specify either:

```text
Domain name
OR
IP address
```

If the server uses SMTPS/IMAPS, select the appropriate encryption method.

> [!important]  
> Email enumeration can become valuable when you obtain valid credentials because mailboxes may contain:
> 
> - Password reset emails
>     
> - Internal usernames
>     
> - Sensitive documents
>     
> - Credentials
>     
> - Network information
>     
> - Internal communications
>     

Source:

---

# 🗄️ 25. Databases

Enterprise environments commonly use databases to store information.

Common categories include:

```text
Relational SQL
NoSQL
Hierarchical
```

This section focuses on relational databases, particularly:

```text
MySQL
MSSQL
```

There are three common ways to interact with databases:

|||
|---|---|
|`1.`|Command Line Utilities (`mysql` or `sqsh`)|
|`2.`|Programming Languages|
|`3.`|A GUI application to interact with databases such as HeidiSQL, MySQL Workbench, or SQL Server Management Studio.|

Source:

---

# 🟥 26. MSSQL — Linux

On Linux, you can use:

```text
sqsh
```

or other MSSQL-compatible tools.

Example:

```bash
sqsh -S 10.129.20.13 -U username -P Password123
```

Where:

```text
-S → SQL Server
-U → Username
-P → Password
```

> [!warning]  
> The password in this example is placed directly on the command line. Treat this as an instructional example and avoid exposing real credentials unnecessarily.

---

# 🪟 27. MSSQL — Windows

Windows can use:

```text
sqlcmd
```

Example:

```cmd
C:\htb> sqlcmd -S 10.129.20.13 -U username -P Password123
```

`sqlcmd` can be used to execute:

```text
Transact-SQL statements
System procedures
Script files
```

---

# 🟦 28. MySQL

Linux:

```bash
mysql -u username -pPassword123 -h 10.129.20.13
```

Windows:

```cmd
mysql.exe -u username -pPassword123 -h 10.129.20.13
```

### 🧠 Important Syntax Detail

```text
-pPassword
```

has **no space** between `-p` and the password in this example.

---

# 🖥️ 29. Database GUI Tools

Databases can also be accessed through graphical tools.

Examples:

```text
MySQL
 └── MySQL Workbench

MSSQL
 └── SQL Server Management Studio (SSMS)

Multiple DB engines
 └── DBeaver
```

DBeaver supports multiple database engines and platforms.

Install a downloaded Debian package:

```bash
sudo dpkg -i dbeaver-<version>.deb
```

Start it:

```bash
dbeaver &
```

> [!tip]  
> A GUI can be useful when:
> 
> - You have many databases/tables to inspect
>     
> - You need a visual query interface
>     
> - You're dealing with several database technologies
>     
> 
> But **CLI tools are usually faster and more convenient during a timed assessment**.

---

# 🔎 30. Database Enumeration After Authentication

Once you successfully connect to a database:

```text
🎯 Database Access
       ↓
🔎 Enumerate Databases
       ↓
📋 Enumerate Tables
       ↓
🔍 Search Sensitive Columns/Data
       ↓
🔑 Find Credentials
       ↓
👤 Identify Useful Accounts
       ↓
⚔️ Continue Attack
```

Potentially interesting information includes:

```text
username
password
hash
token
API key
configuration
internal hostnames
application secrets
```

If sufficient privileges exist, database access may potentially allow command execution under the database service account, depending on the database configuration and attack surface.

Source:

---

# 🛠️ 31. Tools to Interact with Common Services

|**SMBFTPEmailDatabases**||||
|---|---|---|---|
|[smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)|[ftp](https://linux.die.net/man/1/ftp)|[Thunderbird](https://www.thunderbird.net/en-US/)|[mssql-cli](https://github.com/dbcli/mssql-cli)|
|[CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec)|[lftp](https://lftp.yar.ru/)|[Claws](https://www.claws-mail.org/)|[mycli](https://github.com/dbcli/mycli)|
|[SMBMap](https://github.com/ShawnDEvans/smbmap)|[ncftp](https://www.ncftp.com/)|[Geary](https://wiki.gnome.org/Apps/Geary)|[mssqlclient.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/mssqlclient.py)|
|[Impacket](https://github.com/SecureAuthCorp/impacket)|[filezilla](https://filezilla-project.org/)|[MailSpring](https://getmailspring.com/)|[dbeaver](https://github.com/dbeaver/dbeaver)|
|[psexec.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/psexec.py)|[crossftp](http://www.crossftp.com/)|[mutt](http://www.mutt.org/)|[MySQL Workbench](https://dev.mysql.com/downloads/workbench/)|
|[smbexec.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbexec.py)||[mailutils](https://mailutils.org/)|[SQL Server Management Studio or SSMS](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms)|
|||[sendEmail](https://github.com/mogaal/sendemail)||
|||[swaks](http://www.jetmore.org/john/code/swaks/)||
|||[sendmail](https://en.wikipedia.org/wiki/Sendmail)||

> [!important]  
> **Tool selection should follow the service:**
> 
> ```text
> SMB
> ├── smbclient
> ├── SMBMap
> ├── CrackMapExec
> └── Impacket
> 
> FTP
> ├── ftp
> ├── lftp
> └── ncftp
> 
> Email
> ├── Thunderbird
> ├── Claws
> ├── Geary
> ├── mutt
> ├── swaks
> └── sendmail
> 
> Databases
> ├── mssql-cli
> ├── mssqlclient.py
> ├── mycli
> ├── DBeaver
> ├── MySQL Workbench
> └── SSMS
> ```

Source:

---

# 🧭 32. Service Interaction Methodology

Whenever you encounter an unfamiliar service, use this methodology:

```text
                    🎯 SERVICE DISCOVERED
                           │
                           ▼
                    1️⃣ IDENTIFY IT
                           │
                           ▼
                    2️⃣ KNOW ITS PURPOSE
                           │
                           ▼
                    3️⃣ FIND CLIENT / TOOL
                           │
                           ▼
                    4️⃣ TEST AUTHENTICATION
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
              Anonymous          Credentials
                  │                 │
                  └────────┬────────┘
                           ▼
                    5️⃣ ENUMERATE
                           │
                           ▼
                    6️⃣ ACCESS DATA
                           │
                           ▼
                    7️⃣ SEARCH FOR
                       CREDENTIALS
                           │
                           ▼
                    8️⃣ IDENTIFY
                    PRIVILEGES
                           │
                           ▼
                    9️⃣ ATTACK / PIVOT
```

---

# 🧠 33. “I Found This Service — What Do I Do?”

## 📂 SMB

```text
445/TCP
  ↓
SMB
  ↓
Enumerate shares
  ↓
Anonymous?
  ↓
Valid credentials?
  ↓
Read/write permissions?
  ↓
Download files
  ↓
Search credentials/secrets
  ↓
Reuse discovered credentials
```

### Key tools

```text
smbclient
SMBMap
NetExec / CrackMapExec
Impacket
```

---

## 🗄️ MSSQL

```text
1433/TCP
  ↓
MSSQL
  ↓
Do I have credentials?
  ↓
Connect
  ↓
Enumerate databases
  ↓
Enumerate tables
  ↓
Search sensitive data
  ↓
Check privileges
  ↓
Potential command execution / pivot
```

### Key tools

```text
sqsh
sqlcmd
mssqlclient.py
DBeaver
SSMS
```

---

## 🗃️ MySQL

```text
3306/TCP
  ↓
MySQL
  ↓
Credentials?
  ↓
mysql client
  ↓
Enumerate databases
  ↓
Tables
  ↓
Sensitive information
```

---

## 📧 Email

```text
SMTP
 ↓
Sending

POP3 / IMAP
 ↓
Receiving
```

With credentials:

```text
Authenticate
 ↓
Enumerate mailbox
 ↓
Search sensitive messages
 ↓
Credentials / internal information
 ↓
Pivot
```

---

## 📁 FTP / NFS / TFTP

```text
File Service
    ↓
Identify access method
    ↓
Anonymous?
    ↓
Authenticated?
    ↓
Read / write?
    ↓
Enumerate files
    ↓
Search sensitive information
```

---

# 🚨 34. General Troubleshooting

When a connection fails, don't immediately assume the credentials are wrong.

Possible causes include:

```text
🔐 Authentication
🛡️ Privileges
🌐 Network connectivity
🔥 Firewall rules
🔌 Protocol support
🖥️ OS/version differences
🛠️ Client/tool compatibility
```

> [!important]  
> A failed connection is **information**.
> 
> Break the problem into layers:
> 
> ```text
> 1. 🌐 Can I reach the host?
>          ↓
> 2. 🔌 Is the port/service available?
>          ↓
> 3. 🔐 Does authentication work?
>          ↓
> 4. 👤 Does this account have permission?
>          ↓
> 5. 📂 Does the resource exist?
>          ↓
> 6. 🛠️ Does my client support the protocol?
> ```

Source:

---

# ⚡ 35. CPTS Quick Cheat Sheet

|Finding|First Thought|Next Action|
|---|---|---|
|`445` / SMB|Shares|Enumerate shares and permissions|
|SMB share|Files|Search filenames + contents|
|Anonymous SMB|Misconfiguration|Enumerate immediately|
|SMB credentials|Authenticated access|Check accessible shares|
|Huge SMB share|Too many files|Automate filename/content searches|
|`1433` / MSSQL|Database|Connect and enumerate|
|MSSQL credentials|Database access|Enumerate DBs/tables|
|MySQL|Database|Connect and enumerate|
|SMTP|Email sending|Enumerate service/config|
|IMAP/POP3|Email retrieval|Authenticate and inspect mailbox|
|FTP|File transfer|Test anonymous/authenticated access|
|NFS|Remote filesystem|Enumerate exports and access|
|Unknown service|Research required|Identify protocol/client first|

---

# 🧠 36. The Three-Layer Enumeration Model

For almost every service, think in three layers:

```text
                 🌐 SERVICE
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      🔐 ACCESS             📂 DATA
          │                   │
          ▼                   ▼
   Who can connect?     What can I read?
   How do I auth?      What can I write?
   Anonymous?          What sensitive data?
          │                   │
          └─────────┬─────────┘
                    ▼
                 ⚔️ IMPACT
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      Credentials          Execution
          │                   │
          ▼                   ▼
      New Access          New Access
```

> [!tip]  
> This model works beyond SMB.
> 
> For **any service**, ask:
> 
> **1. Can I connect?**
> 
> **2. What can I access?**
> 
> **3. What can I do with what I found?**

---

# 🎯 37. CPTS Service Interaction Mental Model

```text
🔎 ENUMERATE
    ↓
"What service is running?"
    ↓
🧠 UNDERSTAND
    ↓
"What is this service designed to do?"
    ↓
🛠️ SELECT TOOL
    ↓
"How do I interact with it?"
    ↓
🔐 AUTHENTICATE
    ↓
"Anonymous or credentials?"
    ↓
📂 ENUMERATE DATA
    ↓
"What can I read/write?"
    ↓
🔑 HUNT
    ↓
"Are credentials/secrets exposed?"
    ↓
⚔️ ABUSE
    ↓
"Can this access lead somewhere else?"
    ↓
🔄 PIVOT
```

> [!success]
> 
> ### 🏆 Key Takeaways
> 
> - **SMB = think shares, permissions, files, credentials.**
>     
> - `dir` → enumerate/search filenames on Windows.
>     
> - `findstr` → search file contents on Windows CMD.
>     
> - `Get-ChildItem` / `gci` → PowerShell file enumeration.
>     
> - `Select-String` → PowerShell content searching.
>     
> - `mount -t cifs` → mount SMB on Linux.
>     
> - `find` → search filenames.
>     
> - `grep` → search file contents.
>     
> - **MSSQL/MySQL = authenticate → enumerate databases → tables → sensitive data.**
>     
> - **Email = SMTP for sending, POP3/IMAP for receiving.**
>     
> - **Unknown service = identify its protocol and learn how to interact with it before attacking it.**
>     
> - When troubleshooting, separate **network → service → authentication → authorization → protocol/client** problems.
>     
> 
> **The CPTS rule to remember:**
> 
> ```text
> DON'T JUST ASK:
> "How do I connect?"
> 
> ASK:
> "How do I connect?"
>       ↓
> "What can I access?"
>       ↓
> "What information can I extract?"
>       ↓
> "What credentials can I discover?"
>       ↓
> "What can those credentials unlock?"
> ```