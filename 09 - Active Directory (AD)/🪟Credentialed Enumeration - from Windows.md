# 🪟 1. Credentialed Enumeration — from Windows

> [!info] Objective  
> After obtaining a foothold in an Active Directory environment, a **low-privileged domain account** can be used to enumerate the domain from a Windows attack host.
> 
> The goal is to build a larger picture of:
> 
> - 👤 Users and user attributes
>     
> - 👥 Groups and nested group membership
>     
> - 🖥️ Computers and local administrator access
>     
> - 📜 GPOs
>     
> - 🔐 ACLs and permissions
>     
> - 🤝 Domain/forest trusts
>     
> - 📂 SMB/file shares and sensitive files
>     
> - 🎫 SPNs and potential Kerberoasting targets
>     
> - 🕸️ Relationships and attack paths through BloodHound
>     
> 
> Not every finding directly enables an attack. Some findings are **informational or security-posture issues** that may still belong in a penetration-test report.

> [!important] Mental Model  
> **A valid domain credential is an enumeration key.**
> 
> ```text
> 🔑 LOW-PRIV DOMAIN CREDENTIAL
>          │
>          ▼
> 🧭 MAP THE DOMAIN
> ├── Users / attributes
> ├── Groups / membership
> ├── Computers
> ├── Trusts
> └── GPOs / ACLs
>          │
>          ▼
> 📂 FIND ACCESSIBLE RESOURCES
> ├── SMB shares
> ├── Sensitive files
> └── Logged-on users
>          │
>          ▼
> 🔐 FIND PRIVILEGE OPPORTUNITIES
> ├── Local admin access
> ├── Privileged groups
> ├── SPNs
> └── Nested memberships
>          │
>          ▼
> 🕸️ CORRELATE RELATIONSHIPS
> └── BloodHound / SharpHound
>          │
>          ▼
> 🎯 IDENTIFY LATERAL / VERTICAL MOVEMENT PATHS
> ```

---

# **they are not all doing the same job**. Think of them as different layers of AD enumeration:

|Tool|Main purpose|Mental model|
|---|---|---|
|**ActiveDirectory PowerShell Module**|Query AD directly|🗄️ **Ask AD for specific data**|
|**PowerView**|Powerful AD enumeration|🔎 **Explore AD from many angles**|
|**SharpView**|PowerView rewritten in C#|🥷 **PowerView functionality without PowerShell**|
|**Snaffler**|Find sensitive files/credentials|📂 **Search the network for juicy files**|
|**SharpHound**|Collect AD relationship data|🕸️ **Build the data for BloodHound**|
|**BloodHound**|Analyze AD relationships|🧠 **Show attack paths / relationships visually**|

### 1. ActiveDirectory PowerShell Module

This is the **native Microsoft AD management module**.

For example:

```
Get-ADDomain
```

→ "Tell me information about the current domain."

```
Get-ADUser -Filter *
```

→ "Give me AD users."

```
Get-ADGroupMember -Identity "Backup Operators"
```

→ "Who belongs to this group?"

So you're basically **directly querying Active Directory**.

---

### 2. PowerView

PowerView gives you **much broader enumeration capabilities**.

For example:

```
Get-DomainUser
Get-DomainGroupMember
Get-DomainTrustMapping
Test-AdminAccess
Get-DomainUser -SPN
```

Instead of simply asking:

> "Give me this AD object."

you're doing things like:

> "Show me domain relationships, privileged groups, trusts, SPNs, where I have admin access, etc."

So:

**AD Module = direct AD queries**

**PowerView = offensive/security-focused AD reconnaissance**

---

### 3. SharpView

SharpView is basically the **C# version of PowerView's functionality**.

Instead of:

```
Get-DomainUser
```

you might use:

```
SharpView.exe Get-DomainUser
```

The important distinction is **execution method**, not what you're trying to discover.

> PowerView → PowerShell  
> SharpView → compiled C#

This can be useful when PowerShell execution is restricted or you want to avoid relying on a PowerShell script.

---

### 4. Snaffler

Snaffler is different.

It isn't primarily asking:

> "What are the AD relationships?"

It's looking for **interesting/sensitive files across accessible network resources**.

Think:

```
AD enumeration
     ↓
Who are the users?
Who are the admins?
What groups exist?
What systems exist?
     ↓
Snaffler
     ↓
What interesting files/credentials/configurations
can I access?
```

So Snaffler is more about **finding sensitive information**, such as credentials or configuration files.

---

### 5. SharpHound

SharpHound is the **collector**.

It goes around the AD environment and collects relationship information such as:

```
User → MemberOf → Group
User → AdminTo → Computer
Computer → HasSession → User
Group → MemberOf → Group
Domain → Trust → Domain
```

Then it packages that information for BloodHound.

So:

**SharpHound = collect**

---

### 6. BloodHound

BloodHound is the **analysis/visualization side**.

You give BloodHound the data collected by SharpHound, and BloodHound helps you understand relationships like:

```
Low-privileged user
       ↓
Member of Group A
       ↓
Group A has privileges on Computer B
       ↓
Computer B
       ↓
Admin access
       ↓
Domain Admin
```

So:

**SharpHound → gathers the evidence**

**BloodHound → makes sense of the evidence**

---

## 🧠 The big picture

Think of your CPTS workflow like this:

```
                 ACTIVE DIRECTORY
                        │
          ┌─────────────┴─────────────┐
          │                           │
   AD PowerShell                  PowerView
          │                           │
   Specific queries             Broad enumeration
          │                           │
          └─────────────┬─────────────┘
                        │
                   AD INFORMATION
                        │
             ┌──────────┴──────────┐
             │                     │
         Snaffler             SharpHound
             │                     │
       Find sensitive         Collect AD
          files/data          relationships
                                   │
                                   ↓
                              BloodHound
                                   │
                                   ↓
                         Analyze attack paths
```

