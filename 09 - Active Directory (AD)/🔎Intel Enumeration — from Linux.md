# 🔎 1. Credentialed Enumeration — from Linux

> [!info] 🎯 Objective  
> After gaining a foothold in the domain, we can use **low-privilege domain credentials** to significantly expand our visibility.
> 
> We want to enumerate:
> 
> - 👤 Domain users and their attributes
>     
> - 💻 Domain computers
>     
> - 👥 Group membership
>     
> - 📜 Group Policy Objects (GPOs)
>     
> - 🔐 Permissions and ACLs
>     
> - 🤝 Domain trusts
>     
> - 🪑 User sessions
>     
> - 📂 SMB shares and their contents
>     
> - 🧭 Potential attack paths
>     
> 
> Most credentialed enumeration tools require **some form of valid domain access**. At minimum, this may mean:
> 
> - Cleartext domain credentials
>     
> - NTLM password hash
>     
> - SYSTEM access on a domain-joined host
>     
> 
> In this section, the example credentials are:
> 
> ```text
> Domain:   INLANEFREIGHT.LOCAL
> User:     forend
> Password: Klmcargo2
> DC:       172.16.5.5
> ```
> 
> The key idea is:
> 
> **A low-privilege domain account can often see far more of AD than an unauthenticated attacker.**

---

# ⚡ 2. Command Cheat Sheet

## 🛠️ CrackMapExec

```bash
sudo crackmapexec smb -h
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --users
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --groups
sudo crackmapexec smb 172.16.5.130 -u forend -p Klmcargo2 --loggedon-users
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --shares
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M spider_plus --share 'Department Shares'
head -n 10 /tmp/cme_spider_plus/172.16.5.5.json
```

## 📂 SMBMap

```bash
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5 -R 'Department Shares' --dir-only
```

## 🛰️ rpcclient

```bash
rpcclient -U "" -N 172.16.5.5
queryuser 0x457
enumdomusers
```

## 🧰 Impacket

```bash
psexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.125
wmiexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.5
```

## 🌐 Windapsearch

```bash
windapsearch.py -h
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 --da
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 -PU
```

## 🩸 BloodHound.py

```bash
bloodhound-python -h
sudo bloodhound-python -u 'forend' -p 'Klmcargo2' -ns 172.16.5.5 -d inlanefreight.local -c all
ls
sudo neo4j start
zip -r ilfreight_bh.zip *.json
```

---

# 🧠 3. Overall Enumeration Mental Model

```text
                 🔑 LOW-PRIV DOMAIN CREDENTIALS
                              │
                              ▼
                     🏢 DOMAIN CONTROLLER
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
          👤 USERS         👥 GROUPS        💻 HOSTS
              │               │                │
              │               │                ├──► Logged-on users
              │               │                └──► SMB shares
              │               │
              └───────┬───────┘
                      ▼
                 🔐 PRIVILEGES
                      │
             ┌────────┼─────────┐
             ▼        ▼         ▼
            ACLs     GPOs     Trusts
             │        │         │
             └────────┼─────────┘
                      ▼
                🩸 BLOODHOUND
                      │
                      ▼
             🧭 ATTACK PATHS
```

> [!important] 🔑 CPTS Mental Model  
> **Credentialed enumeration changes the game.**
> 
> Instead of asking:
> 
> > "What can I discover from the outside?"
> 
> We can now ask:
> 
> > **"What does a legitimate domain user have permission to see?"**
> 
> This can expose relationships and permissions that are invisible to an unauthenticated attacker.

---

# 🛠️ 4. CrackMapExec

