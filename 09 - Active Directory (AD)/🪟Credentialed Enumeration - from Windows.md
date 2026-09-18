# 🪟 1. Credentialed Enumeration — From Windows

> [!info]  
> **Objective:** Once we have valid domain credentials and access to a Windows host, we can perform extensive **Active Directory enumeration directly from Windows**.
> 
> The goal is not simply to find an immediate attack path. We want to build a complete picture of:
> 
> - 👤 Users and service accounts
>     
> - 👥 Groups and nested group membership
>     
> - 🖥️ Computers and administrative access
>     
> - 🔐 SPNs / potential Kerberoasting targets
>     
> - 🧬 Trust relationships
>     
> - 📜 GPOs and ACLs
>     
> - 📂 File shares and sensitive files
>     
> - 🔑 Credentials and authentication material
>     
> - 🕸️ Relationships between AD objects
>     
> - 🎯 Potential lateral and vertical movement paths
>     
> 
> The major Windows-side tools covered here are **ActiveDirectory PowerShell**, **PowerView**, **SharpView**, **Snaffler**, **SharpHound**, and **BloodHound**.

---

# 💻 2. Command Cheat Sheet

## 🧰 ActiveDirectory PowerShell Module

```powershell
Get-Module
Import-Module ActiveDirectory

Get-ADDomain

Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

Get-ADTrust -Filter *

Get-ADGroup -Filter *
Get-ADGroup -Identity "Backup Operators"
Get-ADGroupMember -Identity "Backup Operators"
```

## 🧭 PowerView

```powershell
Import-Module .\PowerView.ps1

Get-DomainUser -Identity mmorgan -Domain inlanefreight.local | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol

Get-DomainGroupMember -Identity "Domain Admins" -Recurse

Get-DomainTrustMapping

Test-AdminAccess -ComputerName ACADEMY-EA-MS01

Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

## 🪟 SharpView

```powershell
.\SharpView.exe Get-DomainUser -Help

.\SharpView.exe Get-DomainUser -Identity forend
```

## 📂 Snaffler

```powershell
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data

.\Snaffler.exe -d INLANEFREIGHT.LOCAL -s -v data
```

## 🩸 SharpHound

```powershell
.\SharpHound.exe --help

.\SharpHound.exe -c All --zipfilename ILFREIGHT
```

## 🩸 BloodHound

```text
bloodhound
```

Useful built-in Analysis queries mentioned:

```text
Find Computers with Unsupported Operating Systems
Find Computers where Domain Users are Local Admin
```

---

# 🧭 3. Tool Roles & Differences

|Tool|Main purpose|Mental model|
|---|---|---|
|[ActiveDirectory PowerShell module](https://docs.microsoft.com/en-us/powershell/module/activedirectory/)|Native PowerShell-based AD enumeration and administration|🧰 Ask AD directly|
|[PowerView](https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon)|Broad AD reconnaissance and manual relationship discovery|🔎 Investigate AD manually|
|SharpView|.NET port of PowerView functionality|🪟 PowerView without depending on PowerShell|
|[Snaffler](https://github.com/SnaffCon/Snaffler)|Hunts accessible shares and files for sensitive information|📂 Search what the user can read|
|[SharpHound](https://github.com/SpecterOps/SharpHound)|Collects AD relationship/attack-path data|🕸️ Collect graph data|
|[BloodHound](https://github.com/SpecterOps/BloodHound)|Visualizes and analyzes AD relationships and attack paths|🧠 Understand the graph|

## 🧠 The Big Picture

```text
                         WINDOWS HOST
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ↓               ↓                ↓
       ActiveDirectory     PowerView       SharpView
          Module               │                │
              │                └──────┬─────────┘
              │                       │
              └───────────┬───────────┘
                          ↓
                 AD ENUMERATION DATA
                          │
             ┌────────────┴────────────┐
             │                         │
             ↓                         ↓
         Snaffler                  SharpHound
             │                         │
      FILE / CREDENTIALS          RELATIONSHIPS
             │                         │
             │                         ↓
             │                    BloodHound
             │                         │
             └──────────────┬──────────┘
                            ↓
                    CORRELATE INFORMATION
                            ↓
                    IDENTIFY WEAKNESSES
                            ↓
                   LATERAL / VERTICAL
                       MOVEMENT
```

> [!important]  
> **SharpHound and BloodHound are not alternatives.**
> 
> ```text
> SharpHound
>     ↓
> COLLECTS
>     ↓
> AD relationship data
>     ↓
> BloodHound
>     ↓
> ANALYZES / VISUALIZES
>     ↓
> Attack paths
> ```
> 
> Likewise, **Snaffler is not another BloodHound**. It answers a different question:
> 
> ```text
> BloodHound → "How are AD objects connected?"
> Snaffler    → "What sensitive files can I access?"
> ```

---

## 🔎 3.1 What Each Tool Is For

### 🛠️ ActiveDirectory PowerShell Module

**What it is:**  
A Microsoft PowerShell module containing AD-specific cmdlets for administering and querying Active Directory.

**Primary purpose:**  
Perform direct AD queries using native PowerShell functionality.

**Examples:**

```powershell
Get-ADDomain
Get-ADUser
Get-ADGroup
Get-ADGroupMember
Get-ADTrust
```

**Mental model:**

```text
ActiveDirectory Module
        ↓
Ask Active Directory specific questions
        ↓
Users / Groups / Domain / Trusts / Objects
```

> [!tip]  
> This is particularly useful when you land on a Windows system where the module is already available. You may not need to drop a separate offensive tool onto the host.

---

### 🛠️ PowerView

**What it is:**  
A PowerShell-based AD reconnaissance tool.

**Primary purpose:**  
Provide broader and more security-assessment-oriented enumeration than simply using individual native AD cmdlets.

PowerView can enumerate:

- Users
    
- Computers
    
- Groups
    
- OUs
    
- ACLs
    
- Trusts
    
- GPOs
    
- Sessions
    
- Local groups
    
- Shares
    
- Local administrator access
    
- SPNs
    
- File shares
    
- Foreign users/groups
    
- Domain relationships
    

**Mental model:**

```text
PowerView
    ↓
Broad AD reconnaissance
    ↓
Collect lots of individual pieces of information
    ↓
Manually correlate them
```

> [!important]  
> PowerView often requires **more manual analysis than BloodHound**, but this manual approach can expose subtle configuration issues that may not be immediately obvious from a graph.

---

### 🛠️ SharpView

**What it is:**  
A **.NET port of PowerView**.

**Mental model:**

```text
PowerView
   ↓
PowerShell implementation
   ↓
SharpView
   ↓
.NET implementation of similar functionality
```

It provides many of the same enumeration capabilities while being useful in situations where relying on PowerShell is undesirable or restricted.

---

### 🛠️ Snaffler

**What it is:**  
A tool for hunting accessible network shares and files for potentially sensitive information.

**Primary purpose:**

```text
Domain computers
      ↓
Accessible shares
      ↓
Readable directories
      ↓
Interesting files
      ↓
Potential credentials / keys / configuration
```

It requires a domain-joined host or a domain-user context.

---

### 🛠️ SharpHound

**What it is:**  
The data collector for BloodHound.

**Primary purpose:**  
Collect information about relationships and permissions in the AD environment.

It can collect information concerning:

- Groups
    
- Local administrator relationships
    
- Sessions
    
- Logged-on users
    
- Trusts
    
- ACLs
    
- Containers
    
- RDP
    
- DCOM
    
- SPN targets
    
- PowerShell Remoting
    
- GPO-related relationships
    

**Mental model:**

```text
SharpHound
    ↓
COLLECT
    ↓
Relationship data
    ↓
JSON dataset
    ↓