### The key distinction for CPTS

Don't think:

> "Which one should I use?"

Think:

> **"What question am I trying to answer?"**

- **Who/what exists in AD?** → AD Module / PowerView
- **What are the trusts, groups, SPNs, privileges, etc.?** → PowerView
- **Need PowerView functionality without PowerShell?** → SharpView
- **Where are interesting files/credentials?** → Snaffler
- **What relationships exist between users, groups, computers, sessions, ACLs, etc.?** → SharpHound
- **What can I do with those relationships / where are the attack paths?** → BloodHound

And this is why your original CPTS workflow says **learn the built-in AD module first**: it teaches you what the underlying AD objects and relationships actually mean. Then PowerView makes enumeration faster/broader, and BloodHound helps you correlate everything.
# 🧰 2. Cheat Sheet — Commands

## ActiveDirectory PowerShell Module

```powershell
Get-Module
Import-Module ActiveDirectory
Get-ADDomain
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
Get-ADTrust -Filter *
Get-ADGroup -Filter * | select name
Get-ADGroup -Identity "Backup Operators"
Get-ADGroupMember -Identity "Backup Operators"
```

## PowerView

```powershell
Import-Module .\PowerView.ps1

Get-DomainUser -Identity mmorgan -Domain inlanefreight.local | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol

Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-DomainTrustMapping
Test-AdminAccess -ComputerName ACADEMY-EA-MS01
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

## SharpView

```powershell
.\SharpView.exe Get-DomainUser -Help
.\SharpView.exe Get-DomainUser -Identity forend
```

## Snaffler

```powershell
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
.\Snaffler.exe -d INLANEFREIGHT.LOCAL -s -v data
```

## SharpHound

```powershell
.\SharpHound.exe --help
.\SharpHound.exe -c All --zipfilename ILFREIGHT
```

## BloodHound

```text
bloodhound
```

> [!tip] CPTS Workflow  
> Learn the **built-in AD module first**, then PowerView/SharpView, then Snaffler, and finally BloodHound/SharpHound for relationship analysis. The built-in tools teach you what the data actually means; BloodHound then helps correlate that data at scale.

---

# 🧭 3. Initial Windows Enumeration Strategy

The first toolset to understand is the **ActiveDirectory PowerShell module**.

[ActiveDirectory PowerShell Module](https://docs.microsoft.com/en-us/powershell/module/activedirectory/?view=windowsserver2022-ps&utm_source=chatgpt.com)

It contains **147 cmdlets at the time of the source material** and provides native PowerShell functionality for administering and querying AD.

When landing on a Windows host—especially an administrator's workstation or server—check what modules are already available.

---

## 🔎 4. Discover PowerShell Modules

The `Get-Module` cmdlet lists currently available/loaded modules and their exported commands.

[Get-Module Documentation](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-module?view=powershell-7.2&utm_source=chatgpt.com)

### Discover Modules

```powershell
        powershell