[CrackMapExec / NetExec project](https://github.com/byt3bl33d3r/CrackMapExec?utm_source=chatgpt.com)

**CrackMapExec (CME)** is a powerful toolkit for assessing AD environments. The source notes that CME is now commonly associated with **NetExec** and uses functionality from projects such as Impacket and PowerSploit.

The important concept is that CME provides a common interface for interacting with multiple protocols:

```text
crackmapexec
     │
     ├── mssql
     ├── smb
     ├── ssh
     └── winrm
```

Use:

```bash
crackmapexec -h
```

to see available protocols and global options.

### CME Global Options

```text
-t THREADS
```

Number of concurrent threads.

```text
--timeout TIMEOUT
```

Maximum timeout for each thread.

```text
--jitter INTERVAL
```

Random delay between connections.

```text
--verbose
```

Enable verbose output.

The source's example shows:

```text
Version: 5.0.2dev
Codename: P3l1as
```

and the available protocols:

```text
mssql
smb
ssh
winrm
```

---

# 🔎 5. CME — SMB Options

Use:

```bash
crackmapexec smb -h
```

Important authentication options include:

```text
-u USERNAME
-p PASSWORD
-k / --kerberos
-H HASH
-d DOMAIN
--local-auth
```

Important enumeration options include:

```text
--users
--groups
--loggedon-users
--shares
--sessions
--disks
--local-groups
--pass-pol
--rid-brute
```

There are also options for:

```text
--sam
--lsa
--ntds
--wmi
--spider
--get-file
--put-file
--exec-method
-x
-X
```

> [!tip] 🧠 CPTS  
> CME is useful because you can move from **authentication → enumeration → targeted host investigation** without changing tools for every basic SMB task.

---

# 👤 6. CME — Domain User Enumeration

Because the Domain Controller contains the domain database, it is an excellent target for credentialed enumeration.

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --users
```

### 📤 Example

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\forend:Klmcargo2
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] Enumerated domain user(s)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\administrator                  badpwdcount: 0 baddpwdtime: 2022-03-29 12:29:14.476567
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\guest                          badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\lab_adm                        badpwdcount: 0 baddpwdtime: 2022-04-09 23:04:58.611828
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\krbtgt                         badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\htb-student                    badpwdcount: 0 baddpwdtime: 2022-03-30 16:27:41.960920
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\avazquez                       badpwdcount: 3 baddpwdtime: 2022-02-24 18:10:01.903395
```

> [!important] 🔐 `badPwdCount`  
> `badPwdCount` records failed password attempts for the account.
> 
> This is especially useful when performing password spraying because users with existing failed attempts can be treated cautiously.
> 
> A potential workflow:
> 
> ```text
> 👤 Enumerate users
>       ↓
> 🔎 Check badPwdCount
>       ↓
> 🚫 Exclude risky accounts
>       ↓
> 🔐 Perform authorized password testing
> ```

---

# 👥 7. CME — Domain Group Enumeration

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --groups
```

### 📤 Example

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\forend:Klmcargo2
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] Enumerated domain group(s)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Administrators                           membercount: 3
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Users                                    membercount: 4
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Guests                                   membercount: 2
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Print Operators                          membercount: 0
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Backup Operators                         membercount: 1
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Replicator                               membercount: 0

<SNIP>

SMB         172.16.5.5      445    ACADEMY-EA-DC01  Domain Admins                            membercount: 19
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Domain Users                             membercount: 0

<SNIP>

SMB         172.16.5.5      445    ACADEMY-EA-DC01  Contractors                              membercount: 138
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Accounting                               membercount: 15
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Engineering                              membercount: 19
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Executives                               membercount: 10
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Human Resources                          membercount: 36

<SNIP>
```

The output provides:

- Group names
    
- Member counts
    
- Built-in groups
    
- Domain-specific groups
    

### 🎯 Groups Worth Noting

```text
Administrators
Domain Admins
Executives
IT / administrative groups
Backup Operators
Other privileged groups
```

> [!important] 🧠 Why?  
> A group may contain users with significantly greater privileges than the account currently being used.
> 
> Therefore:
> 
> **Groups → Members → Privileges → Potential attack paths**

Save enumeration output to files as you work so that it can later be correlated with other enumeration results and used for reporting.

---

# 🪑 8. CME — Logged-On Users

CME can also target individual hosts rather than only the Domain Controller.

Example against the file server:

```bash
sudo crackmapexec smb 172.16.5.130 -u forend -p Klmcargo2 --loggedon-users
```

### 📤 Example

```text
SMB         172.16.5.130    445    ACADEMY-EA-FILE  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-FILE) (domain:INLANEFREIGHT.LOCAL) (signing:False) (SMBv1:False)
SMB         172.16.5.130    445    ACADEMY-EA-FILE  [+] INLANEFREIGHT.LOCAL\forend:Klmcargo2 (Pwn3d!)
SMB         172.16.5.130    445    ACADEMY-EA-FILE  [+] Enumerated loggedon users
SMB         172.16.5.130    445    ACADEMY-EA-FILE  INLANEFREIGHT\clusteragent              logon_server: ACADEMY-EA-DC01
SMB         172.16.5.130    445    ACADEMY-EA-FILE  INLANEFREIGHT\lab_adm                   logon_server: ACADEMY-EA-DC01
SMB         172.16.5.130    445    ACADEMY-EA-FILE  INLANEFREIGHT\svc_qualys                logon_server: ACADEMY-EA-DC01
SMB         172.16.5.130    445    ACADEMY-EA-FILE  INLANEFREIGHT\wley                      logon_server: ACADEMY-EA-DC01

<SNIP>
```

### 🧠 Important Findings

Several interesting users are currently logged into the host:

```text
clusteragent
lab_adm
svc_qualys
wley
```

The source previously identified `svc_qualys` as a **Domain Admin**.

This makes a logged-on privileged user particularly interesting during an assessment because the host may provide a path to investigate that user's session and credentials.

Also note:

```text
(Pwn3d!)
```

CME uses this indicator when the authenticated account has **local administrator-level access** to the target.

So:

```text
Authentication succeeded
        +
Local administrative access
        ↓
     (Pwn3d!)
```

> [!tip] 🩸 BloodHound Connection  
> CME is excellent for **targeted user hunting**.
> 
> BloodHound can later provide a much broader graphical view of sessions and relationships.

---

# 📂 9. CME — SMB Share Enumeration

Use:

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --shares
```

### 📤 Example

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\forend:Klmcargo2
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] Enumerated shares
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Share           Permissions     Remark
SMB         172.16.5.5      445    ACADEMY-EA-DC01  -----           -----------     ------
SMB         172.16.5.5      445    ACADEMY-EA-DC01  ADMIN$                          Remote Admin
SMB         172.16.5.5      445    ACADEMY-EA-DC01  C$                              Default share
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Department Shares READ
SMB         172.16.5.5      445    ACADEMY-EA-DC01  IPC$            READ            Remote IPC
SMB         172.16.5.5      445    ACADEMY-EA-DC01  NETLOGON        READ            Logon server share
SMB         172.16.5.5      445    ACADEMY-EA-DC01  SYSVOL          READ            Logon server share
SMB         172.16.5.5      445    ACADEMY-EA-DC01  User Shares     READ
SMB         172.16.5.5      445    ACADEMY-EA-DC01  ZZZ_archive     READ
```

### 🔎 Interesting Shares

The standard shares:

```text
IPC$
NETLOGON
SYSVOL
```

are expected in a domain environment.

The non-standard shares are often more interesting:

```text
Department Shares
User Shares
ZZZ_archive
```

These may contain:

- Credentials
    
- Scripts
    
- Configuration files
    
- Sensitive documents
    
- Other assessment-relevant information
    

---

# 🕷️ 10. CME `spider_plus`

CME's `spider_plus` module can recursively enumerate readable files within a share.

Example:

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M spider_plus --share 'Department Shares'
```

### 📤 Example Output

```text
SPIDER_P... 172.16.5.5      445    ACADEMY-EA-DC01  [*] Started spidering plus with option:
SPIDER_P... 172.16.5.5      445    ACADEMY-EA-DC01  [*]        DIR: ['print$']
SPIDER_P... 172.16.5.5      445    ACADEMY-EA-DC01  [*]        EXT: ['ico', 'lnk']
SPIDER_P... 172.16.5.5      445    ACADEMY-EA-DC01  [*]       SIZE: 51200
SPIDER_P... 172.16.5.5      445    ACADEMY-EA-DC01  [*]     OUTPUT: /tmp/cme_spider_plus
```

The results are written as JSON under:

```text
/tmp/cme_spider_plus/<IP>.json
```

For this example:

```bash
head -n 10 /tmp/cme_spider_plus/172.16.5.5.json
```

### 📤 Example

```json
{
    "Department Shares": {
        "Accounting/Private/AddSelect.bat": {
            "atime_epoch": "2022-03-31 14:44:42",
            "ctime_epoch": "2022-03-31 14:44:39",
            "mtime_epoch": "2022-03-31 15:14:46",
            "size": "278 Bytes"
        },
        "Accounting/Private/ApproveConnect.wmf": {
            "atime_epoch": "2022-03-31 14:45:14",
```

### 🧠 What Are We Looking For?

Spidering gives us a **file inventory first**.

Then we can investigate interesting files such as:

```text
web.config
*.bat
*.ps1
*.xml
configuration files
scripts
```

The goal is to identify potentially sensitive information such as hardcoded credentials.

```text
📂 Share
   ↓
🕷️ Spider directories
   ↓
📋 File inventory
   ↓
🔎 Identify interesting files
   ↓
📥 Retrieve files if authorized
   ↓
🔍 Analyze contents
```

---

# 📂 11. SMBMap

[SMBMap](https://github.com/ShawnDEvans/smbmap?utm_source=chatgpt.com)

**SMBMap** is another Linux-based tool for enumerating SMB shares.

It can provide:

- Share listings
    
- Permissions
    
- Directory contents
    
- Recursive directory listings
    
- File searching
    
- File downloads/uploads
    
- Other SMB interaction
    

> [!info] 🧠 CME vs SMBMap  
> Both can enumerate SMB resources.
> 
> **CME:** broader AD/SMB assessment framework.
> 
> **SMBMap:** particularly convenient for exploring **SMB shares, permissions, directories, and files**.

---

# 🔎 12. SMBMap — Check Share Access

```bash
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5
```

### 📤 Example

```text
[+] IP: 172.16.5.5:445  Name: inlanefreight.local
        Disk                                                    Permissions Comment
    ----                                                    ----------- -------
    ADMIN$                                              NO ACCESS   Remote Admin
    C$                                                  NO ACCESS   Default share
    Department Shares                                   READ ONLY
    IPC$                                                READ ONLY   Remote IPC
    NETLOGON                                            READ ONLY   Logon server share
    SYSVOL                                              READ ONLY   Logon server share
    User Shares                                         READ ONLY
    ZZZ_archive                                         READ ONLY
```

This tells us exactly what the `forend` account can access.

### 🧠 Expected Standard Shares

For a normal domain user, it is expected that access to:

```text
ADMIN$
C$
```

may be unavailable.

Meanwhile:

```text
IPC$
NETLOGON
SYSVOL
```

are commonly readable by domain users.

The non-standard shares deserve closer inspection.

---

# 📁 13. SMBMap — Recursive Directory Enumeration

To recursively enumerate directories:

```bash
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5 -R 'Department Shares' --dir-only
```

### 📤 Example

```text
[+] IP: 172.16.5.5:445  Name: inlanefreight.local
        Disk                                                    Permissions Comment
    ----                                                    ----------- -------
    Department Shares                                   READ ONLY
    .\Department Shares\*
    dr--r--r--                0 Thu Mar 31 15:34:29 2022    .
    dr--r--r--                0 Thu Mar 31 15:34:29 2022    ..
    dr--r--r--                0 Thu Mar 31 15:14:48 2022    Accounting
    dr--r--r--                0 Thu Mar 31 15:14:39 2022    Executives
    dr--r--r--                0 Thu Mar 31 15:14:57 2022    Finance
    dr--r--r--                0 Thu Mar 31 15:15:04 2022    HR
    dr--r--r--                0 Thu Mar 31 15:15:21 2022    IT
    dr--r--r--                0 Thu Mar 31 15:15:29 2022    Legal
    dr--r--r--                0 Thu Mar 31 15:15:37 2022    Marketing
    dr--r--r--                0 Thu Mar 31 15:15:47 2022    Operations
    dr--r--r--                0 Thu Mar 31 15:15:58 2022    R&D
    dr--r--r--                0 Thu Mar 31 15:16:10 2022    Temp
    dr--r--r--                0 Thu Mar 31 15:16:18 2022    Warehouse

<SNIP>
```

`--dir-only` restricts the output to directories rather than listing every file.

---

# 🛰️ 14. rpcclient

[Samba rpcclient documentation](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html?utm_source=chatgpt.com)

`rpcclient` is a Samba utility that communicates with Windows systems using **MS-RPC**.

It can perform extensive AD enumeration and can also perform operations that modify AD objects. Those modification capabilities are outside the scope here.

Use:

```bash
man rpcclient
```

to review available commands.

---

# 🔓 15. rpcclient — SMB NULL Session

If the target permits SMB NULL sessions:

```bash
rpcclient -U "" -N 172.16.5.5
```

This attempts an unauthenticated connection.

If successful, you receive an `rpcclient` prompt:

```text
rpcclient $>
```

From there, various domain enumeration commands can be executed.

> [!important] 🔑 Connection With vs Without Credentials  
> `rpcclient` can be useful in both:
> 
> ```text
> SMB NULL session → unauthenticated enumeration
> ```
> 
> and:
> 
> ```text
> Valid domain credentials → authenticated enumeration
> ```
> 
> The amount of information available depends on the target's configuration and permissions.

---

# 🆔 16. RID — Relative Identifier

When enumerating users with `rpcclient`, you will see:

```text
rid:[0x457]
```

A **RID (Relative Identifier)** is the object-specific portion of a Windows security identifier.

The domain has a base SID:

```text
S-1-5-21-3842939050-3880317879-2865463114
```

An object's RID is appended to the domain SID.

For example:

```text
Domain SID:
S-1-5-21-3842939050-3880317879-2865463114

htb-student RID:
0x457

0x457 = 1111 decimal
```

Therefore:

```text
S-1-5-21-3842939050-3880317879-2865463114-1111
```

represents the `htb-student` object in this example.

### 🧠 Mental Model

```text
Domain SID
    │
    └── S-1-5-21-3842939050-3880317879-2865463114
                                      │
                                      ▼
                                   + RID
                                      │
                                      ▼
                          Unique Object SID
```

---

# 🔢 17. Well-Known RIDs

Some built-in accounts have predictable RIDs.

The built-in **Administrator** account uses:

```text
RID = 500 decimal
RID = 0x1f4 hexadecimal
```

Therefore, if the domain SID is:

```text
S-1-5-21-3842939050-3880317879-2865463114
```

the built-in Administrator's SID ends in:

```text
-500
```

> [!tip] CPTS  
> RIDs are useful because once you know an object's RID, you can query additional information about that object.

---

# 👤 18. rpcclient — Query a User by RID

Example:

```text
rpcclient $> queryuser 0x457
```

### 📤 Output

```text
User Name   :   htb-student
Full Name   :   Htb Student
Home Drive  :
Dir Drive   :
Profile Path:
Logon Script:
Description :
Workstations:
Comment     :
Remote Dial :
Logon Time               :      Wed, 02 Mar 2022 15:34:32 EST
Logoff Time              :      Wed, 31 Dec 1969 19:00:00 EST
Kickoff Time             :      Wed, 13 Sep 30828 22:48:05 EDT
Password last set Time   :      Wed, 27 Oct 2021 12:26:52 EDT
Password can change Time :      Thu, 28 Oct 2021 12:26:52 EDT
Password must change Time:      Wed, 13 Sep 30828 22:48:05 EDT
unknown_2[0..31]...
user_rid :      0x457
group_rid:      0x201
acb_info :      0x00000010
fields_present: 0x00ffffff
logon_divs:     168
bad_password_count:     0x00000000
logon_count:    0x0000001d
padding1[0..7]...
logon_hrs[0..21]...
```

The important idea is:

```text
Known RID
   ↓
queryuser
   ↓
Detailed user attributes
```

---

# 👥 19. rpcclient — Enumerate All Domain Users

If you do not know individual RIDs, enumerate users:

```text
rpcclient $> enumdomusers
```

### 📤 Example

```text
user:[administrator] rid:[0x1f4]
user:[guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[lab_adm] rid:[0x3e9]
user:[htb-student] rid:[0x457]
user:[avazquez] rid:[0x458]
user:[pfalcon] rid:[0x459]
user:[fanthony] rid:[0x45a]
user:[wdillard] rid:[0x45b]
user:[lbradford] rid:[0x45c]
user:[sgage] rid:[0x45d]
user:[asanchez] rid:[0x45e]
user:[dbranch] rid:[0x45f]
user:[ccruz] rid:[0x460]
user:[njohnson] rid:[0x461]
user:[mholliday] rid:[0x462]

<SNIP>
```

This gives:

```text
Username ↔ RID
```

which can then be used for more detailed enumeration.

---

# 🧰 20. Impacket Toolkit

[Impacket on GitHub](https://github.com/fortra/impacket?utm_source=chatgpt.com)

**Impacket** is a Python toolkit for interacting with Windows protocols.

It provides tools for:

- Enumeration
    
- Authentication
    
- Remote interaction
    
- Protocol abuse
    
- Exploitation
    

The source focuses on:

```text
psexec.py
wmiexec.py
```

---

# 💻 21. `psexec.py`

`psexec.py` is an implementation inspired by Microsoft's Sysinternals PsExec.

Conceptually:

```text
Credentials with local admin
             │
             ▼
        ADMIN$ share
             │
             ▼
 Upload executable
             │
             ▼
 Register Windows service
             │
             ▼
 Service Control Manager / RPC
             │
             ▼
       Named pipe
             │
             ▼
       SYSTEM shell
```

The tool uploads a randomly named executable to the target's `ADMIN$` share, registers a service through RPC/Service Control Manager, and provides an interactive shell running as **SYSTEM**.

### Command

```bash
psexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.125
```

> [!important] 🔑 Requirement  
> The supplied account must have **local administrator privileges** on the target host.

After successful execution, the shell lands in the target's `system32` directory and:

```cmd
whoami
```

confirms the context as:

```text
SYSTEM
```

This provides extremely high privileges on that host and can enable further enumeration and lateral movement.

---

# 🖥️ 22. `wmiexec.py`

`wmiexec.py` uses **Windows Management Instrumentation (WMI)** to execute commands remotely.

[Microsoft WMI documentation](https://learn.microsoft.com/en-us/windows/win32/wmisdk/wmi-start-page?utm_source=chatgpt.com)

Example:

```bash
wmiexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.5
```

### 🧠 Key Difference From `psexec.py`

`wmiexec.py` does **not drop an executable/service in the same way as `psexec.py`**.

Instead:

```text
Remote WMI
   ↓
cmd.exe
   ↓
Command
```

The shell is semi-interactive rather than a fully interactive terminal.

Commands execute through a new `cmd.exe` process.

The resulting process runs under the context of the authenticated user, rather than automatically becoming SYSTEM.

---

# ⚖️ 23. `psexec.py` vs `wmiexec.py`

|Feature|`psexec.py`|`wmiexec.py`|
|---|---|---|
|Main mechanism|Service + RPC|WMI|
|Requires local admin|Yes|Yes|
|Drops executable/service|Yes|No executable dropped in the same manner|
|Shell|Interactive|Semi-interactive|
|Default context|`SYSTEM`|Authenticated user|
|Main advantage|SYSTEM shell|Less file-based execution|
|Logging/detection|Can be detected|Can still be detected|
|Example|`psexec.py ...`|`wmiexec.py ...`|

> [!warning] 🛡️ Detection  
> `wmiexec.py` should not be considered invisible.
> 
> The source notes that defenders can observe process creation, including **Event ID 4688**, where a new `cmd.exe` process is created to execute commands.
> 
> WMI is also legitimately used for administration, so the presence of WMI activity alone does not necessarily indicate malicious activity.

---

# 🌐 24. Windapsearch

[Windapsearch](https://github.com/ropnop/windapsearch?utm_source=chatgpt.com)

**Windapsearch** is a Python tool that performs Windows domain enumeration using **LDAP queries** against a Domain Controller.

It can enumerate:

```text
👤 Users
👥 Groups
💻 Computers
🔐 Privileged users
👑 Domain Admins
📜 GPOs
```

It can operate with anonymous binding where permitted, or with domain credentials.

---

# 🔎 25. Windapsearch — Important Options

```text
-d / --domain
```

Domain FQDN.

```text
--dc-ip
```

Domain Controller IP.

```text
-u / --user
```

Username used for LDAP bind.

```text
-p / --password
```

Password.

```text
-G / --groups
```

Enumerate AD groups.

```text
-U / --users
```

Enumerate AD users.

```text
-C / --computers
```

Enumerate AD computers.

```text
--da
```

Enumerate members of the Domain Admins group.

```text
-PU / --privileged-users
```

Find privileged users using recursive/nested group membership lookups.

> [!important] 🧠 `-PU` vs `--da`  
> `--da` focuses on **Domain Admins membership**.
> 
> `-PU` goes further and searches recursively for **privileged users**, including users whose privileges may come through nested group membership.

---

# 👑 26. Windapsearch — Domain Admins

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 --da
```

### 📤 Example

```text
[+] Using Domain Controller at: 172.16.5.5
[+] Getting defaultNamingContext from Root DSE
[+] Found: DC=INLANEFREIGHT,DC=LOCAL
[+] Attempting bind
[+] ...success! Binded as:
[+]  u:INLANEFREIGHT\forend
[+] Attempting to enumerate all Domain Admins
[+] Using DN: CN=Domain Admins,CN=Users.CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
[+] Found 28 Domain Admins:

cn: Administrator
userPrincipalName: administrator@inlanefreight.local

cn: lab_adm

cn: Matthew Morgan
userPrincipalName: mmorgan@inlanefreight.local

<SNIP>
```

The example finds:

```text
28 Domain Admins
```

This lets us correlate the Domain Admin group with credentials and users discovered earlier.

---

# 🧬 27. Windapsearch — Privileged Users & Nested Groups

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 -PU
```

### 📤 Example

```text
[+] Attempting to enumerate all AD privileged users
[+] Using DN: CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
[+]     Found 28 nested users for group Domain Admins:

cn: Administrator
userPrincipalName: administrator@inlanefreight.local

cn: lab_adm

cn: Angela Dunn
userPrincipalName: adunn@inlanefreight.local

cn: Matthew Morgan
userPrincipalName: mmorgan@inlanefreight.local

cn: Dorothy Click
userPrincipalName: dclick@inlanefreight.local

<SNIP>

[+] Using DN: CN=Enterprise Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
[+]     Found 3 nested users for group Enterprise Admins:

cn: Administrator
userPrincipalName: administrator@inlanefreight.local

cn: lab_adm

cn: Sharepoint Admin
userPrincipalName: sp-admin@INLANEFREIGHT.LOCAL

<SNIP>
```

### 🧠 Why Nested Groups Matter

Consider:

```text
User
 ↓
Group A
 ↓
Group B
 ↓
Privileged Group
 ↓
High privilege
```

A simple check of the user's direct group memberships may not reveal the final privilege.

`-PU` performs recursive lookups to uncover these relationships.

> [!warning] 🔐 Security Insight  
> Nested group membership can unintentionally give users more privilege than administrators realize.
> 
> This is both:
> 
> - An important penetration-testing finding
>     
> - An important defensive/reporting finding
>     

---

# 🩸 28. BloodHound.py

[BloodHound.py](https://github.com/fox-it/BloodHound.py?utm_source=chatgpt.com)

[BloodHound](https://github.com/BloodHoundAD/BloodHound?utm_source=chatgpt.com)

**BloodHound** is one of the most important tools for understanding AD relationships.

The fundamental idea is **graph theory**.

Instead of looking at hundreds or thousands of disconnected objects:

```text
Users
Groups
Computers
ACLs
Sessions
GPOs
Trusts
```

BloodHound represents them as a graph:

```text
        👤 User
          │
          ▼
       👥 Group
          │
          ▼
       💻 Computer
          │
          ▼
        🔐 ACL
          │
          ▼
      👑 Privileged
```

This makes complicated relationships and potential attack paths much easier to understand.

---

# 🧩 29. BloodHound Architecture

There are two major pieces:

```text
Collector / Ingestor
        │
        ▼
      JSON
        │
        ▼
BloodHound GUI
        │
        ▼
 Neo4j Database
        │
        ▼
 Graph Queries / Attack Paths
```

### Collectors

**SharpHound**

- C# collector
    
- Intended for Windows environments
    

**BloodHound.py**

- Python-based collector
    
- Useful from a Linux attack host
    
- Particularly useful when you have valid domain credentials but don't have convenient access to a domain-joined Windows host
    

### GUI

The BloodHound GUI imports the collected JSON data and lets us query the resulting AD graph.

---

# 📊 30. What BloodHound.py Can Collect

The source lists data such as:

- 👤 Users
    
- 👥 Groups
    
- 💻 Computers
    
- 🔗 Group membership
    
- 📜 GPOs
    
- 🔐 ACLs
    
- 🤝 Domain trusts
    
- 🛡️ Local administrator access
    
- 🪑 User sessions
    
- 🧬 Computer/user properties
    
- 🖥️ RDP access
    
- 🪟 WinRM access
    

This is why BloodHound can uncover relationships that would be extremely time-consuming to identify manually.

---

# ⚙️ 31. BloodHound.py Collection Methods

Use:

```bash
bloodhound-python -h
```

The important option is:

```text
-c / --collectionmethod
```

Supported collection methods shown in the source:

```text
Group
LocalAdmin
Session
Trusts
Default
DCOnly
DCOM
RDP
PSRemote
LoggedOn
ObjectProps
ACL
All
```

Multiple methods can be specified using commas.

### 🧠 Important Modes

```text
Default
```

Collects the default set.

```text
DCOnly
```

Collects domain-controller information without computer connections.

```text
All
```

Collects all supported data except `LoggedOn`.

---

# 🚀 32. BloodHound.py — Full Collection

```bash
sudo bloodhound-python -u 'forend' -p 'Klmcargo2' -ns 172.16.5.5 -d inlanefreight.local -c all
```

### 🧩 Important Arguments

```text
-u 'forend'
```

Domain username.

```text
-p 'Klmcargo2'
```

Password.

```text
-ns 172.16.5.5
```

Use the Domain Controller as the DNS nameserver.

```text
-d inlanefreight.local
```

Target domain.

```text
-c all
```

Collect all supported information except the source-defined exclusions.

### 📤 Example Output

```text
INFO: Found AD domain: inlanefreight.local
INFO: Connecting to LDAP server: ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
INFO: Found 1 domains
INFO: Found 2 domains in the forest
INFO: Found 564 computers
INFO: Connecting to LDAP server: ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
INFO: Found 2951 users
INFO: Connecting to GC LDAP server: ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
INFO: Found 183 groups
INFO: Found 2 trusts
INFO: Starting computer enumeration with 10 workers

<SNIP>
```

This immediately gives us an idea of the environment's scale:

```text
564 computers
2951 users
183 groups
2 trusts
```

---

# 📄 33. BloodHound Output Files

When collection completes, JSON files are written to the current directory.

Example:

```bash
ls
```

```text
20220307163102_computers.json
20220307163102_domains.json
20220307163102_groups.json
20220307163102_users.json
```

These JSON files contain the collected AD relationships and objects that will be imported into BloodHound.

---

# 🗄️ 34. Neo4j + BloodHound GUI

BloodHound uses **Neo4j** as its graph database.

[Neo4j](https://neo4j.com/?utm_source=chatgpt.com)

Start Neo4j:

```bash
sudo neo4j start
```

Then launch:

```bash
bloodhound
```

The source's example credentials are:

```text
Username: neo4j
Password: HTB_@cademy_stdnt!
```

---

# 📦 35. Upload BloodHound Data

The JSON files can be uploaded individually, or compressed first.

```bash
zip -r ilfreight_bh.zip *.json
```

Then upload the resulting ZIP through:

```text
BloodHound GUI
      ↓
Upload Data
      ↓
Select ZIP / JSON
      ↓
Import
      ↓
Analyze
```

---

# 🧭 36. BloodHound Analysis

Once the data is imported, BloodHound can visualize relationships between:

```text
👤 Users
👥 Groups
💻 Computers
🔐 ACLs
📜 GPOs
🤝 Trusts
🪑 Sessions
```

The **Analysis** tab contains pre-built queries.

One particularly useful example from the source is:

```text
Find Shortest Paths To Domain Admins
```

The resulting graph can show logical relationships through:

```text
Users
   ↓
Groups
   ↓
Computers
   ↓
ACLs
   ↓
GPOs
   ↓
Trusts
   ↓
Domain Admin privileges
```

> [!important] 🧠 Why BloodHound Is Different  
> CME, SMBMap, rpcclient, and Windapsearch primarily give you **pieces of information**.
> 
> BloodHound's major advantage is connecting those pieces into a **relationship graph**.
> 
> ```text
> Enumeration tools
>        ↓
>     Raw data
>        ↓
>   BloodHound graph
>        ↓
> Relationships
>        ↓
> Potential attack paths
> ```

---

# 🔗 37. Custom Cypher Queries

BloodHound supports custom queries using **Cypher**, the query language used for graph databases.

[Cypher introduction for BloodHound](https://specterops.io/blog/2017/09/18/bloodhound-intro-to-cypher/?utm_source=chatgpt.com)

The source also recommends experimenting with:

```text
Raw Query
```

to run custom Cypher queries.

This becomes especially useful when the built-in analysis queries do not answer a specific question.

---

# 🧭 38. Complete Linux Credentialed Enumeration Workflow

```text
                  🔑 VALID DOMAIN CREDS
                           │
                           ▼
                🏢 ENUMERATE DOMAIN
                           │
        ┌──────────────────┼───────────────────┐
        ▼                  ▼                   ▼
      CME              Windapsearch        rpcclient
        │                  │                   │
        ▼                  ▼                   ▼
    Users/Groups       Privileged users    Users/RIDs
    Sessions           Nested groups       Attributes
    Shares
        │
        ▼
     SMBMap
        │
        ▼
  Shares / Directories
        │
        ▼
   🕷️ Spider files
        │
        ▼
 Sensitive information
        │
        └──────────────────┐
                           ▼
                       Impacket
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
             psexec.py          wmiexec.py
                 │                   │
                 ▼                   ▼
              SYSTEM             User context
                           │
                           ▼
                    🩸 BloodHound.py
                           │
                           ▼
                    Graph relationships
                           │
                           ▼
                     🧭 Attack paths
```

---

# ⚖️ 39. Tool Selection — Quick Comparison

|Tool|Main Strength|Key Use|
|---|---|---|
|**CrackMapExec**|Broad SMB/Windows enumeration|Users, groups, sessions, shares, host access|
|**SMBMap**|SMB share exploration|Permissions, directories, files|
|**rpcclient**|MS-RPC enumeration|Users, RIDs, detailed object information|
|**Impacket**|Windows protocol interaction|Remote execution and many other AD tasks|
|**Windapsearch**|LDAP enumeration|Users, groups, computers, privileged users|
|**BloodHound.py**|Relationship mapping|ACLs, sessions, trusts, attack paths|

> [!tip] 🧠 The Practical Difference
> 
> **CME:** "Give me useful information from this Windows/SMB host."
> 
> **SMBMap:** "Show me what I can access inside SMB shares."
> 
> **rpcclient:** "Query Windows/MS-RPC objects and their attributes."
> 
> **Windapsearch:** "Ask LDAP detailed questions about the domain."
> 
> **BloodHound:** "Connect all these relationships and show me how they relate."

---

# 🧠 40. CPTS Enumeration Methodology

A useful way to remember the sequence is:

```text
🔑 CREDENTIALS
     ↓
🏢 DOMAIN
     ↓
👤 USERS
     ↓
👥 GROUPS
     ↓
👑 PRIVILEGED USERS
     ↓
💻 COMPUTERS
     ↓
🪑 SESSIONS
     ↓
📂 SHARES
     ↓
📄 FILES
     ↓
🔐 PERMISSIONS / ACLs
     ↓
🤝 TRUSTS
     ↓
🩸 BLOODHOUND
     ↓
🧭 RELATIONSHIPS / ATTACK PATHS
     ↓
➡️ NEXT ENUMERATION OR LATERAL-MOVEMENT STEP
```

> [!success] 🎯 Key Takeaways
> 
> - **Valid domain credentials dramatically expand AD visibility.**
>     
> - Start with the **Domain Controller** because it contains central domain information.
>     
> - Use **CME** for broad SMB-based enumeration.
>     
> - Pay attention to `badPwdCount`, privileged groups, and logged-on users.
>     
> - `Pwn3d!` in CME indicates local administrative access to that host.
>     
> - Enumerate SMB shares and investigate **non-standard readable shares**.
>     
> - Use **SMBMap** when you want to dig deeply into share contents and directory structures.
>     
> - `rpcclient` provides powerful MS-RPC enumeration and exposes useful **RIDs** and object attributes.
>     
> - **Windapsearch** uses LDAP and is particularly useful for finding privileged users and nested group membership.
>     
> - `psexec.py` can provide a **SYSTEM** shell when valid local-admin credentials are available.
>     
> - `wmiexec.py` executes commands through WMI and runs them under the authenticated user's context.
>     
> - **BloodHound.py** turns large amounts of AD enumeration data into a graph of relationships.
>     
> - BloodHound is especially valuable for understanding **ACLs, group membership, sessions, trusts, GPOs, and potential attack paths**.
>     
> 
> **The big CPTS lesson:**
> 
> ```text
> Don't just enumerate objects.
> 
> Enumerate RELATIONSHIPS.
> ```
> 
> A username by itself may not mean much.
> 
> ```text
> User
>  ↓
> Group
>  ↓
> Computer
>  ↓
> Session
>  ↓
> Permission / ACL
>  ↓
> Privileged resource
> ```
> 
> **That chain is where AD attack paths emerge.**