BloodHound
```

---

### 🛠️ BloodHound

**What it is:**  
An open-source AD relationship-analysis and visualization platform.

**Primary purpose:**  
Take collected AD information and make relationships and potential attack paths easier to understand.

**Mental model:**

```text
Raw AD information
        ↓
Relationships
        ↓
Graph
        ↓
BloodHound
        ↓
"What connects to what?"
        ↓
Potential attack paths
```

---

# 🎯 4. The Core Methodology

Credentialed enumeration is about moving from:

```text
"I have valid credentials."
          ↓
"What does this account have access to?"
          ↓
"What exists in the domain?"
          ↓
"What relationships exist?"
          ↓
"What sensitive information can I access?"
          ↓
"What systems can I administer?"
          ↓
"What accounts have elevated privileges?"
          ↓
"What trusts connect this domain to others?"
          ↓
"What attack paths can I identify?"
```

> [!info]  
> **Credentialed enumeration is reconnaissance with context.**
> 
> An unauthenticated scan might tell you that a machine exists.
> 
> A valid domain account can potentially tell you:
> 
> - which groups it belongs to,
>     
> - which shares it can access,
>     
> - where administrators are logged in,
>     
> - which machines it can administer,
>     
> - which trusts exist,
>     
> - which service accounts exist,
>     
> - which accounts have SPNs,
>     
> - and how AD objects relate to each other.
>     

---

# 🧰 5. ActiveDirectory PowerShell Module

## 🔎 5.1 Discovering Available Modules

The first step is determining what PowerShell modules are available.

```powershell
Get-Module
```

Example:

```text
        powershell