PS C:\htb> Get-Module

ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Manifest   3.1.0.0    Microsoft.PowerShell.Utility        {Add-Member, Add-Type, Clear-Variable, Compare-Object...}
Script     2.0.0      PSReadline                          {Get-PSReadLineKeyHandler, Get-PSReadLineOption, Remove-PS...


```

The `ActiveDirectory` module is not yet loaded.

Load it with:

```powershell
Import-Module ActiveDirectory
```

Then verify:

```powershell
Get-Module
```

### Load ActiveDirectory Module

```powershell
        powershell
PS C:\htb> Import-Module ActiveDirectory
PS C:\htb> Get-Module

ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Manifest   1.0.1.0    ActiveDirectory                     {Add-ADCentralAccessPolicyMember, Add-ADComputerServiceAcc...
Manifest   3.1.0.0    Microsoft.PowerShell.Utility        {Add-Member, Add-Type, Clear-Variable, Compare-Object...}
Script     2.0.0      PSReadline                          {Get-PSReadLineKeyHandler, Get-PSReadLineOption, Remove-PS...  


```

> [!important] Why Start Here?  
> Native modules can provide extensive AD visibility **without first dropping a dedicated enumeration tool onto the host**. This can be useful during an assessment where tool transfer is restricted or where the tester wants to understand what is already available.

---

# 🌐 5. Get Domain Information

[Get-ADDomain Documentation](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-addomain?view=windowsserver2022-ps&utm_source=chatgpt.com)

```powershell
PS C:\htb> Get-ADDomain
```

### Important Output

```text
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
LinkedGroupPolicyObjects            : {cn={DDBB8574-E94E-4525-8C9D-ABABE31223D0},cn=policies,cn=system,DC=INLANEFREIGHT,
                                      DC=LOCAL, CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=INLANEFREIGHT,
                                      DC=LOCAL}
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

### 🧠 What Matters

|Property|Why it matters|
|---|---|
|`DNSRoot`|Current domain name|
|`DomainSID`|Security identifier for the domain|
|`DomainMode`|Domain functional level|
|`ChildDomains`|Reveals child domains|
|`Forest`|Identifies the forest|
|`DomainControllersContainer`|Location of DC objects|
|`PDCEmulator`|Identifies the PDC Emulator|
|`RIDMaster`|Identifies the RID Master|
|`ReplicaDirectoryServers`|Shows domain replication servers|
|`LinkedGroupPolicyObjects`|Shows GPO relationships|
|`ForeignSecurityPrincipalsContainer`|Indicates where foreign security principals are represented|

> [!important] Trust / Domain Context  
> `ChildDomains` tells us whether the current domain has child domains. This is different from a **trust relationship**: a child domain is part of the same forest hierarchy, while a trust describes a relationship allowing authentication/authorization between domains or forests.

---

# 🎫 6. Find Users With SPNs

[Get-ADUser Documentation](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-aduser?view=windowsserver2022-ps&utm_source=chatgpt.com)

A `ServicePrincipalName` (SPN) identifies a service instance associated with an account.

Filtering for users with an SPN is useful because these accounts **may be susceptible to Kerberoasting**, which is covered later in the module.

```powershell
PS C:\htb> Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

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

<SNIP>
```

> [!tip] CPTS Connection  
> During AD enumeration, **SPN present → identify service account → consider Kerberoasting**. Don't assume every SPN account is vulnerable; this enumeration step identifies candidates.

---

# 🤝 7. Trust Enumeration

[Get-ADTrust Documentation](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-adtrust?view=windowsserver2022-ps&utm_source=chatgpt.com)

```powershell
PS C:\htb> Get-ADTrust -Filter *
```

### Output

```text
Direction               : BiDirectional
DisallowTransivity      : False
DistinguishedName       : CN=LOGISTICS.INLANEFREIGHT.LOCAL,CN=System,DC=INLANEFREIGHT,DC=LOCAL
ForestTransitive         : False
IntraForest             : True
IsTreeParent             : False
IsTreeRoot               : False
Name                     : LOGISTICS.INLANEFREIGHT.LOCAL
ObjectClass              : trustedDomain
ObjectGUID               : f48a1169-2e58-42c1-ba32-a6ccb10057ec
SelectiveAuthentication  : False
SIDFilteringForestAware  : False
SIDFilteringQuarantined  : False
Source                   : DC=INLANEFREIGHT,DC=LOCAL
Target                   : LOGISTICS.INLANEFREIGHT.LOCAL
TGTDelegation            : False
TrustAttributes          : 32
TrustedPolicy            :
TrustingPolicy           :
TrustType                : Uplevel
UplevelOnly              : False
UsesAESKeys              : False
UsesRC4Encryption        : False

Direction               : BiDirectional
DisallowTransivity       : False
DistinguishedName        : CN=FREIGHTLOGISTICS.LOCAL,CN=System,DC=INLANEFREIGHT,DC=LOCAL
ForestTransitive         : True
IntraForest             : False
IsTreeParent             : False
IsTreeRoot               : False
Name                     : FREIGHTLOGISTICS.LOCAL
ObjectClass              : trustedDomain
ObjectGUID               : 1597717f-89b7-49b8-9cd9-0801d52475ca
SelectiveAuthentication  : False
SIDFilteringForestAware  : False
SIDFilteringQuarantined  : False
Source                   : DC=INLANEFREIGHT,DC=LOCAL
Target                   : FREIGHTLOGISTICS.LOCAL
TGTDelegation            : False
TrustAttributes          : 8
TrustedPolicy            :
TrustingPolicy            :
TrustType                : Uplevel
UplevelOnly              : False
UsesAESKeys              : False
UsesRC4Encryption        : False
```

### 🧠 Read the Important Fields

|Field|Meaning|
|---|---|
|`Direction`|Direction of the trust|
|`IntraForest`|Indicates a trust within the same forest|
|`ForestTransitive`|Indicates forest-transitive behavior|
|`Source`|Domain initiating/owning the trust object|
|`Target`|Domain on the other side|
|`TrustType`|Type of trust|
|`SelectiveAuthentication`|Whether selective authentication is configured|
|`SIDFilteringForestAware`|SID-filtering configuration|
|`TGTDelegation`|TGT delegation setting|

> [!important] CPTS Focus  
> Trust enumeration expands the attack surface beyond the current domain.
> 
> ```text
> CURRENT DOMAIN
>      │
>      ├── Child / same-forest domains
>      │
>      └── External / other-forest trusts
>              │
>              ▼
>       Additional AD environments
> ```
> 
> Later in the module, trust relationships become relevant when examining **child-to-parent** and **cross-forest** attack paths.

---

# 👥 8. Group Enumeration

[Get-ADGroup Documentation](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-adgroup?view=windowsserver2022-ps&utm_source=chatgpt.com)

Enumerate all groups:

```powershell
PS C:\htb> Get-ADGroup -Filter * | select name
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

### 🎯 Groups Worth Investigating

- `Domain Admins`
    
- `Enterprise Admins`
    
- `Schema Admins`
    
- `Administrators`
    
- `Backup Operators`
    
- `Remote Management Users`
    
- `Remote Desktop Users`
    
- `Hyper-V Administrators`
    
- `Certificate Service DCOM Access`
    
- Other custom administrative/security groups
    

> [!warning] Don't Only Look at Domain Admins  
> Built-in or custom groups can provide important privileges without being named `Domain Admins`. Always examine **membership and effective privileges**, not merely the group name.

---

# 🔍 9. Detailed Group Information

```powershell
PS C:\htb> Get-ADGroup -Identity "Backup Operators"

DistinguishedName : CN=Backup Operators,CN=Builtin,DC=INLANEFREIGHT,DC=LOCAL
GroupCategory     : Security
GroupScope        : DomainLocal
Name              : Backup Operators
ObjectClass       : group
ObjectGUID        : 6276d85d-9c39-4b7c-8449-cad37e8abc38
SamAccountName    : Backup Operators
SID               : S-1-5-32-551
```

This gives additional context such as:

- Group scope
    
- Group category
    
- SID
    
- Distinguished Name
    
- Object GUID
    

---

# 👤 10. Group Membership

[Get-ADGroupMember Documentation](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-adgroupmember?view=windowsserver2022-ps&utm_source=chatgpt.com)

```powershell
PS C:\htb> Get-ADGroupMember -Identity "Backup Operators"
```

Output:

```text
distinguishedName : CN=BACKUPAGENT,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
name              : BACKUPAGENT
objectClass       : user
objectGUID        : 2ec53e98-3a64-4706-be23-1d824ff61bed
SamAccountName    : backupagent
SID               : S-1-5-21-3842939050-3880317879-2865463114-5220
```

`backupagent` is a member of `Backup Operators`.

> [!important] Why This Matters  
> A service account may look uninteresting until its **group membership** is examined.
> 
> ```text
> backupagent
>      │
>      ▼
> Backup Operators
>      │
>      ▼
> Potentially significant privileges
> ```
> 
> The source specifically notes that compromising this service account could provide a path toward domain takeover. The exact path would still need to be validated during the assessment.

---

# 🧠 11. Why Manual AD Enumeration Gets Tedious

The ActiveDirectory module is important because it teaches the underlying AD objects and relationships.

However:

```text
Users
  ↓
Groups
  ↓
Nested Groups
  ↓
Computers
  ↓
Sessions
  ↓
ACLs
  ↓
GPOs
  ↓
Trusts
  ↓
Local Admin Rights
```

quickly becomes **too much data to manually correlate**.

This is where tools such as **PowerView** and especially **BloodHound** become valuable.

---

# 🕵️ 12. PowerView

[PowerView / PowerSploit Recon](https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon?utm_source=chatgpt.com)

[PowerView in HTB Academy](https://academy.hackthebox.com/course/preview/active-directory-powerview?utm_source=chatgpt.com)

PowerView is a PowerShell tool for gaining situational awareness within an AD environment.

It can enumerate:

- 👤 Users
    
- 🖥️ Computers
    
- 👥 Groups
    
- 🔐 ACLs
    
- 🤝 Trusts
    
- 📂 Shares
    
- 🔑 Password-related information
    
- 🎫 SPNs / Kerberoasting candidates
    
- 👤 User locations/sessions
    
- 🛡️ Local administrator access
    

Compared with BloodHound:

|PowerView|BloodHound|
|---|---|
|Manual investigation|Relationship-oriented graph|
|Powerful individual queries|Correlates many relationships|
|Great for targeted enumeration|Great for attack-path discovery|
|Can identify subtle misconfigurations|Makes complex relationships easier to visualize|
|PowerShell-based|Collector + graph database/GUI|

---

## 📊 13. PowerView Function Cheat Sheet

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

> [!tip] PowerView Mental Model  
> **PowerView = manually query the exact AD relationship you are interested in.**
> 
> BloodHound is excellent when you don't yet know which relationship matters. PowerView is excellent when you want to investigate a specific object or relationship in detail.

---

# 👤 14. Detailed Domain User Enumeration

[PowerView Get-DomainUser Documentation](https://powersploit.readthedocs.io/en/latest/Recon/Get-DomainUser/?utm_source=chatgpt.com)

```powershell
PS C:\Windows\system32> cd C:\Tools\
PS C:\Tools> Import-Module .\PowerView.ps1
PS C:\htb> Get-DomainUser -Identity mmorgan -Domain inlanefreight.local | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol
```

Output:

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

### 🔎 Important Fields

|Field|Why inspect it|
|---|---|
|`samaccountname`|Windows account name|
|`memberof`|Group memberships|
|`whencreated`|Account creation time|
|`pwdlastset`|Password modification timestamp|
|`lastlogontimestamp`|Approximate recent logon information|
|`accountexpires`|Account expiration|
|`admincount`|Indicates association with protected/privileged group administration|
|`userprincipalname`|User's UPN|
|`serviceprincipalname`|SPNs associated with the account|
|`useraccountcontrol`|Important account-control flags|

The example shows:

```text
admincount           : 1
useraccountcontrol   : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
```

These attributes are worth investigating further.

> [!warning] Attribute ≠ Immediate Exploit  
> Enumeration tells you **what exists**. An interesting attribute does not automatically mean exploitation is possible. Validate the surrounding configuration and permissions.

---

# 👥 15. Recursive Group Membership

[PowerView Get-DomainGroupMember Documentation](https://powersploit.readthedocs.io/en/latest/Recon/Get-DomainGroupMember/?utm_source=chatgpt.com)

```powershell
PS C:\htb> Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

The `-Recurse` switch is extremely important because AD groups can contain **other groups**.

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
MemberDomain             : INLANEFREIGHT.LOCAL
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
Secadmins
   ↓
Domain Admins
```

The important lesson is that **direct membership is not the whole story**.

> [!important] CPTS  
> Whenever you enumerate privileged groups, check **nested membership**. A user may inherit powerful privileges without appearing as a direct member of the privileged group.

---

# 🤝 16. PowerView Trust Mapping

```powershell
PS C:\htb> Get-DomainTrustMapping
```

Output:

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

This provides a trust map showing:

```text
INLANEFREIGHT.LOCAL
       │
       ├── WITHIN_FOREST
       │      ↓
       │ LOGISTICS.INLANEFREIGHT.LOCAL
       │
       └── FOREST_TRANSITIVE
              ↓
       FREIGHTLOGISTICS.LOCAL
```

---

# 🛡️ 17. Test Local Administrator Access

[PowerView Test-AdminAccess Documentation](https://powersploit.readthedocs.io/en/latest/Recon/Test-AdminAccess/?utm_source=chatgpt.com)

```powershell
PS C:\htb> Test-AdminAccess -ComputerName ACADEMY-EA-MS01

ComputerName    IsAdmin
------------    -------
ACADEMY-EA-MS01    True
```

This answers a very important question:

> **Can my current domain user administer this remote machine?**

In a real environment, this can be repeated against multiple hosts to determine where the current account has administrative rights.

### Mental Model

```text
Current User
     │
     ├── Host A → IsAdmin = False
     ├── Host B → IsAdmin = True  ← administrative foothold
     ├── Host C → IsAdmin = False
     └── Host D → IsAdmin = True  ← administrative foothold
```

> [!important] Why This Matters  
> Local administrator rights can provide a bridge to another machine, where additional credentials, sessions, files, or other useful information may exist.

---

# 🎫 18. Find Users With SPNs — PowerView

```powershell
PS C:\htb> Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName

serviceprincipalname                          samaccountname
--------------------                          --------------
adfsconnect/azure01.inlanefreight.local       adfs
backupjob/veam001.inlanefreight.local          backupagent
d0wngrade/kerberoast.inlanefreight.local      d0wngrade
kadmin/changepw                               krbtgt
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433 sqldev
MSSQLSvc/SPSJDB.inlanefreight.local:1433      sqlprod
MSSQLSvc/SQL-CL01-01inlanefreight.local:49351 sqlqa
sts/inlanefreight.local                       solarwindsmonitor
testspn/kerberoast.inlanefreight.local        testspn
testspn2/kerberoast.inlanefreight.local       testspn2
```

> [!tip] Enumeration Pattern
> 
> ```text
> Get users
>    ↓
> Filter SPN
>    ↓
> Identify service accounts
>    ↓
> Assess Kerberoasting exposure
> ```
> 
> This is a **candidate-identification step**, not proof that every account can be successfully Kerberoasted.

---

# ⚡ 19. SharpView

PowerView is associated with the PowerSploit ecosystem, while **SharpView** is a .NET port providing many of the same enumeration functions.

The source notes that PowerView has also been maintained in the BC-Security Empire ecosystem and that the newer version includes functions such as `Get-NetGmsa`.

[Empire / BC-Security PowerView](https://github.com/BC-SECURITY/Empire/blob/master/empire/server/data/module_source/situational_awareness/network/powerview.ps1?utm_source=chatgpt.com)

SharpView is useful when:

- PowerShell usage is restricted/hardened
    
- A .NET executable is more practical
    
- You want many PowerView-style queries without importing the PowerShell script
    

### SharpView Help

```powershell
PS C:\htb> .\SharpView.exe Get-DomainUser -Help

Get_DomainUser -Identity <String[]> -DistinguishedName <String[]> -SamAccountName <String[]> -Name <String[]> -MemberDistinguishedName <String[]> -MemberName <String[]> -SPN <Boolean> -AdminCount <Boolean> -AllowDelegation <Boolean> -DisallowDelegation <Boolean> -TrustedToAuth <Boolean> -PreauthNotRequired <Boolean> -KerberosPreauthNotRequired <Boolean> -NoPreauth <Boolean> -Domain <String> -LDAPFilter <String> -Filter <String> -Properties <String[]> -SearchBase <String> -ADSPath <String> -Server <String> -DomainController <String> -SearchScope <SearchScope> -ResultPageSize <Int32> -ServerTimeLimit <Nullable`1> -SecurityMasks <Nullable`1> -Tombstone <Boolean> -FindOne <Boolean> -ReturnOne <Boolean> -Credential <NetworkCredential> -Raw <Boolean> -UACFilter <UACEnum>
```

### Query a Specific User

```powershell
PS C:\htb> .\SharpView.exe Get-DomainUser -Identity forend
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
dscorepropagationdata          : {3/24/2022 3:58:07 PM, 3/24/2022 3:57:44 PM, 3/24/2022 3:52:58 PM, 3/24/2022 3:49:31 PM, 7/14/1601 10:36:49 PM}
usncreated                     : 3054181
instancetype                   : 4
codepage                       : 0
```

> [!important] SharpView vs PowerView
> 
> ```text
> PowerView
>    └── PowerShell-based AD enumeration
> 
> SharpView
>    └── .NET implementation of many PowerView-style functions
> ```
> 
> The source specifically notes SharpView can be useful when PowerShell is hardened or unavailable.

---

# 📂 20. SMB Shares — Why Pillage Them?

Domain shares are designed to let users access information required for their roles.

The security model should generally be:

```text
User
  ↓
Authentication
  ↓
Authorization
  ↓
Only required shares/data
```

Overly permissive shares can expose:

- 🔑 Passwords
    
- 🔐 SSH keys
    
- ⚙️ Configuration files
    
- 🧾 Personnel/HR information
    
- ⚖️ Legal information
    
- 🏥 Medical information
    
- 🗄️ Database files
    
- 🛠️ Infrastructure files
    

This can be especially valuable after compromising a standard domain account that has access to IT/infrastructure shares.

> [!warning] Reporting Perspective  
> Share exposure is not merely an attack-path problem. Sensitive information accessible to users who do not need it can be a standalone security finding and may have regulatory implications such as HIPAA or PCI requirements.

PowerView can hunt for shares, but manually searching large environments can become tedious. **Snaffler** automates much of this process.

---

# 🦅 21. Snaffler

[Snaffler](https://github.com/SnaffCon/Snaffler?utm_source=chatgpt.com)

Snaffler searches an AD environment for potentially sensitive information.

High-level workflow:

```text
🏢 Enumerate domain hosts
        ↓
📂 Find shares
        ↓
🔐 Determine readable directories
        ↓
🔎 Search accessible files
        ↓
🚨 Identify potentially sensitive files
```

Snaffler requires a **domain-joined host or domain-user context**.

### Snaffler Execution

```powershell
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
```

Important options:

|Option|Purpose|
|---|---|
|`-s`|Print results to console|
|`-d`|Domain to search|
|`-o`|Output logfile|
|`-v`|Verbosity level|

The source notes that `data` is typically useful for the verbosity setting because it makes the initial output easier to review.

Because Snaffler can generate a large amount of information, logging results to a file is useful.

### Snaffler in Action

```powershell
PS C:\htb> .\Snaffler.exe  -d INLANEFREIGHT.LOCAL -s -v data

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
2022-03-31 12:17:19 -07:00 [Share] {Red}(\\ACADEMY-EA-FILE.INLANEFREIGHT.LOCAL\WsusContent)
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.keychain$|295B|3/31/2022 12:08:42 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\SetStep.keychain) .keychain
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.tblk$|279B|3/31/2022 12:05:25 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\FindConnect.tblk) .tblk
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.psafe3$|301B|3/31/2022 12:09:33 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\GetUpdate.psafe3) .psafe3
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.keypair$|278B|3/31/2022 12:09:09 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Infosec\UnprotectConvertTo.keypair) .keypair
2022-03-31 12:17:19 -07:00 [File] {Black}<KeepExtExactBlack|R|^\.tblk$|280B|3/31/2022 12:05:17 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\ExportJoin.tblk) .tblk
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.mdf$|305B|3/31/2022 12:09:27 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\FormatShow.mdf) .mdf
2022-03-31 12:17:19 -07:00 [File] {Red}<KeepExtExactRed|R|^\.mdf$|299B|3/31/2022 12:09:14 PM>(\\ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL\Department Shares\IT\Development\LockConfirm.mdf) .mdf

