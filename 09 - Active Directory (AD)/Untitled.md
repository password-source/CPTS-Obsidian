# 🛡️ 1. Enumerating Security Controls

> [!info] 🎯 Objective  
> After gaining a foothold in an Active Directory environment, enumerate the **defensive security controls** on compromised hosts.
> 
> Security controls directly affect:
> 
> - 🔎 AD enumeration
>     
> - 🛠️ Tool selection
>     
> - 💥 Exploitation
>     
> - 🔧 Post-exploitation
>     
> - 🏃 Living-off-the-land techniques
>     
> - 🧭 Overall attack methodology
>     
> 
> The goal is **not necessarily to bypass the controls immediately**, but to understand what protections exist so that tool and technique selection can be adapted accordingly. Controls may also differ between machines, meaning one host may have restrictions that another does not.
> 
> [!warning] 📚 Module Scope  
> This section demonstrates possible security controls that may be encountered during an assessment. **Enumerating and bypassing these controls is outside the scope of this module.**

---

# ⚡ 2. Quick Command Cheat Sheet

|Purpose|Command|
|---|---|
|Check Microsoft Defender status|`Get-MpComputerStatus`|
|Check effective AppLocker rules|`Get-AppLockerPolicy -Effective \| select -ExpandProperty RuleCollections`|
|Check PowerShell Language Mode|`$ExecutionContext.SessionState.LanguageMode`|
|Find groups delegated LAPS rights|`Find-LAPSDelegatedGroups`|
|Find LAPS extended rights|`Find-AdmPwdExtendedRights`|
|Find LAPS-enabled computers/passwords|`Get-LAPSComputers`|

### 🔗 Tools / Documentation