PS C:\htb> Get-Module

ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Manifest   3.1.0.0    Microsoft.PowerShell.Utility        {Add-Member, Add-Type, Clear-Variable, Compare-Object...}
Script     2.0.0      PSReadline                          {Get-PSReadLineKeyHandler, Get-PSReadLineOption, Remove-PS...
```

`Get-Module` can show available/loaded modules, versions, and exported commands.

> [!tip]  
> This is useful beyond AD enumeration. When landing on a Windows host, checking installed modules can reveal useful capabilities, including Git modules or custom administrator scripts.

---

## 📥 5.2 Loading the ActiveDirectory Module

If the module is not loaded:

```powershell
Import-Module ActiveDirectory
```

Then:

```powershell
Get-Module
```

Example:

```text
        powershell
PS C:\htb> Import-Module ActiveDirectory
PS C:\htb> Get-Module

ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Manifest   1.0.1.0    ActiveDirectory                     {Add-ADCentralAccessPolicyMember, Add-ADComputerServiceAcc...
Manifest   3.1.0.0    Microsoft.PowerShell.Utility        {Add-Member, Add-Type, Clear-Variable, Compare-Object...}
Script     2.0.0      PSReadline                          {Get-PSReadLineKeyHandler, Get-PSReadLineOption, Remove-PS...
```

---

# 🌐 6. Domain Enumeration

## 🔎 6.1 Get-ADDomain

```powershell
Get-ADDomain
```

This returns information about the current AD domain.

Important fields include:

|Field|What it tells you|
|---|---|
|`DNSRoot`|DNS name of the domain|
|`NetBIOSName`|NetBIOS domain name|
|`DomainSID`|Security identifier of the domain|
|`DomainMode`|Domain functional level|
|`Forest`|Forest containing the domain|
|`ChildDomains`|Child domains|
|`DomainControllersContainer`|Location of domain controller objects|
|`PDCEmulator`|DC holding the PDC Emulator role|
|`RIDMaster`|DC holding the RID Master role|
|`InfrastructureMaster`|DC holding the Infrastructure Master role|
|`ReplicaDirectoryServers`|Domain controllers participating in replication|

Example:

```text
PS C:\htb> Get-ADDomain

AllowedDNSSuffixes                 : {}
ChildDomains                       : {LOGISTICS.INLANEFREIGHT.LOCAL}
ComputersContainer                 : CN=Computers,DC=INLANEFREIGHT,DC=LOCAL
DeletedObjectsContainer            : CN=Deleted Objects,DC=INLANEFREIGHT,DC=LOCAL
DistinguishedName                  : DC=INLANEFREIGHT,DC=LOCAL
DNSRoot                            : INLANEFREIGHT.LOCAL
DomainControllersContainer         : OU=Domain Controllers,DC=INLANEFREIGHT,DC=LOCAL
DomainMode                         : Windows2016Domain
DomainSID                          : S-1-5-21-3842939050-3880317879-2865463114
ForeignSecurityPrincipalsContainer : CN=ForeignSecurityPrincipals,DC=INLANEFREIGHT,DC=LOCAL
Forest                             : INLANEFREIGHT.LOCAL
InfrastructureMaster               : ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
LastLogonReplicationInterval       :
LinkedGroupPolicyObjects           : {cn={DDBB8574-E94E-4525-8C9D-ABABE31223D0},cn=policies,cn=system,DC=INLANEFREIGHT,
                                     DC=LOCAL, CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=INLAN
                                     EFREIGHT,DC=LOCAL}
LostAndFoundContainer              : CN=LostAndFound,DC=INLANEFREIGHT,DC=LOCAL
ManagedBy                          :
Name                               : INLANEFREIGHT
NetBIOSName                        : INLANEFREIGHT
ObjectClass                        : domainDNS
ObjectGUID                         : 71e4ecd1-a9f6-4f55-8a0b-e8c398fb547a
ParentDomain                       :
PDCEmulator                        : ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
PublicKeyRequiredPasswordRolling   : True
QuotasContainer                    : CN=NTDS Quotas,DC=INLANEFREIGHT,DC=LOCAL
ReadOnlyReplicaDirectoryServers    : {}
ReplicaDirectoryServers            : {ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL}
RIDMaster                          : ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
SubordinateReferences              : {DC=LOGISTICS,DC=INLANEFREIGHT,DC=LOCAL,
                                     DC=ForestDnsZones,DC=INLANEFREIGHT,DC=LOCAL,
                                     DC=DomainDnsZones,DC=INLANEFREIGHT,DC=LOCAL,
                                     CN=Configuration,DC=INLANEFREIGHT,DC=LOCAL}
SystemsContainer                   : CN=System,DC=INLANEFREIGHT,DC=LOCAL
UsersContainer                     : CN=Users,DC=INLANEFREIGHT,DC=LOCAL
```

> [!important]  
> `ChildDomains` immediately tells us that `LOGISTICS.INLANEFREIGHT.LOCAL` exists beneath the current domain.

---

# 🔐 7. Finding Accounts With SPNs

## 🎯 Get-ADUser + ServicePrincipalName

```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

This filters users whose `ServicePrincipalName` attribute is populated.

Example:

```text
DistinguishedName    : CN=adfs,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
Enabled              : True
GivenName            : Sharepoint
Name                 : adfs
ObjectClass          : user
ObjectGUID           : 49b53bea-4bc4-4a68-b694-b806d9809e95
SamAccountName       : adfs
ServicePrincipalName : {adfsconnect/azure01.inlanefreight.local}
SID                  : S-1-5-21-3842939050-3880317879-2865463114-5244
Surname              : Admin
UserPrincipalName    :

DistinguishedName    : CN=BACKUPAGENT,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
Enabled              : True
GivenName            : Jessica
Name                 : BACKUPAGENT
ObjectClass          : user
ObjectGUID           : 2ec53e98-3a64-4706-be23-1d824ff61bed
SamAccountName       : backupagent
ServicePrincipalName : {backupjob/veam001.inlanefreight.local}
SID                  : S-1-5-21-3842939050-3880317879-2865463114-5220
Surname              : Systemmailbox 8Cc370d3-822A-4Ab8-A926-Bb94bd0641a9
UserPrincipalName    :
```

> [!important]  
> A user account with an SPN can be a **potential Kerberoasting target**.
> 
> This command **does not perform Kerberoasting**. It identifies accounts whose attributes indicate that they may be relevant to the technique.

```text
LDAP / AD
   ↓
Find user accounts
   ↓
Filter ServicePrincipalName != null
   ↓
Potential service accounts
   ↓
Investigate
   ↓
Potential Kerberoasting target
```

---

# 🧬 8. Trust Enumeration

## 🔎 Get-ADTrust

```powershell
Get-ADTrust -Filter *
```

Example:

```text
Direction               : BiDirectional
DisallowTransivity      : False
DistinguishedName       : CN=LOGISTICS.INLANEFREIGHT.LOCAL,CN=System,DC=INLANEFREIGHT,DC=LOCAL
ForestTransitive        : False
IntraForest             : True
IsTreeParent            : False
IsTreeRoot              : False
Name                    : LOGISTICS.INLANEFREIGHT.LOCAL
ObjectClass              : trustedDomain
ObjectGUID               : f48a1169-2e58-42c1-ba32-a6ccb10057ec
SelectiveAuthentication : False
SIDFilteringForestAware : False
SIDFilteringQuarantined : False
Source                  : DC=INLANEFREIGHT,DC=LOCAL
Target                  : LOGISTICS.INLANEFREIGHT.LOCAL
TGTDelegation           : False
TrustAttributes         : 32
TrustedPolicy           :
TrustingPolicy          :
TrustType               : Uplevel
UplevelOnly             : False
UsesAESKeys              : False
UsesRC4Encryption       : False

Direction               : BiDirectional
DisallowTransivity      : False
DistinguishedName       : CN=FREIGHTLOGISTICS.LOCAL,CN=System,DC=INLANEFREIGHT,DC=LOCAL
ForestTransitive        : True
IntraForest             : False
IsTreeParent            : False
IsTreeRoot              : False
Name                    : FREIGHTLOGISTICS.LOCAL
ObjectClass              : trustedDomain
ObjectGUID              : 1597717f-89b7-49b8-9cd9-0801d52475ca
SelectiveAuthentication : False
SIDFilteringForestAware : False
SIDFilteringQuarantined : False
Source                  : DC=INLANEFREIGHT,DC=LOCAL
Target                  : FREIGHTLOGISTICS.LOCAL
TGTDelegation           : False
TrustAttributes         : 8
TrustedPolicy            :
TrustingPolicy           :
TrustType               : Uplevel
UplevelOnly             : False
UsesAESKeys              : False
UsesRC4Encryption       : False
```

### 🧠 Important Trust Fields

|Field|Meaning|
|---|---|
|`Direction`|Direction of the trust relationship|
|`IntraForest`|Indicates a trust within the same forest|
|`ForestTransitive`|Indicates a forest-transitive trust|
|`Source`|Domain establishing the trust relationship|
|`Target`|Domain on the other side|
|`TrustType`|Type of trust|
|`SIDFiltering...`|Information about SID filtering|
|`SelectiveAuthentication`|Whether selective authentication is configured|

> [!important]  
> Trust enumeration expands the scope of our mental model.
> 
> We are no longer asking only:
> 
> **"What exists inside this domain?"**
> 
> We are also asking:
> 
> **"What other domains can this environment communicate with through trust relationships?"**

```text
CURRENT DOMAIN
      │
      ├── Child Domain
      │      └── LOGISTICS.INLANEFREIGHT.LOCAL
      │
      └── External / Other Forest
             └── FREIGHTLOGISTICS.LOCAL
```

---

# 👥 9. Group Enumeration

## 🔎 List All Groups

```powershell
Get-ADGroup -Filter * | select name
```

Example:

```text
name
----
Administrators
Users
Guests
Print Operators
Backup Operators
Replicator
Remote Desktop Users
Network Configuration Operators
Performance Monitor Users
Performance Log Users
Distributed COM Users
IIS_IUSRS
Cryptographic Operators
Event Log Readers
Certificate Service DCOM Access
RDS Remote Access Servers
RDS Endpoint Servers
RDS Management Servers
Hyper-V Administrators
Access Control Assistance Operators
Remote Management Users
Storage Replica Administrators
Domain Computers
Domain Controllers
Schema Admins
Enterprise Admins
Cert Publishers
Domain Admins

<SNIP>
```

This provides the **group inventory**.

---

## 🔎 9.1 Detailed Group Information

Once an interesting group is identified:

```powershell
Get-ADGroup -Identity "Backup Operators"
```

Example:

```text
DistinguishedName : CN=Backup Operators,CN=Builtin,DC=INLANEFREIGHT,DC=LOCAL
GroupCategory     : Security
GroupScope        : DomainLocal
Name              : Backup Operators
ObjectClass       : group
ObjectGUID        : 6276d85d-9c39-4b7c-8449-cad37e8abc38
SamAccountName    : Backup Operators
SID               : S-1-5-32-551
```

---

## 👤 9.2 Enumerating Group Members

```powershell
Get-ADGroupMember -Identity "Backup Operators"
```

Example:

```text
distinguishedName : CN=BACKUPAGENT,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
name              : BACKUPAGENT
objectClass       : user
objectGUID        : 2ec53e98-3a64-4706-be23-1d824ff61bed
SamAccountName    : backupagent
SID               : S-1-5-21-3842939050-3880317879-2865463114-5220
```

This reveals that:

```text
backupagent
    ↓
Backup Operators
    ↓
Potentially elevated capabilities
```

> [!warning]  
> Group membership alone does not automatically mean that an account can immediately compromise the domain. The **effective permissions and privileges** granted by the group must be understood in context.

---

# 🧠 10. Why Group Enumeration Matters

The methodology is:

```text
LIST GROUPS
    ↓
IDENTIFY INTERESTING GROUP
    ↓
ENUMERATE MEMBERS
    ↓
CHECK NESTED MEMBERSHIP
    ↓
UNDERSTAND EFFECTIVE PRIVILEGES
    ↓
IDENTIFY ACCOUNTS WORTH INVESTIGATING
```

The same process can be repeated against other groups.

The problem is scale.

> [!info]  
> In a large domain, manually running commands for every group can generate an enormous amount of information. This is one of the reasons specialized tools such as **PowerView, SharpHound, and BloodHound** become valuable.

---

# 🔎 11. PowerView

[PowerView](https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon) is a PowerShell-based tool designed for gaining **situational awareness in Active Directory**.

It can enumerate:

- 👤 Users
    
- 🖥️ Computers
    
- 👥 Groups
    
- 📜 ACLs
    
- 🧬 Trusts
    
- 📂 Shares
    
- 🔑 Password-related information
    
- 🎫 SPNs
    
- 👨‍💻 Logged-on users
    
- 🛡️ Local administrator access
    

> [!important]  
> The major difference from BloodHound is the **analysis model**.
> 
> **PowerView** gives us many individual pieces of information that we manually correlate.
> 
> **BloodHound** is designed around collecting relationships and visualizing those relationships as a graph.

---

## 📊 11.1 PowerView Functions

|**CommandDescription**||
|---|---|
|`Export-PowerViewCSV`|Append results to a CSV file|
|`ConvertTo-SID`|Convert a User or group name to its SID value|
|`Get-DomainSPNTicket`|Requests the Kerberos ticket for a specified Service Principal Name (SPN) account|
|**Domain/LDAP Functions:**||
|`Get-Domain`|Will return the AD object for the current (or specified) domain|
|`Get-DomainController`|Return a list of the Domain Controllers for the specified domain|
|`Get-DomainUser`|Will return all users or specific user objects in AD|
|`Get-DomainComputer`|Will return all computers or specific computer objects in AD|
|`Get-DomainGroup`|Will return all groups or specific group objects in AD|
|`Get-DomainOU`|Search for all or specific OU objects in AD|
|`Find-InterestingDomainAcl`|Finds object ACLs in the domain with modification rights set to non-built in objects|
|`Get-DomainGroupMember`|Will return the members of a specific domain group|
|`Get-DomainFileServer`|Returns a list of servers likely functioning as file servers|
|`Get-DomainDFSShare`|Returns a list of all distributed file systems for the current (or specified) domain|
|**GPO Functions:**||
|`Get-DomainGPO`|Will return all GPOs or specific GPO objects in AD|
|`Get-DomainPolicy`|Returns the default domain policy or the domain controller policy for the current domain|
|**Computer Enumeration Functions:**||
|`Get-NetLocalGroup`|Enumerates local groups on the local or a remote machine|
|`Get-NetLocalGroupMember`|Enumerates members of a specific local group|
|`Get-NetShare`|Returns open shares on the local (or a remote) machine|
|`Get-NetSession`|Will return session information for the local (or a remote) machine|
|`Test-AdminAccess`|Tests if the current user has administrative access to the local (or a remote) machine|
|**Threaded 'Meta'-Functions:**||
|`Find-DomainUserLocation`|Finds machines where specific users are logged in|
|`Find-DomainShare`|Finds reachable shares on domain machines|
|`Find-InterestingDomainShareFile`|Searches for files matching specific criteria on readable shares in the domain|
|`Find-LocalAdminAccess`|Find machines on the local domain where the current user has local administrator access|
|**Domain Trust Functions:**||
|`Get-DomainTrust`|Returns domain trusts for the current domain or a specified domain|
|`Get-ForestTrust`|Returns all forest trusts for the current forest or a specified forest|
|`Get-DomainForeignUser`|Enumerates users who are in groups outside of the user's domain|
|`Get-DomainForeignGroupMember`|Enumerates groups with users outside of the group's domain and returns each foreign member|
|`Get-DomainTrustMapping`|Will enumerate all trusts for the current domain and any others seen.|

> [!tip]  
> Think of the PowerView functions as specialized questions you can ask the domain:
> 
> ```text
> Get-DomainUser              → Who are the users?
> Get-DomainGroup             → What groups exist?
> Get-DomainGroupMember       → Who belongs to this group?
> Get-DomainComputer          → What computers exist?
> Get-DomainTrust             → What trusts exist?
> Get-DomainTrustMapping      → What does the trust topology look like?
> Get-NetShare                → What shares exist?
> Test-AdminAccess            → Where am I admin?
> Get-DomainUser -SPN         → Which users have SPNs?
> ```

---

# 👤 12. PowerView User Enumeration

Import PowerView:

```powershell
cd C:\Tools\
Import-Module .\PowerView.ps1
```

Then query a specific user:

```powershell
Get-DomainUser -Identity mmorgan -Domain inlanefreight.local | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol
```

Example:

```text
name                 : Matthew Morgan
samaccountname       : mmorgan
description          :
memberof             : {CN=VPN Users,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL, CN=Shared Calendar
                       Read,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL, CN=Printer Access,OU=Security
                       Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL, CN=File Share H Drive,OU=Security
                       Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL...}
whencreated          : 10/27/2021 5:37:06 PM
pwdlastset           : 11/18/2021 10:02:57 AM
lastlogontimestamp   : 2/27/2022 6:34:25 PM
accountexpires       : NEVER
admincount           : 1
userprincipalname    : mmorgan@inlanefreight.local
serviceprincipalname :
mail                 :
useraccountcontrol   : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
```

### 🔎 Important Attributes

|Attribute|Why it matters|
|---|---|
|`samaccountname`|Account's legacy/Windows logon name|
|`memberof`|Groups the user directly belongs to|
|`pwdlastset`|Password last-set information|
|`lastlogontimestamp`|Approximate last logon information|
|`accountexpires`|Account expiration|
|`admincount`|Indicates special protection associated with privileged accounts/groups|
|`userprincipalname`|User's UPN|
|`serviceprincipalname`|SPNs associated with the account|
|`useraccountcontrol`|Account configuration flags|

> [!warning]  
> `admincount : 1` should be treated as an **interesting indicator**, not automatically interpreted as "this user is currently a Domain Admin." It indicates that the account has been associated with protected administrative groups and requires further enumeration.

---

# 👥 13. Recursive Group Membership

```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

The `-Recurse` switch is important because groups can contain **other groups**.

Example:

```text
GroupDomain             : INLANEFREIGHT.LOCAL
GroupName               : Domain Admins
GroupDistinguishedName  : CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
MemberDomain            : INLANEFREIGHT.LOCAL
MemberName              : svc_qualys
MemberDistinguishedName : CN=svc_qualys,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
MemberObjectClass       : user
MemberSID               : S-1-5-21-3842939050-3880317879-2865463114-5613

GroupDomain             : INLANEFREIGHT.LOCAL
GroupName               : Domain Admins
GroupDistinguishedName  : CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
MemberDomain             : INLANEFREIGHT.LOCAL
MemberName              : sp-admin
MemberDistinguishedName : CN=Sharepoint Admin,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
MemberObjectClass       : user
MemberSID               : S-1-5-21-3842939050-3880317879-2865463114-5228

GroupDomain             : INLANEFREIGHT.LOCAL
GroupName               : Secadmins
GroupDistinguishedName  : CN=Secadmins,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
MemberDomain            : INLANEFREIGHT.LOCAL
MemberName              : spong1990
MemberDistinguishedName : CN=Maggie
                          Jablonski,OU=Operations,OU=Logistics-HK,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
MemberObjectClass       : user
MemberSID               : S-1-5-21-3842939050-3880317879-2865463114-1965

<SNIP>
```

### 🧠 Nested Group Mental Model

```text
Domain Admins
      │
      ├── svc_qualys
      │
      ├── sp-admin
      │
      └── Secadmins
             │
             └── spong1990
```

Therefore:

```text
spong1990
    ↓
member of Secadmins
    ↓
Secadmins nested in Domain Admins
    ↓
effective Domain Admin membership
```

> [!important]  
> **Nested groups are why simple direct-membership enumeration can be misleading.**
> 
> Always distinguish:
> 
> ```text
> Direct membership
>       vs
> Effective membership
> ```
> 
> `-Recurse` helps expose this relationship.

---

# 🧬 14. PowerView Trust Mapping

```powershell
Get-DomainTrustMapping
```

Example:

```text
SourceName      : INLANEFREIGHT.LOCAL
TargetName      : LOGISTICS.INLANEFREIGHT.LOCAL
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : WITHIN_FOREST
TrustDirection  : Bidirectional
WhenCreated     : 11/1/2021 6:20:22 PM
WhenChanged     : 2/26/2022 11:55:55 PM

SourceName      : INLANEFREIGHT.LOCAL
TargetName      : FREIGHTLOGISTICS.LOCAL
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : FOREST_TRANSITIVE
TrustDirection  : Bidirectional
WhenCreated     : 11/1/2021 8:07:09 PM
WhenChanged     : 2/27/2022 12:02:39 AM

SourceName      : LOGISTICS.INLANEFREIGHT.LOCAL
TargetName      : INLANEFREIGHT.LOCAL
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : WITHIN_FOREST
TrustDirection  : Bidirectional
WhenCreated     : 11/1/2021 6:20:22 PM
WhenChanged     : 2/26/2022 11:55:55 PM
```

This is particularly useful for building the **domain/forest topology**.

```text
INLANEFREIGHT.LOCAL
       │
       │ WITHIN_FOREST
       ↓
LOGISTICS.INLANEFREIGHT.LOCAL

       │
       │ FOREST_TRANSITIVE
       ↓
FREIGHTLOGISTICS.LOCAL
```

---

# 🛡️ 15. Testing Local Administrator Access

```powershell
Test-AdminAccess -ComputerName ACADEMY-EA-MS01
```

Output:

```text
ComputerName    IsAdmin
------------    -------
ACADEMY-EA-MS01    True
```

This answers:

> **"Does my current context have local administrative access to this computer?"**

The command can be repeated against multiple hosts.

### 🧠 Why This Matters

```text
VALID CREDENTIAL
      ↓
LOCAL ADMIN ACCESS
      ↓
REMOTE HOST
      ↓
POTENTIAL LATERAL MOVEMENT
      ↓
ENUMERATE THAT HOST
```

> [!important]  
> Local administrator access is different from domain administrator membership.
> 
> ```text
> Domain Admin
>     ↓
> Domain-level privileged group membership
> 
> Local Admin
>     ↓
> Administrative control over a particular host
> ```
> 
> A normal domain user can potentially have local administrator rights on one or more individual machines.

---

# 🎫 16. Finding Users With SPNs Using PowerView

```powershell
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

Output:

```text
serviceprincipalname                          samaccountname
--------------------                          --------------
adfsconnect/azure01.inlanefreight.local       adfs
backupjob/veam001.inlanefreight.local         backupagent
d0wngrade/kerberoast.inlanefreight.local      d0wngrade
kadmin/changepw                               krbtgt
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433 sqldev
MSSQLSvc/SPSJDB.inlanefreight.local:1433      sqlprod
MSSQLSvc/SQL-CL01-01inlanefreight.local:49351 sqlqa
sts/inlanefreight.local                       solarwindsmonitor
testspn/kerberoast.inlanefreight.local        testspn
testspn2/kerberoast.inlanefreight.local       testspn2
```

> [!important]  
> This is an **enumeration step**.
> 
> ```text
> Get-DomainUser -SPN
>          ↓
> Identify accounts with SPNs
>          ↓
> Investigate service accounts
>          ↓
> Potential Kerberoasting target
> ```
> 
> The command itself does not constitute the complete Kerberoasting attack.

---

# 🪟 17. SharpView

[SharpView](https://github.com/tevora-threat/SharpView) is a **.NET implementation/port of PowerView functionality**.

The purpose is largely similar:

```text
PowerView
   │
   │ PowerShell
   ↓
AD reconnaissance

SharpView
   │
   │ .NET
   ↓
AD reconnaissance
```

> [!info]  
> The original PowerSploit version of PowerView was deprecated, while the source material discusses the BC-Security/Empire development version. SharpView provides another way to perform similar reconnaissance functionality.

---

## 🔎 17.1 SharpView Help

```powershell
.\SharpView.exe Get-DomainUser -Help
```

Output:

```text
Get_DomainUser -Identity <String[]> -DistinguishedName <String[]> -SamAccountName <String[]> -Name <String[]> -MemberDistinguishedName <String[]> -MemberName <String[]> -SPN <Boolean> -AdminCount <Boolean> -AllowDelegation <Boolean> -DisallowDelegation <Boolean> -TrustedToAuth <Boolean> -PreauthNotRequired <Boolean> -KerberosPreauthNotRequired <Boolean> -NoPreauth <Boolean> -Domain <String> -LDAPFilter <String> -Filter <String> -Properties <String[]> -SearchBase <String> -ADSPath <String> -Server <String> -DomainController <String> -SearchScope <SearchScope> -ResultPageSize <Int32> -ServerTimeLimit <Nullable`1> -SecurityMasks <Nullable`1> -Tombstone <Boolean> -FindOne <Boolean> -ReturnOne <Boolean> -Credential <NetworkCredential> -Raw <Boolean> -UACFilter <UACEnum>
```

---

## 👤 17.2 Enumerating a User

```powershell
.\SharpView.exe Get-DomainUser -Identity forend
```

Output:

```text
[Get-DomainSearcher] search base: LDAP://ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL/DC=INLANEFREIGHT,DC=LOCAL
[Get-DomainUser] filter string: (&(samAccountType=805306368)(|(samAccountName=forend)))
objectsid                      : {S-1-5-21-3842939050-3880317879-2865463114-5614}
samaccounttype                 : USER_OBJECT
objectguid                     : 53264142-082a-4cb8-8714-8158b4974f3b
useraccountcontrol             : NORMAL_ACCOUNT
accountexpires                 : 12/31/1600 4:00:00 PM
lastlogon                      : 4/18/2022 1:01:21 PM
lastlogontimestamp             : 4/9/2022 1:33:21 PM
pwdlastset                     : 2/28/2022 12:03:45 PM
lastlogoff                     : 12/31/1600 4:00:00 PM
badPasswordTime                : 4/5/2022 7:09:07 AM
name                           : forend
distinguishedname              : CN=forend,OU=IT Admins,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
whencreated                    : 2/28/2022 8:03:45 PM
whenchanged                    : 4/9/2022 8:33:21 PM
samaccountname                 : forend
memberof                       : {CN=VPN Users,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL, CN=Shared Calendar Read,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL, CN=Printer Access,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL, CN=File Share H Drive,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL, CN=File Share G Drive,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL}
cn                             : {forend}
objectclass                    : {top, person, organizationalPerson, user}
badpwdcount                    : 0
countrycode                    : 0
usnchanged                     : 3259288
logoncount                     : 26618
primarygroupid                 : 513
objectcategory                 : CN=Person,CN=Schema,CN=Configuration,DC=INLANEFREIGHT,DC=LOCAL
dscorepropagationdata          : {3/24/2022 3:58:07 PM, 3/24/2022 3:57:44 PM, 3/24/2022 3:52:58 PM, 3/24/2022 3:49:31 PM, 7/14/1601 10:36:49:49:49 PM}
usncreated                     : 3054181
instancetype                   : 4
codepage                       : 0
```

> [!tip]  
> SharpView is especially useful to understand when you encounter environments where PowerShell use is hardened or restricted. Functionally, the goal remains the same: **enumerate AD information**.

---

# 📂 18. Shares

Network shares are an important part of credentialed enumeration because a standard domain account may have access to significantly more information than expected.

Potentially sensitive information may include:

- 🔑 Passwords
    
- 🔐 SSH keys
    
- ⚙️ Configuration files
    
- 🗄️ Database files
    
- 🧾 Backups
    
- 🧑‍💼 HR/personnel information
    
- 🏥 Medical information
    
- 🔧 IT infrastructure information
    

The assessment question becomes:

> **"What information can this compromised domain account access through network shares?"**

---

## 🧠 Share Enumeration Mental Model

```text
COMPROMISED DOMAIN USER
          ↓
   ACCESSIBLE SHARES
          ↓
   READABLE DIRECTORIES
          ↓
     INTERESTING FILES
          ↓
 ┌────────┴─────────┐
 ↓                  ↓
Credentials       Configuration
 ↓                  ↓
More access       More knowledge
        \          /
         ↓        ↓
          Further Access
```

> [!warning]  
> Not every accessible file is an attack path. Some findings may primarily represent **data exposure** or compliance/security-posture issues and may still be important to report.

---

# 🧲 19. Snaffler

[Snaffler](https://github.com/SnaffCon/Snaffler) automates much of the tedious process of hunting through accessible domain shares.

It:

```text
1. Obtains a list of hosts
          ↓
2. Enumerates shares
          ↓
3. Identifies readable directories
          ↓
4. Searches accessible files
          ↓
5. Looks for potentially sensitive data
```

Snaffler requires a **domain-joined host or domain-user context**.

---

## ▶️ 19.1 Snaffler Execution

```bash
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
```

### Important Options

|Option|Purpose|
|---|---|
|`-s`|Print results to the console|
|`-d`|Specify the domain to search|
|`-o`|Write results to a logfile|
|`-v`|Set verbosity level|

The source notes that `data` is typically useful for displaying results while keeping the output manageable.

> [!tip]  
> Snaffler can produce a **large amount of output**. Saving results to a file lets the tool run while you return to the results later.

---

## 🔎 19.2 Snaffler in Action

```powershell
.\Snaffler.exe  -d INLANEFREIGHT.LOCAL -s -v data
```

Example output:

```text
 .::::::.:::.    :::.  :::.    .-:::::'.-:::::':::    .,:::::: :::::::..
;;;`    ``;;;;,  `;;;  ;;`;;   ;;;'''' ;;;'''' ;;;    ;;;;'''' ;;;;``;;;;
'[==/[[[[, [[[[[. '[[ ,[[ '[[, [[[,,== [[[,,== [[[     [[cccc   [[[,/[[['
  '''    $ $$$ 'Y$c$$c$$$cc$$$c`$$$'`` `$$$'`` $$'     $$""   $$$$$$c
 88b    dP 888    Y88 888   888,888     888   o88oo,.__888oo,__ 888b '88bo,
  'YMmMY'  MMM     YM YMM   ''` 'MM,    'MM,  ''''YUMMM''''YUMMMMMMM   'W'
                         by l0ss and Sh3r4 - github.com/SnaffCon/Snaffler

2022-03-31 12:16:54 -07:00 [Share] {Black}(\\ACADEMY-EA-MS01.INLANEFREIGHT.LOCAL\ADMIN$)
2022-03-31 12:16:54 -07:00 [Share] {Black}(\\ACADEMY-EA-MS01.INLANEFREIGHT.LOCAL\C$)
2022-03-31 12:16:54 -07:00 [Share] {Green}(\\ACADEMY-EA-MX01.INLANEFREIGHT.LOCAL\address)
2022-03-31 12:16:54 -07:00 [Share] {Green}(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares)
2022-03-31 12:16:54 -07:00 [Share] {Green}(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\User Shares)
2022-03-31 12:16:54 -07:00 [Share] {Green}(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\ZZZ_archive)
2022-03-31 12:16:54 -07:00 [Share] {Green}(\\ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL\CertEnroll)
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.kdb$|289B|3/31/2022 12:09:22 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\GroupBackup.kdb) .kdb
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.key$|299B|3/31/2022 12:05:33 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\ShowReset.key) .key
2022-03-31 12:17:19 -07:00 [Share] {Green}(\\ACADEMY-EA-FILE.INLANEFREIGHT.LOCAL\UpdateServicesPackages)
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.kwallet$|302B|3/31/2022 12:04:45 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\WriteUse.kwallet) .kwallet
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.key$|298B|3/31/2022 12:05:10 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\ProtectStep.key) .key
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.ppk$|275B|3/31/2022 12:04:40 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\StopTrace.ppk) .ppk
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.key$|301B|3/31/2022 12:09:17 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\WaitClear.key) .key
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.sqldump$|312B|3/31/2022 12:05:30 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\DenyRedo.sqldump) .sqldump
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.sqldump$|310B|3/31/2022 12:05:02 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\AddPublish.sqldump) .sqldump
2022-03-31 12:17:19 -07:00 [Share] {Green}(\\ACADEMY-EA-FILE.INLANEFREIGHT.LOCAL\WsusContent)
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.keychain$|295B|3/31/2022 12:08:42 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\SetStep.keychain) .keychain
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.tblk$|279B|3/31/2022 12:05:25 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\FindConnect.tblk) .tblk
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.psafe3$|301B|3/31/2022 12:09:33 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\GetUpdate.psafe3) .psafe3
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.keypair$|278B|3/31/2022 12:09:09 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\UnprotectConvertTo.keypair) .keypair
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.tblk$|280B|3/31/2022 12:05:17 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\ExportJoin.tblk) .tblk
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.mdf$|305B|3/31/2022 12:09:27 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\FormatShow.mdf) .mdf
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.mdf$|299B|3/31/2022 12:09:14 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\LockConfirm.mdf) .mdf