<SNIP>
```

### 🧠 Interpreting Snaffler

The output can reveal files such as:

- `.key`
    
- `.keychain`
    
- `.ppk`
    
- `.kdb`
    
- `.kwallet`
    
- `.psafe3`
    
- `.keypair`
    
- `.sqldump`
    
- `.mdf`
    

These extensions can indicate potentially sensitive credentials, keys, password databases, database data, or configuration material.

> [!important] Don't Treat Every Hit as a Credential  
> Snaffler identifies **potentially interesting files**. The extension alone does not prove the file contains usable credentials. Each hit requires validation and contextual analysis.

---

# 🕸️ 22. BloodHound / SharpHound

[BloodHound](https://github.com/BloodHoundAD/BloodHound?utm_source=chatgpt.com)

[SharpHound](https://github.com/BloodHoundAD/SharpHound?utm_source=chatgpt.com)

BloodHound is designed to visualize relationships between AD objects and identify potential attack paths.

Instead of manually correlating:

```text
User
 ↓
Group
 ↓
Computer
 ↓
Local Admin
 ↓
Session
 ↓
ACL
 ↓
GPO
 ↓
Privileged Account
```

BloodHound turns these relationships into a graph.

> [!tip] Mental Model  
> **PowerView asks:**  
> _“Tell me about this object.”_
> 
> **BloodHound asks:**  
> _“Show me how these objects are connected.”_

BloodHound can help analyze:

- Users
    
- Groups
    
- Computers
    
- Group memberships
    
- GPOs
    
- ACLs
    
- Trusts
    
- Local administrator relationships
    
- Sessions
    
- RDP access
    
- WinRM / PSRemote access
    
- DCOM access
    
- SPN-related relationships
    
- Computer/user properties
    

---

# 🦅 23. SharpHound Collection

SharpHound is the Windows collector used to gather AD data for BloodHound.

The Windows attack host may be:

- A domain-joined host
    
- A Windows host positioned inside the network
    
- Another Windows attack host from which the tool can authenticate/access the domain
    

[HTB File Transfer Module](https://academy.hackthebox.com/course/preview/file-transfers?utm_source=chatgpt.com)

The source notes methods such as:

- Python HTTP server
    
- `smbserver.py` from Impacket
    
- Other authorized file-transfer methods
    

### SharpHound Help

```powershell
PS C:\htb> .\SharpHound.exe --help

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