- [Microsoft Defender PowerShell documentation](https://learn.microsoft.com/en-us/powershell/module/defender/get-mpcomputerstatus?utm_source=chatgpt.com)
    
- [Microsoft AppLocker documentation](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/?utm_source=chatgpt.com)
    
- [PowerShell Constrained Language Mode](https://devblogs.microsoft.com/powershell/powershell-constrained-language-mode/?utm_source=chatgpt.com)
    
- [Microsoft LAPS](https://www.microsoft.com/en-us/download/details.aspx?id=46899&utm_source=chatgpt.com)
    
- [LAPSToolkit](https://github.com/leoloobeek/LAPSToolkit?utm_source=chatgpt.com)
    

---

# 🧠 3. Why Enumerate Security Controls?

Once we obtain an initial foothold:

```text
             INITIAL FOOTHOLD
                    │
                    ▼
          🛡️ Enumerate Defenses
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
     Defender   AppLocker      CLM
        │           │            │
        └───────────┼────────────┘
                    ▼
                 LAPS
                    │
                    ▼
          🧭 Adapt Methodology
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      Tool-based          Native tools /
      enumeration         LOLBins
```

> [!important] 🧩 Mental Model  
> **Security controls determine what techniques are practical on a host.**
> 
> Before blindly running your usual enumeration toolkit, determine whether the host is protected by controls that may block, restrict, or detect those tools.

Different hosts can have different policies:

```text
Host A
├── Defender enabled
├── AppLocker restrictive
└── Constrained Language Mode

Host B
├── Defender enabled
├── AppLocker permissive
└── Full Language Mode
```

Therefore, **a technique failing on one host does not necessarily mean the technique itself is wrong**. The local security configuration may be the reason.

---

# 🛡️ 4. Windows Defender

[Microsoft Defender](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-windows?utm_source=chatgpt.com)

Windows Defender, now generally referred to as **Microsoft Defender**, has improved significantly over time.

By default, it can block tools such as **PowerView**.

The first step is to determine whether Defender and its protections are enabled.

## 🔎 4.1 Check Defender Status

PowerShell provides the built-in:

```powershell
Get-MpComputerStatus
```

The important parameter in this context is:

```text
RealTimeProtectionEnabled
```

If it is:

```text
True
```

then real-time protection is enabled on the system.

### 💻 Command

```powershell
PS C:\htb> Get-MpComputerStatus
```

### 📤 Example Output

```text
AMEngineVersion                 : 1.1.17400.5
AMProductVersion                : 4.10.14393.0
AMServiceEnabled                : True
AMServiceVersion                : 4.10.14393.0
AntispywareEnabled              : True
AntispywareSignatureAge         : 1
AntispywareSignatureLastUpdated : 9/2/2020 11:31:50 AM
AntispywareSignatureVersion     : 1.323.392.0
AntivirusEnabled                : True
AntivirusSignatureAge           : 1
AntivirusSignatureLastUpdated   : 9/2/2020 11:31:51 AM
AntivirusSignatureVersion       : 1.323.392.0
BehaviorMonitorEnabled          : False
ComputerID                      : 07D23A51-F83F-4651-B9ED-110FF2B83A9C
ComputerState                   : 0
FullScanAge                     : 4294967295
FullScanEndTime                 :
FullScanStartTime               :
IoavProtectionEnabled           : False
LastFullScanSource              : 0
LastQuickScanSource             : 2
NISEnabled                      : False
NISEngineVersion               : 0.0.0.0
NISSignatureAge                : 4294967295
NISSignatureLastUpdated        :
NISSignatureVersion            : 0.0.0.0
OnAccessProtectionEnabled      : False
QuickScanAge                   : 0
QuickScanEndTime               : 9/3/2020 12:50:45 AM
QuickScanStartTime             : 9/3/2020 12:49:49 AM
RealTimeProtectionEnabled      : True
RealTimeScanDirection          : 0
PSComputerName                 :
```

### 🔑 Important Fields

|Parameter|What it tells you|
|---|---|
|`AMServiceEnabled`|Whether the Defender service is enabled|
|`AntispywareEnabled`|Whether antispyware protection is enabled|
|`AntivirusEnabled`|Whether antivirus protection is enabled|
|`BehaviorMonitorEnabled`|Whether behavior monitoring is enabled|
|`IoavProtectionEnabled`|Whether downloaded-file protection is enabled|
|`NISEnabled`|Whether Network Inspection System is enabled|
|`OnAccessProtectionEnabled`|Whether on-access protection is enabled|
|`RealTimeProtectionEnabled`|Whether real-time protection is enabled|

> [!tip] 🧠 CPTS Perspective  
> Don't simply ask **"Is Defender installed?"**
> 
> Ask:
> 
> **"Which Defender protections are actually enabled on this host?"**
> 
> `Get-MpComputerStatus` gives you a much more useful picture than merely confirming that Defender exists.

---

# 🔐 5. AppLocker

[Microsoft AppLocker documentation](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/?utm_source=chatgpt.com)

**AppLocker** is Microsoft's application-whitelisting technology.

An application whitelist defines which software, executables, scripts, installers, DLLs, packaged applications, and packaged-app installers users are permitted to run.

The basic security model is:

```text
                User attempts to run file
                           │
                           ▼
                     🛡️ AppLocker
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
              ALLOWED               DENIED
                 │                   │
                 ▼                   ▼
              Execute             Block
```

Organizations may create rules that:

- Block `cmd.exe`
    
- Block `PowerShell.exe`
    
- Restrict scripts
    
- Restrict writable directories
    
- Allow only approved applications
    
- Restrict execution based on path, publisher, or hash
    

---

## 🔎 5.1 AppLocker Can Be More Specific Than "PowerShell Is Blocked"

A common configuration is to block the standard 64-bit PowerShell executable:

```text
%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe
```

The source example shows that **Domain Users** are denied from executing this particular path.

However, PowerShell can exist at other executable locations, including:

```text
%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe
```

and:

```text
PowerShell_ISE.exe
```

Therefore:

> [!important] 🧠 Mental Model  
> **"PowerShell.exe is blocked" ≠ "PowerShell is completely unavailable."**
> 
> You need to enumerate the **actual AppLocker rules and their scope**.

---

# 🔎 6. Enumerating Effective AppLocker Rules

Use:

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

### 📤 Example Output

```text
PathConditions      : {%SYSTEM32%\WINDOWSPOWERSHELL\V1.0\POWERSHELL.EXE}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 3d57af4a-6cf8-4e5b-acfc-c2c2956061fa
Name                : Block PowerShell
Description         : Blocks Domain Users from using PowerShell on workstations
UserOrGroupSid      : S-1-5-21-2974783224-3764228556-2640795941-513
Action              : Deny

PathConditions      : {%PROGRAMFILES%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 921cc481-6e17-4653-8f75-050b80acca20
Name                : (Default Rule) All files located in the Program Files folder
Description         : Allows members of the Everyone group to run applications that are located in the Program Files folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {%WINDIR%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : a61c8b2c-a319-4cd0-9690-d2177cad7b51
Name                : (Default Rule) All files located in the Windows folder
Description         : Allows members of the Everyone group to run applications that are located in the Windows folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : fd686d83-a829-4351-8ff4-27c7de5755d2
Name                : (Default Rule) All files
Description         : Allows members of the local Administrators group to run all applications.
UserOrGroupSid      : S-1-5-32-544
Action              : Allow
```

### 🧩 Reading the Rules

The important fields are:

|Field|Meaning|
|---|---|
|`PathConditions`|File/path the rule applies to|
|`PathExceptions`|Paths excluded from the rule|
|`PublisherExceptions`|Publisher-based exclusions|
|`HashExceptions`|File-hash exclusions|
|`UserOrGroupSid`|User/group the rule applies to|
|`Action`|`Allow` or `Deny`|
|`Name`|Human-readable rule name|
|`Description`|Administrator's explanation of the rule|

For example:

```text
PathConditions : %SYSTEM32%\WINDOWSPOWERSHELL\V1.0\POWERSHELL.EXE
Action         : Deny
```

means the policy explicitly denies execution of that PowerShell executable for the specified SID.

---

# 🧠 7. PowerShell Constrained Language Mode

[PowerShell Constrained Language Mode](https://devblogs.microsoft.com/powershell/powershell-constrained-language-mode/?utm_source=chatgpt.com)

Even when PowerShell itself can execute, the environment may restrict **what PowerShell is capable of doing**.

**PowerShell Constrained Language Mode (CLM)** restricts many features that are available in Full Language Mode, including:

- COM objects
    
- Unapproved .NET types
    
- XAML-based workflows
    
- PowerShell classes
    
- Other functionality commonly needed for unrestricted PowerShell operation
    

### 🔎 7.1 Check the Current Language Mode

```powershell
PS C:\htb> $ExecutionContext.SessionState.LanguageMode
```

### 📤 Example

```text
ConstrainedLanguage
```

### 🧠 Mental Model

Think of the three controls as different layers:

```text
                    PowerShell
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        AppLocker              Language Mode
             │                     │
       "Can I run it?"       "What can I do?"
             │                     │
             ▼                     ▼
       Execution policy       Capability limits
```

> [!important] 🔑 Key Distinction  
> **AppLocker** can prevent an executable from running.
> 
> **Constrained Language Mode** can allow PowerShell to run while restricting what PowerShell can do.

---

# 🔑 8. LAPS — Local Administrator Password Solution

[Microsoft LAPS](https://www.microsoft.com/en-us/download/details.aspx?id=46899&utm_source=chatgpt.com)

**LAPS (Local Administrator Password Solution)** is designed to randomize and rotate local administrator passwords on Windows hosts.

Its security purpose includes preventing password reuse from enabling easy lateral movement.

Conceptually:

```text
Without LAPS

PC01 ── Administrator : Password123
PC02 ── Administrator : Password123
PC03 ── Administrator : Password123
                         │
                         ▼
                 One password → many hosts
```

With LAPS:

```text
PC01 ── Administrator : RandomPassword_A
PC02 ── Administrator : RandomPassword_B
PC03 ── Administrator : RandomPassword_C
```

Therefore, obtaining one local administrator password does not automatically provide the same credential for other machines.

---

# 🔎 9. Enumerating LAPS Delegation

During AD enumeration, an important question is:

> **Which users or groups can read LAPS passwords?**

The source uses **LAPSToolkit** to help answer this.

[LAPSToolkit on GitHub](https://github.com/leoloobeek/LAPSToolkit?utm_source=chatgpt.com)

One useful function is:

```powershell
Find-LAPSDelegatedGroups
```

This parses the `ExtendedRights` associated with computers and identifies groups specifically delegated to read LAPS passwords.

### 📤 Example

```text
OrgUnit                                             Delegated Groups
-------                                             ----------------
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                INLANEFREIGHT\Domain Admins
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                INLANEFREIGHT\LAPS Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL           INLANEFREIGHT\Domain Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL           INLANEFREIGHT\LAPS Admins
OU=Web Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\Domain Admins
OU=Web Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\LAPS Admins
OU=SQL Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\Domain Admins
OU=SQL Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\LAPS Admins
OU=File Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\Domain Admins
OU=File Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\LAPS Admins
OU=Contractor Laptops,OU=Workstations,DC=INLANEF... INLANEFREIGHT\Domain Admins
OU=Contractor Laptops,OU=Workstations,DC=INLANEF... INLANEFREIGHT\LAPS Admins
OU=Staff Workstations,OU=Workstations,DC=INLANEF... INLANEFREIGHT\Domain Admins
OU=Staff Workstations,OU=Workstations,DC=INLANEF... INLANEFREIGHT\LAPS Admins
OU=Executive Workstations,OU=Workstations,DC=INL... INLANEFREIGHT\Domain Admins
OU=Executive Workstations,OU=Workstations,DC=INLANEF... INLANEFREIGHT\LAPS Admins
OU=Mail Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\Domain Admins
OU=Mail Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\LAPS Admins
```

### 🧠 Why This Matters

The output tells us **which groups have delegated access to LAPS passwords for particular OUs**.

This can reveal:

```text
Computer / OU
      │
      ▼
LAPS delegation
      │
      ▼
Who can read password?
      │
      ▼
Potentially valuable AD account/group
```

> [!tip] CPTS Focus  
> Don't just enumerate **"Does LAPS exist?"**
> 
> Also determine:
> 
> **Which computers use it → who can read the passwords → which users/groups have that right.**

---

# 🔎 10. Find LAPS Extended Rights

The next function is:

```powershell
Find-AdmPwdExtendedRights
```

This checks each computer with LAPS enabled for:

- Groups with read access
    
- Users with **All Extended Rights**
    

### 📤 Example

```text
ComputerName                Identity                    Reason
------------                --------                    ------
EXCHG01.INLANEFREIGHT.LOCAL INLANEFREIGHT\Domain Admins Delegated
EXCHG01.INLANEFREIGHT.LOCAL INLANEFREIGHT\LAPS Admins   Delegated
SQL01.INLANEFREIGHT.LOCAL   INLANEFREIGHT\Domain Admins Delegated
SQL01.INLANEFREIGHT.LOCAL   INLANEFREIGHT\LAPS Admins   Delegated
WS01.INLANEFREIGHT.LOCAL    INLANEFREIGHT\Domain Admins Delegated
WS01.INLANEFREIGHT.LOCAL    INLANEFREIGHT\LAPS Admins   Delegated
```

### 🧠 Important Concept — `All Extended Rights`

An account with **All Extended Rights** over a computer can read the LAPS password for that computer.

The source notes that users with `All Extended Rights` may be less protected than users belonging to explicitly delegated groups, making these accounts worth checking.

---

# 🔐 11. Enumerating LAPS Passwords

Once we understand LAPS delegation, the next question is:

> **Which computers have LAPS enabled, and can our current account read their passwords?**

Use:

```powershell
Get-LAPSComputers
```

This can identify:

- 💻 Computers with LAPS enabled
    
- ⏰ Password expiration times
    
- 🔑 Randomized passwords, **if the current user has permission to read them**
    

### 📤 Example

```text
ComputerName                Password       Expiration
------------                --------       ----------
DC01.INLANEFREIGHT.LOCAL    6DZ[+A/[]19d$F 08/26/2020 23:29:45
EXCHG01.INLANEFREIGHT.LOCAL oj+2A+[hHMMtj, 09/26/2020 00:51:30
SQL01.INLANEFREIGHT.LOCAL   9G#f;p41dcAe,s 09/26/2020 00:30:09
WS01.INLANEFREIGHT.LOCAL    TCaG-F)3No;l8C 09/26/2020 00:46:04
```

> [!warning] 🔐 Permission Matters  
> `Get-LAPSComputers` can show randomized passwords **only when the current account has the necessary access**.
> 
> The important enumeration chain is therefore:
> 
> **LAPS enabled → delegated rights → current user/group membership → password readable or not**

---

# 🧭 12. Complete Security-Control Enumeration Workflow

```text
                  🎯 FOOTHOLD
                      │
                      ▼
             🛡️ ENUMERATE DEFENSES
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   Defender       AppLocker         CLM
       │              │              │
       │              │              └──► Full / Constrained
       │              │
       │              └──► Allowed / Denied paths
       │
       └──► Real-time / AV / other protections
                      │
                      ▼
                    LAPS
                      │
             ┌────────┴─────────┐
             ▼                  ▼
       Delegated Groups    Extended Rights
             │                  │
             └────────┬─────────┘
                      ▼
              LAPS-enabled Hosts
                      │
                      ▼
             Password readable?
                      │
                      ▼
             🧭 AD ENUMERATION
             / TOOL SELECTION
             / NEXT ACTION
```

---

# 🧠 13. Strong CPTS Mental Model

The key idea is **not simply memorizing individual commands**.

Think of security controls as **environmental constraints** on your methodology:

```text
             "I have a foothold."
                       │
                       ▼
            "What protects this host?"
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
      Defender      AppLocker         CLM
        │              │              │
        ▼              ▼              ▼
   What gets       What can         What can
   detected/       execute?         PowerShell
   blocked?                         actually do?
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                     LAPS
                       │
                       ▼
             Who can access local
             administrator secrets?
                       │
                       ▼
              🧭 AD ATTACK PATH
```

> [!important] 🔑 Core Principle  
> **Enumeration should include the environment's defenses, not only its assets.**
> 
> You are building two pictures at the same time:
> 
> **1. What can I access?**
> 
> **2. What controls restrict how I can access it?**

---

# 📊 14. Security Controls — Quick Revision

|Control|Primary Question|Enumeration|
|---|---|---|
|🛡️ Microsoft Defender|Is Defender/protection enabled?|`Get-MpComputerStatus`|
|🔐 AppLocker|What executables/files can this user run?|`Get-AppLockerPolicy -Effective \| select -ExpandProperty RuleCollections`|
|🧱 PowerShell CLM|What PowerShell functionality is available?|`$ExecutionContext.SessionState.LanguageMode`|
|🔑 LAPS Delegation|Who can read LAPS passwords?|`Find-LAPSDelegatedGroups`|
|🔑 LAPS Extended Rights|Which users/groups have rights over computers?|`Find-AdmPwdExtendedRights`|
|💻 LAPS Computers|Which machines have LAPS and what passwords can I read?|`Get-LAPSComputers`|

---

# ✅ 15. Key Takeaways

> [!success] 🎯 CPTS Exam Mental Checklist
> 
> - 🛡️ **Defender:** check whether real-time and other protections are enabled.
>     
> - 🔐 **AppLocker:** enumerate the **effective rules**, not merely whether AppLocker exists.
>     
> - 🧱 **Constrained Language Mode:** determine whether PowerShell is running in `FullLanguage` or `ConstrainedLanguage`.
>     
> - 🔑 **LAPS:** determine which machines use LAPS and **who has rights to read the passwords**.
>     
> - 👥 **LAPS delegation:** `Find-LAPSDelegatedGroups` identifies delegated groups.
>     
> - 🔎 **Extended rights:** `Find-AdmPwdExtendedRights` checks computer-level LAPS rights.
>     
> - 💻 **LAPS passwords:** `Get-LAPSComputers` can expose randomized passwords when the current account has permission.
>     
> - 🧭 **Methodology:** security controls should influence subsequent tool and technique selection.
>     
> - 🧠 **Most important:** a failed technique may indicate a **security-control restriction**, not necessarily an incorrect technique.
>