<SNIP>
```

Snaffler can identify potentially interesting file types such as:

```text
.kdb
.key
.kwallet
.ppk
.sqldump
.keychain
.psafe3
.keypair
.mdf
```

These may represent potentially valuable authentication, configuration, database, or credential-related material depending on their contents.

> [!important]  
> **Snaffler answers a different question from BloodHound:**
> 
> ```text
> BloodHound
>     ↓
> AD relationships
>     ↓
> "What can connect to what?"
> 
> Snaffler
>     ↓
> File/share access
>     ↓
> "What sensitive information can I read?"
> ```

---

# 🩸 20. BloodHound + SharpHound

[BloodHound](https://github.com/SpecterOps/BloodHound) is used to visualize and analyze relationships between AD objects.

[SharpHound](https://github.com/SpecterOps/SharpHound) is the collector that gathers the information BloodHound needs.

## 🧠 Collector vs Analyzer

```text
                    AD ENVIRONMENT
                          │
                          ↓
                     SharpHound
                          │
                    DATA COLLECTION
                          │
                          ↓
                    JSON DATASET
                          │
                          ↓
                     BloodHound
                          │
                 GRAPH / CORRELATION
                          │
                          ↓
                  ATTACK PATH ANALYSIS
```

> [!important]  
> This distinction is one of the most important concepts in this section:
> 
> **SharpHound collects. BloodHound analyzes.**

---

# 🔧 21. SharpHound Help

```powershell
.\SharpHound.exe --help
```

Example:

```text
SharpHound 1.0.3
Copyright (C) 2022 SpecterOps

  -c, --collectionmethods    (Default: Default) Collection Methods: Container, Group, LocalGroup, GPOLocalGroup,
                             Session, LoggedOn, ObjectProps, ACL, ComputerOnly, Trusts, Default, RDP, DCOM, DCOnly

  -d, --domain               Specify domain to enumerate

  -s, --searchforest         (Default: false) Search all available domains in the forest

  --stealth                  Stealth Collection (Prefer DCOnly whenever possible!)

  -f                         Add an LDAP filter to the pregenerated filter.

  --distinguishedname        Base DistinguishedName to start the LDAP search at

  --computerfile             Path to file containing computer names to enumerate
  
  <SNIP>