### Run Collection

```powershell
PS C:\htb> .\SharpHound.exe -c All --zipfilename ILFREIGHT
```

Output:

```text
2022-04-18T13:58:22.1163680-07:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2022-04-18T13:58:22.1163680-07:00|INFORMATION|Initializing SharpHound at 1:58 PM on 4/18/2022
2022-04-18T13:58:22.1163680-07:00|INFORMATION|Flags: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2022-04-18T13:58:23.0851206-07:00|INFORMATION|Beginning LDAP search for INLANEFREIGHT.LOCAL
2022-04-18T13:58:53.9132950-07:00|INFORMATION|Status: 0 objects finished (+0 0)/s -- Using 67 MB RAM
2022-04-18T13:59:15.7882419-07:00|INFORMATION|Producer has finished, closing LDAP channel
2022-04-18T13:59:16.1788930-07:00|INFORMATION|LDAP channel closed, waiting for consumers
2022-04-18T13:59:23.9288698-07:00|INFORMATION|Status: 3793 objects finished (+3793 63.21667)/s -- Using 112 MB RAM
2022-04-18T13:59:45.4132561-07:00|INFORMATION|Consumers finished, closing output channel
Closing writers
2022-04-18T13:59:45.4601086-07:00|INFORMATION|Output channel closed, waiting for output task to complete
2022-04-18T13:59:45.8663660-07:00|INFORMATION|Status: 3809 objects finished (+16 46.45122)/s -- Using 110 MB RAM
2022-04-18T13:59:45.8663660-07:00|INFORMATION|Enumeration finished in 00:01:22.7919186
2022-04-18T13:59:46.3663660-07:00|INFORMATION|SharpHound Enumeration Completed at 1:59 PM on 4/18/2022! Happy Graphing
```

> [!important] Collection vs Analysis  
> **SharpHound collects the data. BloodHound analyzes and visualizes it.**
> 
> ```text
> Windows Host
>      │
>      ▼
> SharpHound
>      │
>      ├── Users
>      ├── Groups
>      ├── Computers
>      ├── ACLs
>      ├── Sessions
>      ├── Trusts
>      └── Other relationships
>              │
>              ▼
>        JSON dataset
>              │
>              ▼
>         BloodHound
>              │
>              ▼
>       Graph / attack paths
> ```

---

# 🖥️ 24. BloodHound GUI

After collection, the resulting ZIP/JSON dataset can be loaded into BloodHound.

On the lab MS01 host:

```text
bloodhound
```

If prompted in the source lab:

```text
neo4j: HTB_@cademy_stdnt!
```

Then:

1. Select **Upload Data**
    
2. Select the generated ZIP
    
3. Wait for all `.json` files to reach `100%`
    
4. Browse the domain
    
5. Use the **Analysis** tab
    
6. Use **Raw Query** for custom Cypher
    

The source recommends searching:

```text
domain:
```

and selecting:

```text
INLANEFREIGHT.LOCAL
```

The example environment contains more than **550 hosts** and trusts with other domains.

---

# 🔎 25. BloodHound Analysis Queries