```

---

# 🚀 22. Running SharpHound

```powershell
.\SharpHound.exe -c All --zipfilename ILFREIGHT
```

Example output:

```text
2022-04-18T13:58:22.1163680-07:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2022-04-18T13:58:22.1163680-07:00|INFORMATION|Initializing SharpHound at 1:58 PM on 4/18/2022
2022-04-18T13:58:22.6788709-07:00|INFORMATION|Flags: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2022-04-18T13:58:23.0851206-07:00|INFORMATION|Beginning LDAP search for INLANEFREIGHT.LOCAL
2022-04-18T13:58:53.9132950-07:00|INFORMATION|Status: 0 objects finished (+0 0)/s -- Using 67 MB RAM
2022-04-18T13:59:15.7882419-07:00|INFORMATION|Producer has finished, closing LDAP channel
2022-04-18T13:59:16.1788930-07:00|INFORMATION|LDAP channel closed, waiting for consumers
2022-04-18T13:59:23.9288698-07:00|INFORMATION|Status: 3793 objects finished (+3793 63.21667)/s -- Using 112 MB RAM
2022-04-18T13:59:45.4132561-07:00|INFORMATION|Consumers finished, closing output channel
Closing writers
2022-04-18T13:59:45.4601086-07:00|INFORMATION|Output channel closed, waiting for output task to complete
2022-04-18T13:59:45.8663528-07:00|INFORMATION|Status: 3809 objects finished (+16 46.45122)/s -- Using 110 MB RAM
2022-04-18T13:59:45.8663528-07:00|INFORMATION|Enumeration finished in 00:01:22.7919186
2022-04-18T13:59:46.3663660-07:00|INFORMATION|SharpHound Enumeration Completed at 1:59 PM on 4/18/2022! Happy Graphing
```

The resulting dataset can then be loaded into BloodHound.

---

# 🧠 23. BloodHound Analysis

Once the SharpHound data has been collected, it can be:

```text
Windows Host
     ↓
SharpHound
     ↓
ZIP / JSON dataset
     ↓
BloodHound
     ↓
Graph database
     ↓
Relationships
     ↓
Analysis
```

The source describes uploading the generated ZIP through the BloodHound GUI and then searching for:

```text
domain:
```

to select:

```text
INLANEFREIGHT.LOCAL
```

The example environment contains more than 550 hosts and trusts with two other domains.

---

# 🔎 24. BloodHound Analysis Queries

## 🖥️ Find Computers with Unsupported Operating Systems

This query can identify hosts running legacy operating systems.

The source example identifies:

- Windows 7
    
- Windows Server 2008
    

These systems may have vulnerabilities associated with outdated software.

The source specifically references **MS08-067**, an old Server Service vulnerability.

> [!warning]  
> An old operating system appearing in AD does **not automatically mean the host is currently live**.
> 
> A stale computer object can remain in Active Directory after a machine has been decommissioned.
> 
> Therefore:
> 
> ```text
> AD record
>     ↓
> Validate host
>     ↓
> Is it actually live?
>     ↓
> Only then assess exposure
> ```

This distinction matters both for exploitation decisions and accurate penetration-test reporting.

---

# 🛡️ 25. Finding Excessive Local Administrator Rights

BloodHound can identify situations where users or groups have local administrator rights over computers.

The example query is:

```text
Find Computers where Domain Users are Local Admin
```

The dangerous relationship can be visualized as:

```text
Domain Users
     │
     │ Local Admin
     ↓