## 🖥️ Find Computers with Unsupported Operating Systems

The query:

```text
Find Computers with Unsupported Operating Systems
```

can identify legacy operating systems.

The source example identifies:

- Windows 7
    
- Windows Server 2008
    

These systems may be vulnerable to older issues such as:

[MS08-067 — Microsoft](https://support.microsoft.com/en-us/topic/ms08-067-vulnerability-in-server-service-could-allow-remote-code-execution-ac7878fc-be69-7143-472d-2507a179cd15?utm_source=chatgpt.com)

> [!warning] Validate Legacy Hosts  
> A stale AD record does **not necessarily mean the host is currently alive**.
> 
> Before recommending remediation or attempting an attack against a legacy host, validate whether it is actually powered on and whether it supports a critical application/service.
> 
> The source notes that an old host could warrant:
> 
> - A finding for legacy operating systems
>     
> - A recommendation to clean stale AD records
>     
> - Network segmentation
>     
> - A plan to decommission/replace the host
>     

---

# 🛡️ 26. Find Computers Where Domain Users Are Local Admin

Query:

```text
Find Computers where Domain Users are Local Admin
```

This can reveal hosts where a broad population of domain users has local administrator rights.

### Mental Model

```text
Domain Users
      │
      ▼
Local Administrators
      │
      ▼
Multiple hosts
      │
      ▼
Any compromised account may have local admin access
```

The source notes that this can enable access to:

- Host resources
    
- Credentials in memory
    
- Sensitive files
    
- Additional sessions or credentials
    

> [!important] Why This Is Valuable  
> A broad local-admin assignment converts a **single compromised domain account** into potential administrative access across additional hosts.

---

# 📊 27. BloodHound vs PowerView vs Native AD Tools

|Tool|Main Strength|Best Mental Model|
|---|---|---|
|**ActiveDirectory PowerShell**|Native AD enumeration|🧱 Understand the raw AD objects|
|**PowerView**|Flexible manual AD recon|🔎 Investigate specific relationships|
|**SharpView**|.NET PowerView-style enumeration|⚡ PowerView-style recon without relying on PowerShell|
|**Snaffler**|Share/file hunting|📂 Find sensitive data|
|**SharpHound**|Windows BloodHound collection|🦅 Collect relationship data|
|**BloodHound**|Graph analysis|🕸️ Correlate relationships and attack paths|

> [!success] CPTS Progression
> 
> ```text
> Native AD Module
>       ↓
> Understand users/groups/trusts/attributes
>       ↓
> PowerView / SharpView
>       ↓
> Investigate relationships manually
>       ↓
> Snaffler
>       ↓
> Hunt accessible shares/files
>       ↓
> SharpHound
>       ↓
> Collect broad AD relationship data
>       ↓
> BloodHound
>       ↓
> Visualize + correlate + identify paths
> ```

---

# 🧹 28. Operational Documentation

During an assessment, document:

- Every file transferred to a host
    
- Every file transferred from a host
    
- Where files were placed
    
- Tools introduced into the environment
    
- Relevant output
    
- Findings and relationships discovered
    

This matters for:

- Customer deconfliction
    
- Reporting
    
- Reproducing findings
    
- Cleanup
    
- Engagement closeout
    

> [!important] Cleanup  
> Depending on the engagement scope, remove tools, scripts, temporary files, and other artifacts introduced into the environment at the conclusion of the assessment.

---

# 🧠 29. The Big Picture — Credentialed Enumeration from Windows

Once we have a valid domain account:

```text
🔑 VALID DOMAIN CREDENTIAL
             │
             ▼
     🌐 GET-ADDOMAIN
             │
             ├── Domain SID
             ├── Forest
             ├── Child domains
             └── DC information
             │
             ▼
       👤 USERS / SPNs
             │
             ├── User attributes
             ├── Privileged attributes
             └── Kerberoasting candidates
             │
             ▼
        👥 GROUPS
             │
             ├── Privileged groups
             ├── Membership
             └── Nested membership
             │
             ▼
         🤝 TRUSTS
             │
             ├── Within-forest
             └── Cross-forest
             │
             ▼
        🛡️ LOCAL ADMIN
             │
             └── Find hosts where current user is admin
             │
             ▼
          📂 SHARES
             │
             ├── Readable shares
             └── Sensitive files
             │
             ▼
        🦅 BLOODHOUND
             │
             ├── ACLs
             ├── Sessions
             ├── GPOs
             ├── Trusts
             ├── Local admin
             └── Other relationships
             │
             ▼
       🕸️ GRAPH ANALYSIS
             │
             ▼
    🎯 ATTACK-PATH / MISCONFIGURATION DISCOVERY
```

---

# 🧩 30. What Each Tool Answers

### 🧱 ActiveDirectory PowerShell Module

> **“What objects exist in AD, and what attributes do they have?”**

Use for:

- Domain information
    
- Users
    
- Groups
    
- Group members
    
- Trusts
    
- SPNs
    

---

### 🔎 PowerView

> **“Tell me exactly what I want to know about this AD relationship.”**

Use for:

- Detailed users
    
- Recursive group membership
    
- Trust mapping
    
- Local admin checks
    
- SPNs
    
- Sessions
    
- Shares
    
- ACLs
    
- GPOs
    

---

### ⚡ SharpView

> **“Can I perform PowerView-style enumeration through .NET?”**

Useful when PowerShell is restricted/hardened.

---

### 📂 Snaffler

> **“What sensitive files can my domain context reach?”**

Focus:

```text
Hosts → Shares → Readable directories → Interesting files
```

---

### 🦅 SharpHound

> **“Collect the AD relationship data.”**

---

### 🕸️ BloodHound

> **“How are all these AD objects connected?”**

This is the major advantage:

```text
Individual facts
      ↓
Relationships
      ↓
Graph
      ↓
Potential paths
```

---

# 🧠 31. CPTS Mental Model — Don't Enumerate Randomly

A common beginner mistake is:

```text
Run tool
↓
Get huge output
↓
Scroll
↓
Get overwhelmed
```

Instead:

```text
1️⃣ DOMAIN
   ↓
2️⃣ USERS / GROUPS
   ↓
3️⃣ PRIVILEGED MEMBERSHIP
   ↓
4️⃣ COMPUTERS
   ↓
5️⃣ LOCAL ADMIN ACCESS
   ↓
6️⃣ SESSIONS
   ↓
7️⃣ SHARES / FILES
   ↓
8️⃣ TRUSTS
   ↓
9️⃣ SPNs
   ↓
🔟 BLOODHOUND CORRELATION
```

Then ask:

> **“What does this piece of information allow me to investigate next?”**

Examples:

```text
Found privileged group
        ↓
Enumerate its members
        ↓
Check nested groups
        ↓
Find where those users are logged in
        ↓
Check administrative access
        ↓
Correlate with BloodHound
```

Or:

```text
Found readable IT share
        ↓
Search files
        ↓
Find configuration / credential material
        ↓
Identify account
        ↓
Retest credentials against relevant services
```

Or:

```text
Found trust
        ↓
Identify trust direction/type
        ↓
Determine which domains are reachable
        ↓
Map relationships in BloodHound
        ↓
Investigate cross-domain paths
```

---

# 🚨 32. Important Distinctions for the Exam

|Concept|Remember|
|---|---|
|**Domain**|AD administrative/security boundary containing users, computers, groups, etc.|
|**Forest**|Collection of one or more AD domains sharing the forest structure|
|**Child Domain**|A domain beneath another domain in the same forest hierarchy|
|**Trust**|Relationship allowing authentication/authorization between domains/forests|
|**Within-Forest Trust**|Trust relationship between domains in the same forest|
|**Forest-Transitive Trust**|Trust extending across forests|
|**Direct Group Membership**|User directly belongs to the group|
|**Nested Membership**|User belongs to another group that is itself a member of the target group|
|**SPN**|Service Principal Name associated with a service/account; useful for identifying Kerberoasting candidates|
|**Local Admin**|Administrative rights over a specific host, not automatically Domain Admin|
|**SharpHound**|Collector|
|**BloodHound**|Graph analysis/visualization|
|**PowerView**|Manual AD reconnaissance|
|**Snaffler**|Automated share/file hunting|

---

# 🔗 33. Resources

- [ActiveDirectory PowerShell Module](https://docs.microsoft.com/en-us/powershell/module/activedirectory/?view=windowsserver2022-ps&utm_source=chatgpt.com)
    
- [PowerView / PowerSploit](https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon?utm_source=chatgpt.com)
    
- [PowerView — HTB Academy](https://academy.hackthebox.com/course/preview/active-directory-powerview?utm_source=chatgpt.com)
    
- [Snaffler](https://github.com/SnaffCon/Snaffler?utm_source=chatgpt.com)
    
- [BloodHound](https://github.com/BloodHoundAD/BloodHound?utm_source=chatgpt.com)
    
- [SharpHound](https://github.com/BloodHoundAD/SharpHound?utm_source=chatgpt.com)
    
- [BloodHound Cypher Cheat Sheet](https://hausec.com/2019/09/09/bloodhound-cypher-cheatsheet/?utm_source=chatgpt.com)
    
- [Active Directory BloodHound — HTB Academy](https://academy.hackthebox.com/course/preview/active-directory-bloodhound?utm_source=chatgpt.com)
    
- [File Transfers — HTB Academy](https://academy.hackthebox.com/course/preview/file-transfers?utm_source=chatgpt.com)
    

---

# 🏁 34. Key Takeaways

> [!success] Remember These
> 
> - 🔑 **Valid domain credentials are extremely valuable for enumeration**, even when they are low privilege.
>     
> - 🧱 **ActiveDirectory PowerShell** teaches you the underlying AD objects and attributes.
>     
> - 👥 Always investigate **group membership and nested groups**.
>     
> - 🎫 Users with **SPNs** are candidates for further Kerberoasting investigation.
>     
> - 🤝 Always enumerate **trust relationships**, including within-forest and cross-forest relationships.
>     
> - 🛡️ Determine where your current user has **local administrator access**.
>     
> - 📂 Enumerate readable **SMB shares** and investigate potentially sensitive files.
>     
> - 🦅 **Snaffler** automates much of the share/file hunting process.
>     
> - 🔎 **PowerView** is excellent for targeted, manual AD enumeration.
>     
> - ⚡ **SharpView** provides many PowerView-style functions through .NET.
>     
> - 🦅 **SharpHound collects** AD relationship data.
>     
> - 🕸️ **BloodHound analyzes and visualizes** those relationships.
>     
> - 🧠 The goal is not to run every command blindly; each enumeration result should answer **what to investigate next**.
>     
> - 📝 Document tools, files, output, and changes throughout the engagement.
>     
> - 🧹 Clean up artifacts at the end of the engagement according to the agreed scope.
>     
> 
> ```text
> 🔑 CREDS
>    ↓
> 🧭 ENUMERATE
>    ↓
> 👥 UNDERSTAND MEMBERSHIP
>    ↓
> 🖥️ MAP ACCESS
>    ↓
> 📂 PILLAGE REACHABLE DATA
>    ↓
> 🤝 MAP TRUSTS
>    ↓
> 🎫 IDENTIFY SPNs
>    ↓
> 🦅 COLLECT
>    ↓
> 🕸️ CORRELATE WITH BLOODHOUND
>    ↓
> 🎯 UNDERSTAND POSSIBLE PATHS
> ```