Computer
```

If an account we control is a member of `Domain Users`, this relationship can potentially give that account administrative access to the affected host.

> [!important]  
> This is an example of why **relationship mapping** is powerful.
> 
> Individually:
> 
> ```text
> User exists
> Computer exists
> Domain Users exists
> ```
> 
> does not immediately tell us much.
> 
> The relationship:
> 
> ```text
> Domain Users
>       ↓
> Local Admin
>       ↓
> Computer
> ```
> 
> is what makes the finding interesting.

---

# 🧩 26. PowerView vs SharpHound vs BloodHound

|Question|Tool|
|---|---|
|"What users exist?"|ActiveDirectory / PowerView / SharpView|
|"What groups exist?"|ActiveDirectory / PowerView / SharpView|
|"Who belongs to this group?"|ActiveDirectory / PowerView / SharpView|
|"Who has an SPN?"|ActiveDirectory / PowerView / SharpView|
|"What trusts exist?"|ActiveDirectory / PowerView / SharpView|
|"Where am I local admin?"|PowerView / SharpHound + BloodHound|
|"What files can I access?"|Snaffler / PowerView|
|"What sensitive files exist on accessible shares?"|Snaffler|
|"What relationships exist between AD objects?"|SharpHound|
|"How do those relationships form attack paths?"|BloodHound|
|"What can I manually investigate in AD?"|PowerView|
|"What can I visualize and correlate?"|BloodHound|

---

# 🧠 27. The Complete Windows Credentialed Enumeration Workflow

```text
                 VALID DOMAIN CREDENTIALS
                           │
                           ↓
                 🪟 WINDOWS ATTACK HOST
                           │
            ┌──────────────┴──────────────┐
            │                             │
            ↓                             ↓
     Native AD Module                 PowerView
            │                             │
            └──────────────┬──────────────┘
                           ↓
                    DOMAIN ENUMERATION
                           │
          ┌────────────────┼─────────────────┐
          ↓                ↓                 ↓
       USERS             GROUPS            TRUSTS
          │                │                 │
          ↓                ↓                 ↓
       SPNs          Memberships       Forest/Domain
          │                │             relationships
          │                ↓                 │
          │         Privileged Groups       │
          │                │                 │
          └────────┬───────┴─────────────────┘
                   ↓
             RELATIONSHIP DATA
                   │
          ┌────────┴─────────┐
          ↓                  ↓
     SharpHound            Snaffler
          │                  │
          ↓                  ↓
    AD relationships     File/share data
          │                  │
          ↓                  ↓
      BloodHound          Sensitive data
          │                  │
          └────────┬─────────┘
                   ↓
             CORRELATE DATA
                   ↓
          IDENTIFY WEAKNESSES
                   ↓
       ┌───────────┼────────────┐
       ↓           ↓            ↓
   Local Admin   SPN/Service   Trusts
       │          Accounts       │
       ↓             │           ↓
    Lateral          ↓       Cross-domain
    Movement     Credentials    paths
       │             │           │
       └─────────────┼───────────┘
                     ↓
             FURTHER ACCESS
```

---

# 🎯 28. The CPTS Mental Model

The entire section can be remembered as **five questions**:

```text
1️⃣ WHAT EXISTS?
       ↓
   Users
   Groups
   Computers
   Domains
   OUs
   GPOs

2️⃣ WHAT CAN I ACCESS?
       ↓
   Shares
   Local admin rights
   RDP
   WinRM
   Other permissions

3️⃣ WHAT IS PRIVILEGED?
       ↓
   Admin groups
   Nested groups
   Service accounts
   SPNs
   ACLs

4️⃣ HOW IS EVERYTHING CONNECTED?
       ↓
   Trusts
   Group relationships
   Sessions
   Local admin relationships
   ACLs
   GPOs

5️⃣ WHAT CAN I DO WITH THE INFORMATION?
       ↓
   Identify attack paths
   Identify sensitive data
   Identify lateral movement opportunities
   Identify privilege escalation paths
```

---

# 🧩 29. Why We Use Multiple Tools

The tools are not redundant.

They approach the same AD environment from different angles:

```text
                     ACTIVE DIRECTORY
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
       ↓                   ↓                    ↓
   AD Module            PowerView            Snaffler
       │                   │                    │
  Native queries      Deep/manual recon     File hunting
       │                   │                    │
       └───────────┬───────┘                    │
                   ↓                            │
             AD INFORMATION                    │
                   │                            │
                   ↓                            │
              SharpHound                        │
                   │                            │
            Relationship data                   │
                   ↓                            │
              BloodHound                        │
                   │                            │
             Graph analysis                     │
                   │                            │
                   └──────────────┬─────────────┘
                                  ↓
                       COMPLETE PICTURE
                                  ↓
                 ┌─────────────────────────┐
                 │                         │
             AD ATTACK PATHS          DATA EXPOSURE
                 │                         │
             Lateral Move             Credentials
             Priv Esc                 Keys
             Trust Paths              Configs
                 │                         │
                 └────────────┬────────────┘
                              ↓
                       FURTHER ACCESS
```

> [!important]  
> **Do not think:**
> 
> `"Which one of these tools should I use?"`
> 
> Think:
> 
> `"What question am I trying to answer?"`
> 
> Then select the tool based on its role.

---

# 📋 30. Enumeration vs Analysis vs Exploitation

|Phase|Example tools|Main question|
|---|---|---|
|🔎 Enumeration|ActiveDirectory, PowerView, SharpView|What exists?|
|📂 File discovery|Snaffler|What sensitive information can I access?|
|🧩 Relationship collection|SharpHound|How are AD objects related?|
|🧠 Relationship analysis|BloodHound|What meaningful paths exist?|
|🎯 Target identification|Combination of collected data|Which objects deserve further investigation?|
|🛠️ Exploitation|Depends on discovered weakness|Can the identified weakness be validated?|
|🔓 Lateral/vertical movement|Depends on discovered access|Can access be extended?|

---

# 🕵️ 31. Stealth and Assessment Considerations

Using native functionality such as the ActiveDirectory PowerShell module may blend into normal administrative activity more than dropping a separate offensive tool onto a host.

```text
Native AD cmdlets
       ↓
Already part of Windows administration
       ↓
Potentially less unusual than
dropping a custom binary
```

> [!warning]  
> "Stealthier" does **not** mean invisible or undetectable.
> 
> The source material's point is that native administrative functionality can potentially blend more naturally into normal activity than introducing an unfamiliar tool.

Likewise, assessment scope matters.

Some findings may be:

```text
Attack-enabling
      OR
Informational
      OR
Compliance-related
      OR
Security-posture related
```

A finding does not need to directly produce privilege escalation to be relevant to the customer.

---

# 🧾 32. Reporting Mental Model

During a real assessment, document:

- Files transferred to hosts
    
- Files transferred from hosts
    
- Where tools were placed
    
- Where generated data was stored
    
- Relevant enumeration output
    
- Sensitive findings
    
- Attack paths
    
- Systems accessed
    

```text
ACTION
  ↓
DOCUMENT
  ↓
CORRELATE
  ↓
VALIDATE
  ↓
REPORT
  ↓
CLEAN UP
```

> [!important]  
> Depending on engagement scope, cleanup is part of responsible assessment operations. Tools and files introduced into the environment should be removed when appropriate and permitted by the engagement.

---

# 🔄 33. From Enumeration to Attack Paths

At the end of credentialed enumeration, we should have transformed:

```text
USERNAME + PASSWORD
```

into a much richer dataset:

```text
USER
 │
 ├── GROUP MEMBERSHIP
 │       │
 │       └── PRIVILEGES
 │
 ├── SPNs
 │       │
 │       └── SERVICE ACCOUNTS
 │
 ├── LOCAL ADMIN RIGHTS
 │       │
 │       └── COMPUTERS
 │
 ├── SHARE ACCESS
 │       │
 │       └── SENSITIVE FILES
 │
 └── TRUST RELATIONSHIPS
         │
         └── OTHER DOMAINS / FORESTS
```

Then:

```text
                 RAW INFORMATION
                        ↓
                   CORRELATION
                        ↓
                RELATIONSHIPS
                        ↓
                 WEAKNESSES
                        ↓
                ATTACK PATHS
                        ↓
              FURTHER ACCESS
```

---

# ⚠️ 34. Important Technical Distinctions

## ActiveDirectory vs PowerView

```text
ActiveDirectory Module
        ↓
Microsoft's AD PowerShell cmdlets
        ↓
Direct AD queries

PowerView
        ↓
Offensive security reconnaissance toolkit
        ↓
Broader manual enumeration
```

## PowerView vs SharpView

```text
PowerView
   ↓
PowerShell
   ↓
AD reconnaissance

SharpView
   ↓
.NET
   ↓
Similar AD reconnaissance
```

## SharpHound vs BloodHound

```text
SharpHound
   ↓
COLLECT

BloodHound
   ↓
ANALYZE / VISUALIZE
```

## Snaffler vs BloodHound

```text
Snaffler
   ↓
"What files can I access?"

BloodHound
   ↓
"What relationships and paths exist?"
```

## Direct vs Recursive Group Enumeration

```text
Get-ADGroupMember
        ↓
Direct members

Get-DomainGroupMember -Recurse
        ↓
Direct + nested membership
        ↓
Effective membership picture
```

## Domain Admin vs Local Admin

```text
DOMAIN ADMIN
     ↓
Domain-level privileged group

LOCAL ADMIN
     ↓
Administrative rights on a specific host
```

---

# 🧠 35. Final CPTS Mental Model

> [!success]  
> **Credentialed enumeration from Windows is about turning one valid account into a complete understanding of the AD environment.**

```text
🔑 VALID CREDENTIALS
        ↓
🔎 ENUMERATE DOMAIN
        ↓
👤 USERS
👥 GROUPS
🖥️ COMPUTERS
🧬 TRUSTS
📜 GPOs / ACLs
🎫 SPNs
        ↓
🛡️ DETERMINE ACCESS
        ↓
LOCAL ADMIN
RDP
WINRM
SHARES
        ↓
📂 HUNT FILES
        ↓
CREDENTIALS
KEYS
CONFIGURATION
DATABASES
        ↓
🕸️ COLLECT RELATIONSHIPS
        ↓
SharpHound
        ↓
🧠 ANALYZE
        ↓
BloodHound
        ↓
🎯 IDENTIFY POTENTIAL PATHS
        ↓
🛠️ VALIDATE / EXPLOIT
        ↓
🔓 LATERAL / VERTICAL MOVEMENT
```

> [!tip]  
> The most important shift in thinking is:
> 
> **Don't memorize individual commands as isolated tricks.**
> 
> Instead, understand what question each command answers:
> 
> ```text
> Get-ADDomain
>      → What domain am I in?
> 
> Get-ADUser
>      → Who are the users / which accounts have interesting attributes?
> 
> Get-ADGroup / Get-ADGroupMember
>      → Who has which group privileges?
> 
> Get-ADTrust
>      → What other domains are connected?
> 
> Get-DomainUser -SPN
>      → Which user accounts have SPNs?
> 
> Test-AdminAccess
>      → Where am I already an administrator?
> 
> Snaffler
>      → What sensitive files can I access?
> 
> SharpHound
>      → What relationship data can I collect?
> 
> BloodHound
>      → What does all that relationship data mean?
> ```

# ✅ 36. Key Takeaways

1. **Credentialed Windows enumeration gives much deeper visibility than simple network scanning.**
    
2. The **ActiveDirectory PowerShell module** provides native AD querying capabilities.
    
3. `Get-ADDomain` reveals important domain structure, including **domain SID, functional level, child domains, forest information, and DC roles**.
    
4. Accounts with a populated `ServicePrincipalName` attribute are potential **Kerberoasting targets**.
    
5. `Get-ADTrust` and PowerView's `Get-DomainTrustMapping` reveal **domain and forest trust relationships**.
    
6. Group enumeration must consider **nested membership**, not only direct membership.
    
7. **Local administrator access** and **Domain Admin membership** are different concepts.
    
8. **PowerView** is a broad, manual AD reconnaissance framework.
    
9. **SharpView** provides similar PowerView-style functionality through .NET.
    
10. **Snaffler** focuses on accessible shares and potentially sensitive files.
    
11. **SharpHound collects relationship data.**
    
12. **BloodHound analyzes and visualizes that relationship data.**
    
13. BloodHound is especially powerful because it changes AD enumeration from isolated facts into a **relationship graph**.
    
14. Not every finding needs to directly lead to exploitation; **data exposure, excessive permissions, stale systems, and other security-posture issues can still matter to the customer**.
    
15. Always distinguish:
    
    - **Enumeration** → collecting information
        
    - **Correlation** → connecting information
        
    - **Analysis** → understanding relationships
        
    - **Exploitation** → validating a weakness
        
16. The overall CPTS workflow is:
    

```text
🔎 ENUMERATE
     ↓
📊 COLLECT
     ↓
🧩 CORRELATE
     ↓
🧠 ANALYZE
     ↓
🎯 IDENTIFY PATH
     ↓
🛠️ VALIDATE
     ↓
🔓 MOVE / ESCALATE
```

> [!success]  
> **Core CPTS takeaway:**
> 
> A valid domain account is not just a credential for logging in somewhere. It is a starting point for discovering the **permissions, relationships, trust boundaries, accessible data, privileged accounts, and potential movement paths** throughout the AD environment.