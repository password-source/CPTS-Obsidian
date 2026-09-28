# 🔐 1. ACL Enumeration

> [!info]  
> **Objective:** Learn how to enumerate Active Directory ACLs from a controlled user, identify exactly which objects that user can control, translate unreadable ACE GUIDs into meaningful rights, and follow those permissions through **nested groups and multiple objects** until they reveal a viable attack path.
> 
> The key methodology is:
> 
> **Current foothold → Enumerate ACLs → Identify controlled object → Understand the ACE → Follow new privileges → Re-enumerate → Build the attack path**

---

## 💻 2. Command Cheat Sheet

|Command|What it does|
|---|---|
|`Import-Module .\PowerView.ps1`|Loads [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) into the current PowerShell session.|
|`$sid = Convert-NameToSid wley`|Converts the `wley` account name into its SID for targeted ACL searches.|
|`Find-InterestingDomainAcl`|Searches the domain for potentially interesting ACL entries, but can return a very large amount of data.|
|`Get-DomainObjectACL -Identity *`|Enumerates ACLs for domain objects.|
|`Get-DomainObjectACL -Identity * \| ? {$_.SecurityIdentifier -eq $sid}`|Filters ACL results to objects controlled by the SID stored in `$sid`.|
|`Get-DomainObjectACL -ResolveGUIDs -Identity * \| ? {$_.SecurityIdentifier -eq $sid}`|Enumerates ACLs for the SID while resolving object-right GUIDs into readable names.|
|`$guid= "00299570-246d-11d0-a768-00aa006e0529"`|Stores an ACE/control-right GUID for lookup.|
|`Get-ADRootDSE`|Retrieves information about the current AD forest/domain root, including naming contexts.|
|`Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties *`|Enumerates AD control-access-right objects from the Configuration partition.|
|`Select Name,DisplayName,DistinguishedName,rightsGuid`|Selects the useful fields needed to map a control-right GUID to its name.|
|`?{$_.rightsGuid -eq $guid}`|Filters control-right objects to the specified GUID.|
|`fl`|Displays PowerShell output in list format.|
|`Get-ADUser -Filter * \| Select-Object -ExpandProperty SamAccountName > ad_users.txt`|Creates a file containing all domain user SAM account names.|
|`foreach($line in [System.IO.File]::ReadLines("C:\Users\htb-student\Desktop\ad_users.txt")) {...}`|Iterates through every username in the generated file.|
|`Get-ADUser $line`|Retrieves the AD user represented by the current username.|
|`Get-Acl "AD:\$(Get-ADUser $line)"`|Retrieves the ACL/security information for the current AD user object.|
|`Select-Object Path -ExpandProperty Access`|Extracts the access-control entries from the ACL output.|
|`Where-Object {$_.IdentityReference -match 'INLANEFREIGHT\\wley'}`|Filters ACL entries to permissions granted to `wley`.|
|`$sid2 = Convert-NameToSid damundsen`|Converts `damundsen` into a SID for another targeted ACL search.|
|`Get-DomainObjectACL -ResolveGUIDs -Identity * \| ? {$_.SecurityIdentifier -eq $sid2} -Verbose`|Finds objects over which `damundsen` has rights and resolves GUIDs.|
|`Get-DomainGroup -Identity "Help Desk Level 1" \| select memberof`|Checks which groups the `Help Desk Level 1` group belongs to.|
|`$itgroupsid = Convert-NameToSid "Information Technology"`|Converts the `Information Technology` group name into its SID.|
|`Get-DomainObjectACL -ResolveGUIDs -Identity * \| ? {$_.SecurityIdentifier -eq $itgroupsid} -Verbose`|Finds objects controlled by the `Information Technology` group.|
|`$adunnsid = Convert-NameToSid adunn`|Converts `adunn` into a SID for targeted ACL enumeration.|
|`Get-DomainObjectACL -ResolveGUIDs -Identity * \| ? {$_.SecurityIdentifier -eq $adunnsid} -Verbose`|Finds objects controlled by `adunn` and resolves ACE GUIDs.|

---

# 🧭 3. Tool Roles & Differences

ACL enumeration can be performed manually with built-in PowerShell or more efficiently with PowerView, while BloodHound provides a different capability: **correlating the resulting relationships into attack paths**.

|Tool / Technique|Main purpose|Mental model|
|---|---|---|
|[PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)|Enumerates AD objects, ACLs, groups, users, and relationships|🔎 "Show me the permissions."|
|`Get-Acl`|Uses built-in PowerShell functionality to retrieve security descriptors/ACLs|🧰 "Use what is already installed."|
|`Get-ADUser`|Retrieves AD user objects|👤 "Give me the users to inspect."|
|`Get-ADObject`|Queries arbitrary AD objects, including control-access-right definitions|🧩 "Look up the AD object behind this GUID."|
|[BloodHound](https://bloodhound.specterops.io/)|Visualizes relationships and transitive attack paths|🕸️ "Show me how the permissions connect."|
|[SharpHound](https://github.com/SpecterOps/SharpHound)|Collects AD relationship data for BloodHound|📡 "Collect the graph data."|

### 🧠 The Big Picture

```text
                         ACTIVE DIRECTORY
                                │
                                ↓
                     CURRENT CONTROLLED USER
                                │
                                ↓
                    ┌───────────────────────┐
                    │   ACL ENUMERATION     │
                    └───────────┬───────────┘
                                │
                 ┌──────────────┼──────────────┐
                 ↓              ↓              ↓
             PowerView       Get-Acl       BloodHound
                 │              │              │
          Fast targeted      Built-in       Correlate
          enumeration        enumeration    relationships
                 │              │              │
                 └──────────────┼──────────────┘
                                ↓
                         CONTROLLED OBJECT
                                │
                                ↓
                           ACE / RIGHT
                                │
                                ↓
                       FOLLOW RELATIONSHIP
                                │
                                ↓
                        NEW USER / GROUP
                                │
                                ↓
                         NEW ACL RIGHTS
                                │
                                ↓
                         ATTACK PATH
```

---

## 🎯 3.1 Why These Tools Are Not Equivalent

### 🛠️ PowerView

Use PowerView when you want **targeted, command-line AD enumeration**.

```text
PowerView
   ↓
Query AD
   ↓
Find ACL
   ↓
Filter by SID
   ↓
Identify controlled object
```

It is especially useful when you already have a foothold and want to answer:

> **"What objects does this particular account have rights over?"**

---

### 🧰 Built-in PowerShell

`Get-Acl`, `Get-ADUser`, and `Get-ADObject` are useful when you **cannot or do not want to bring PowerView onto the host**.

Mental model:

```text
CLIENT SYSTEM
     ↓
Existing Microsoft tools
     ↓
Get-ADUser
Get-Acl
Get-ADObject
     ↓
ACL information
```

> [!important]  
> The built-in approach is generally more cumbersome and slower for this workflow, but it is valuable because it reduces your dependency on third-party tooling.

---

### 🕸️ BloodHound

BloodHound answers a higher-level question:

> **"Given everything we've collected, what relationships and attack paths connect my current access to valuable objects?"**

```text
SharpHound
    ↓
COLLECT
    ↓
AD relationship data
    ↓
BloodHound
    ↓
CORRELATE / VISUALIZE
    ↓
ATTACK PATHS
```

### 🧠 CPTS Mental Model

```text
PowerView / Built-in PowerShell
              ↓
          ENUMERATE
              ↓
        "What do I control?"
              ↓
         BloodHound
              ↓
          CORRELATE
              ↓
     "Where can this lead?"
              ↓
          ATTACK PATH
```

---

# 🔎 4. Why Broad ACL Enumeration Is a Problem

A tempting first step is:

```powershell
Find-InterestingDomainAcl
```

This can return a **huge amount of information**.

Example:

```text
ObjectDN                : DC=INLANEFREIGHT,DC=LOCAL
AceQualifier            : AccessAllowed
ActiveDirectoryRights   : ExtendedRight
ObjectAceType           : ab721a53-1e2f-11d0-9819-00aa0040529b
AceFlags                : ContainerInherit
AceType                 : AccessAllowedObject
InheritanceFlags        : ContainerInherit
SecurityIdentifier      : S-1-5-21-3842939050-3880317879-2865463114-5189
IdentityReferenceName   : Exchange Windows Permissions
IdentityReferenceDomain : INLANEFREIGHT.LOCAL
IdentityReferenceDN     : CN=Exchange Windows Permissions,OU=Microsoft Exchange Security
                          Groups,DC=INLANEFREIGHT,DC=LOCAL
IdentityReferenceClass  : group

ObjectDN                : DC=INLANEFREIGHT,DC=LOCAL
AceQualifier            : AccessAllowed
ActiveDirectoryRights   : ExtendedRight
ObjectAceType           : 00299570-246d-11d0-a768-00aa006e0529
AceFlags                : ContainerInherit
AceType                 : AccessAllowedObject
InheritanceFlags        : ContainerInherit
SecurityIdentifier      : S-1-5-21-3842939050-3880317879-2865463114-5189
IdentityReferenceName   : Exchange Windows Permissions
IdentityReferenceDomain : INLANEFREIGHT.LOCAL
IdentityReferenceDN     : CN=Exchange Windows Permissions,OU=Microsoft Exchange Security
                          Groups,DC=INLANEFREIGHT,DC=LOCAL
IdentityReferenceClass  : group

<SNIP>
```

> [!warning]  
> **Broad enumeration ≠ useful enumeration.**
> 
> During a time-boxed assessment, dumping every interesting ACL and manually reviewing the results can consume significant time.
> 
> A better approach is to start from a principal you **already control** and ask:
> 
> **"What objects does this principal control?"**

---

# 🎯 5. Targeted ACL Enumeration with PowerView

Assume we currently control:

```text
wley
```

The first step is to obtain the user's SID.

## 5.1 Import PowerView

```powershell
Import-Module .\PowerView.ps1
```

This loads PowerView into the current PowerShell session.

---

## 5.2 Convert the Username to a SID

```powershell
$sid = Convert-NameToSid wley
```

Mental model:

```text
wley
 │
 ↓
Convert-NameToSid
 │
 ↓
S-1-5-21-...-....
 │
 ↓
Use SID to identify ACEs belonging to wley
```

### 🧠 Why use the SID?

ACL entries identify security principals using their **security identifiers**.

Therefore, instead of asking vaguely:

> "Show me interesting ACLs."

we can ask:

> **"Show me ACL entries where the security identifier belongs to `wley`."**

This dramatically narrows the search.

---

# 🔍 6. Get-DomainObjectACL

The core PowerView function for this workflow is:

```powershell
Get-DomainObjectACL
```

We can query all domain objects:

```powershell
Get-DomainObjectACL -Identity *
```

Then filter for the SID belonging to `wley`:

```powershell
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

The important part is:

```powershell
$_.SecurityIdentifier -eq $sid
```

This means:

```text
Every ACL entry
      ↓
Check SecurityIdentifier
      ↓
Does it equal wley's SID?
      ↓
YES → keep result
NO  → discard result
```

---

# 🧩 7. The GUID Problem

Without `-ResolveGUIDs`, the output may contain:

```text
ActiveDirectoryRights : ExtendedRight
ObjectAceType         : 00299570-246d-11d0-a768-00aa006e0529
```

This is technically useful but not immediately readable.

For example:

```text
ExtendedRight
     +
00299570-246d-11d0-a768-00aa006e0529
```

doesn't immediately tell us:

> **"This is the right to force another user's password to be changed."**

---

## 7.1 Example Output

```powershell
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

```text
ObjectDN               : CN=Dana Amundsen,OU=DevOps,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
ObjectSID              : S-1-5-21-3842939050-3880317879-2865463114-1176
ActiveDirectoryRights  : ExtendedRight
ObjectAceFlags         : ObjectAceTypePresent
ObjectAceType          : 00299570-246d-11d0-a768-00aa006e0529
InheritedObjectAceType : 00000000-0000-0000-0000-000000000000
BinaryLength           : 56
AceQualifier            : AccessAllowed
IsCallback              : False
OpaqueLength            : 0
AccessMask              : 256
SecurityIdentifier     : S-1-5-21-3842939050-3880317879-2865463114-1181
AceType                 : AccessAllowedObject
AceFlags                : ContainerInherit
IsInherited             : False
InheritanceFlags        : ContainerInherit
PropagationFlags        : None
AuditFlags              : None
```

The interesting relationship is:

```text
wley
 ↓
ACE
 ↓
damundsen
 ↓
ObjectAceType = 00299570-246d-11d0-a768-00aa006e0529
```

We now need to translate the GUID.

---

# 🔄 8. Mapping an ACE GUID to a Human-Readable Right

There are two approaches.

## Method 1 — External Lookup

The GUID:

```text
00299570-246d-11d0-a768-00aa006e0529
```

corresponds to the Microsoft Active Directory extended right:

```text
User-Force-Change-Password
```

Microsoft documents this right as **Reset Password**.

Reference: [User-Force-Change-Password](https://learn.microsoft.com/en-us/windows/win32/adschema/r-user-force-change-password)

So we can translate:

```text
ObjectAceType
       ↓
00299570-246d-11d0-a768-00aa006e0529
       ↓
User-Force-Change-Password
       ↓
Reset Password
```

---

# 🔬 9. Reverse GUID Lookup with PowerShell

Instead of searching externally, we can query AD's own **Extended-Rights** container.

First:

```powershell
$guid= "00299570-246d-11d0-a768-00aa006e0529"
```

Then:

```powershell
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * |Select Name,DisplayName,DistinguishedName,rightsGuid| ?{$_.rightsGuid -eq $guid} | fl
```

Output:

```text
Name              : User-Force-Change-Password
DisplayName       : Reset Password
DistinguishedName : CN=User-Force-Change-Password,CN=Extended-Rights,CN=Configuration,DC=INLANEFREIGHT,DC=LOCAL
rightsGuid        : 00299570-246d-11d0-a768-00aa006e0529
```

### 🧠 What happened?

```text
GUID
 ↓
Search Extended-Rights
 ↓
Find ControlAccessRight
 ↓
Match rightsGuid
 ↓
Return Name / DisplayName
```

This is useful because it demonstrates that the GUID isn't arbitrary data: it identifies an **AD control access right** stored in the directory.

> [!important]  
> The source notes that if PowerView has already been imported, this particular `Get-ADObject` example may produce an error in the lab environment. A fresh PowerShell session may be required.

---

# ⚡ 10. The Efficient Method: `-ResolveGUIDs`

PowerView can perform this mapping for us.

Instead of manually translating every GUID:

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

Now the output becomes:

```text
AceQualifier           : AccessAllowed
ObjectDN               : CN=Dana Amundsen,OU=DevOps,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
ActiveDirectoryRights  : ExtendedRight
ObjectAceType          : User-Force-Change-Password
ObjectSID              : S-1-5-21-3842939050-3880317879-2865463114-1176
InheritanceFlags       : ContainerInherit
BinaryLength           : 56
AceType                : AccessAllowedObject
ObjectAceFlags         : ObjectAceTypePresent
PropagationFlags       : None
SecurityIdentifier     : S-1-5-21-3842939050-3880317879-2865463114-1181
AccessMask             : 256
AuditFlags             : None
IsInherited             : False
AceFlags               : ContainerInherit
InheritedObjectAceType : All
OpaqueLength           : 0
```

Now the important line is immediately understandable:

```text
ObjectAceType : User-Force-Change-Password
```

Therefore:

```text
wley
  │
  │ User-Force-Change-Password
  ↓
damundsen
```

> [!tip]  
> **For practical enumeration, use `-ResolveGUIDs`.**
> 
> But understand the manual GUID lookup process anyway. CPTS is testing whether you understand the underlying AD mechanism, not just whether you know which switch to type.

---

# 🧠 11. Why Learn the Manual Method?

The question is not:

> "Why didn't we just use `-ResolveGUIDs`?"

The reason is **tool independence**.

A professional assessment can involve:

- Restricted systems
    
- Missing tools
    
- Blocked scripts
    
- Client-owned hosts
    
- Limited permissions
    
- EDR/tooling restrictions
    
- Situations where you cannot simply import your preferred toolkit
    

Therefore:

```text
Preferred tool
      ↓
Unavailable?
      ↓
Alternative method
      ↓
Built-in functionality
      ↓
Same information
```

The goal is to understand the **operation underneath the tool**.

---

# 🧰 12. ACL Enumeration Without PowerView

We can perform a similar investigation using built-in Microsoft PowerShell functionality.

The relevant tools are:

- `Get-ADUser`
    
- `Get-Acl`
    
- `Get-ADObject`
    

This is particularly useful when working directly on a client system where importing PowerView is undesirable or unavailable.

> [!important]  
> The built-in method is **less efficient** and can take considerably longer in a large environment. Its value is primarily **tool independence**, not speed.

---

# 👥 13. Creating a Domain User List

First, enumerate all domain users and save their SAM account names:

```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
```

This creates:

```text
ad_users.txt
```

containing usernames such as:

```text
user1
user2
user3
...
```

Mental model:

```text
Get-ADUser
     ↓
All domain users
     ↓
SamAccountName
     ↓
ad_users.txt
```

---

# 🔄 14. Using `foreach` + `Get-Acl`

The source then loops through every username:

```powershell
foreach($line in [System.IO.File]::ReadLines("C:\Users\htb-student\Desktop\ad_users.txt")) {get-acl  "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match 'INLANEFREIGHT\\wley'}}
```

### 🧠 Break the command apart

```text
ReadLines()
    ↓
Read each username
    ↓
Get-ADUser
    ↓
Resolve username to AD object
    ↓
Get-Acl
    ↓
Retrieve ACL
    ↓
Expand Access property
    ↓
Filter IdentityReference
    ↓
Keep entries belonging to wley
```

The result:

```text
Path                  : Microsoft.ActiveDirectory.Management.dll\ActiveDirectory:://RootDSE/CN=Dana
                        Amundsen,OU=DevOps,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL

ActiveDirectoryRights : ExtendedRight
InheritanceType       : All
ObjectType            : 00299570-246d-11d0-a768-00aa006e0529
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : INLANEFREIGHT\wley
IsInherited           : False
InheritanceFlags      : ContainerInherit
PropagationFlags      : None
```

Notice again:

```text
ObjectType : 00299570-246d-11d0-a768-00aa006e0529
```

We can then perform the same GUID lookup described earlier.

---

# ⚖️ 15. PowerView vs Built-in PowerShell

|Method|Strength|Weakness|When to use|
|---|---|---|---|
|**PowerView**|Fast, targeted AD enumeration|Requires importing/using PowerView|Preferred when tooling is available|
|**Get-Acl + Get-ADUser**|Built into many Windows environments|Slow and cumbersome for large-scale enumeration|When third-party tooling is unavailable/restricted|
|**Manual GUID lookup**|Shows exactly how AD control rights are represented|Time-consuming|Troubleshooting / tool-independent workflow|
|**PowerView `-ResolveGUIDs`**|Automatically translates GUIDs|Depends on PowerView|Efficient day-to-day enumeration|
|**BloodHound**|Correlates relationships and attack paths visually|Requires collection/import of data|Finding transitive attack paths|

### 🧠 Exam Mental Model

```text
Need speed?
    ↓
PowerView

No PowerView?
    ↓
Get-ADUser + Get-Acl + Get-ADObject

Need to understand GUID?
    ↓
Query Extended-Rights

Need attack-path visibility?
    ↓
BloodHound
```

---

# 🔗 16. Following the ACL Chain

The really important part of this exercise begins **after** finding the first ACE.

We discovered:

```text
wley
  │
  │ ForceChangePassword
  ↓
damundsen
```

That does not mean we stop.

The next question is:

> **"What can damundsen control?"**

This is where ACL enumeration becomes an **iterative attack-path methodology**.

---

# 🔎 17. Enumerating Rights of `damundsen`

Convert `damundsen` to a SID:

```powershell
$sid2 = Convert-NameToSid damundsen
```

Then search for objects controlled by that SID:

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid2} -Verbose
```

Relevant output:

```text
AceType               : AccessAllowed
ObjectDN              : CN=Help Desk Level 1,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
ActiveDirectoryRights : ListChildren, ReadProperty, GenericWrite
OpaqueLength          : 0
ObjectSID             : S-1-5-21-3842939050-3880317879-2865463114-4022
InheritanceFlags      : ContainerInherit
BinaryLength          : 36
IsInherited           : False
PropagationFlags      : None
SecurityIdentifier    : S-1-5-21-3842939050-3880317879-2865463114-1176
AccessMask             : 131132
AuditFlags             : None
AceFlags               : ContainerInherit
AceQualifier           : AccessAllowed
```

The important relationship is:

```text
damundsen
    │
    │ GenericWrite
    ↓
Help Desk Level 1
```

---

# 👥 18. Why Group Membership Matters

`Help Desk Level 1` is a **group**, so we should not only inspect the group's direct permissions.

We should also ask:

> **"What groups is Help Desk Level 1 a member of?"**

Why?

Because AD groups can be **nested**.

```text
User
 ↓
Help Desk Level 1
 ↓
Information Technology
 ↓
Permissions assigned to Information Technology
```

Therefore, membership in the lower-level group can indirectly provide the privileges of the higher-level group.

---

# 🔗 19. Enumerating Nested Group Membership

Use:

```powershell
Get-DomainGroup -Identity "Help Desk Level 1" | select memberof
```

Output:

```text
memberof
--------
CN=Information Technology,OU=Security Groups,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
```

This tells us:

```text
Help Desk Level 1
        │
        │ MemberOf
        ↓
Information Technology
```

### 🧠 Critical AD Mental Model

```text
GROUP A
  │
  │ MemberOf
  ↓
GROUP B
  │
  ↓
GROUP B's permissions
```

If an account becomes a member of **Group A**, its effective group memberships can include **Group B** and therefore the permissions granted through that membership chain.

> [!important]  
> **Never stop at the first group.**
> 
> When ACL enumeration leads to a group, always consider:
> 
> **Who is this group a member of?**
> 
> Nested groups can transform a seemingly ordinary permission into a much more valuable attack path.

---

# 🧬 20. The First Complete Chain

At this point, our path becomes:

```text
wley
 │
 │ ForceChangePassword
 ↓
damundsen
 │
 │ GenericWrite
 ↓
Help Desk Level 1
 │
 │ MemberOf
 ↓
Information Technology
```

This is much more meaningful than looking at each ACE individually.

We have transformed:

```text
4 separate AD facts
```

into:

```text
1 connected attack path
```

---

# 👑 21. Enumerating `Information Technology`

Now convert the group to a SID:

```powershell
$itgroupsid = Convert-NameToSid "Information Technology"
```

Then enumerate its ACL relationships:

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $itgroupsid} -Verbose
```

Relevant output:

```text
AceType               : AccessAllowed
ObjectDN              : CN=Angela Dunn,OU=Server Admin,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
ActiveDirectoryRights : GenericAll
OpaqueLength          : 0
ObjectSID             : S-1-5-21-3842939050-3880317879-2865463114-1164
InheritanceFlags      : ContainerInherit
BinaryLength          : 36
IsInherited           : False
PropagationFlags      : None
SecurityIdentifier    : S-1-5-21-3842939050-3880317879-2865463114-4016
AccessMask            : 983551
AuditFlags             : None
AceFlags              : ContainerInherit
AceQualifier           : AccessAllowed
```

Now we have:

```text
Information Technology
          │
          │ GenericAll
          ↓
       adunn
```

---

# 🧨 22. Understanding the `GenericAll` Relationship

`GenericAll` means the group has broad/full control over the target object.

For the `adunn` user, the source identifies several potential consequences:

- Modify group membership
    
- Force-change the password
    
- Perform targeted Kerberoasting and attempt offline cracking if conditions permit
    

So our chain is now:

```text
wley
 │
 │ ForceChangePassword
 ↓
damundsen
 │
 │ GenericWrite
 ↓
Help Desk Level 1
 │
 │ MemberOf
 ↓
Information Technology
 │
 │ GenericAll
 ↓
adunn
```

> [!important]  
> This is the central lesson of ACL enumeration:
> 
> **The first permission you discover may not be the final objective.**
> 
> It may simply be the first link in a chain of relationships.

---

# 🔎 23. Enumerating `adunn`

Finally, convert `adunn` into a SID:

```powershell
$adunnsid = Convert-NameToSid adunn
```

Then search for objects over which `adunn` has rights:

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $adunnsid} -Verbose
```

Relevant output:

```text
AceQualifier           : AccessAllowed
ObjectDN               : DC=INLANEFREIGHT,DC=LOCAL
ActiveDirectoryRights  : ExtendedRight
ObjectAceType          : DS-Replication-Get-Changes-In-Filtered-Set
ObjectSID              : S-1-5-21-3842939050-3880317879-2865463114
InheritanceFlags       : ContainerInherit
BinaryLength           : 56
AceType                : AccessAllowedObject
ObjectAceFlags         : ObjectAceTypePresent
IsCallback             : False
PropagationFlags       : None
SecurityIdentifier     : S-1-5-21-3842939050-3880317879-2865463114-1164
AccessMask             : 256
AuditFlags             : None
IsInherited             : False
AceFlags                : ContainerInherit
InheritedObjectAceType : All
OpaqueLength            : 0

AceQualifier           : AccessAllowed
ObjectDN               : DC=INLANEFREIGHT,DC=LOCAL
ActiveDirectoryRights  : ExtendedRight
ObjectAceType           : DS-Replication-Get-Changes
ObjectSID              : S-1-5-21-3842939050-3880317879-2865463114
InheritanceFlags       : ContainerInherit
BinaryLength            : 56
AceType                : AccessAllowedObject
ObjectAceFlags         : ObjectAceTypePresent
IsCallback             : False
PropagationFlags       : None
SecurityIdentifier     : S-1-5-21-3842939050-3880317879-2865463114-1164
AccessMask              : 256
AuditFlags             : None
IsInherited             : False
AceFlags                : ContainerInherit
InheritedObjectAceType : All
OpaqueLength            : 0
```

The important rights are:

```text
DS-Replication-Get-Changes
DS-Replication-Get-Changes-In-Filtered-Set
```

over:

```text
DC=INLANEFREIGHT,DC=LOCAL
```

These are replication-related rights that can form part of the permissions required for a **DCSync** attack.

The source identifies this as the point where `adunn` can be leveraged for DCSync, which is covered in depth separately.

---

# 🧠 24. Complete Manual Attack-Path Methodology

The entire enumeration process can now be represented as:

```text
                 CONTROLLED ACCOUNT
                        │
                        ↓
                  Convert to SID
                        │
                        ↓
              Get-DomainObjectACL
                        │
                        ↓
               Filter by SecurityIdentifier
                        │
                        ↓
                  CONTROLLED OBJECT
                        │
                        ↓
                  Identify ACE/right
                        │
                        ↓
              ┌─────────┴─────────┐
              │                   │
            USER                 GROUP
              │                   │
              ↓                   ↓
       Enumerate user       Check nested groups
          permissions              │
              │                    ↓
              │              New security principal
              │                    │
              └──────────┬─────────┘
                         ↓
                  Convert to SID
                         ↓
                Enumerate again
                         ↓
                  NEW OBJECT / RIGHT
                         ↓
                      Repeat
                         ↓
                    Attack Path
```

### 🎯 The key loop

```text
ENUMERATE
    ↓
FIND CONTROL
    ↓
IDENTIFY RIGHT
    ↓
FOLLOW TARGET
    ↓
RE-ENUMERATE TARGET
    ↓
FIND NEXT CONTROL
    ↓
REPEAT
```

This is the **ACL enumeration loop**.

---

# 🕸️ 25. BloodHound: Doing the Same Job Visually

The manual PowerView process works, but BloodHound can make the same relationship chain much easier to understand.

The workflow is:

```text
SharpHound
    ↓
Collect AD relationship data
    ↓
Upload data
    ↓
BloodHound
    ↓
Select starting user
    ↓
Inspect Outbound Control Rights
    ↓
Inspect Transitive Object Control
    ↓
Follow attack path
```

---

# 🎯 26. Starting from `wley` in BloodHound

After importing the SharpHound data:

1. Select the `wley` user.
    
2. Open **Node Info**.
    
3. Locate **Outbound Control Rights**.
    

This provides information about objects that `wley` can control, either:

- Directly
    
- Through group membership
    
- Through transitive ACL relationships
    

The key sections are:

```text
Outbound Control Rights
        │
        ├── First Degree Object Control
        │
        └── Transitive Object Control
```

### 🧠 First Degree vs Transitive

```text
FIRST DEGREE
    ↓
Direct relationship

TRANSITIVE
    ↓
Multiple relationships
    ↓
Potential attack path
```

For the lab:

```text
wley
  │
  │ ForceChangePassword
  ↓
damundsen
```

appears as the first-degree control relationship.

---

# 🔎 27. BloodHound Edge Help

If we right-click the relationship between the two objects and choose **Help**, BloodHound provides information about the edge.

This can include:

- Information about the specific right
    
- Tools that can be used
    
- Commands associated with the abuse
    
- Operational-security considerations
    
- External references
    

Mental model:

```text
BloodHound Edge
      ↓
"What does this permission mean?"
      ↓
Help
      ↓
Understand the ACE
      ↓
Determine abuse method
```

> [!tip]  
> BloodHound's edge documentation is useful not only for discovering paths but also for learning **what a particular relationship actually means**.

---

# 🧬 28. Viewing the Entire Transitive Path

The `Transitive Object Control` section can reveal the chain we manually constructed.

Conceptually:

```text
wley
 │
 │ ForceChangePassword
 ↓
damundsen
 │
 │ GenericWrite
 ↓
Help Desk Level 1
 │
 │ MemberOf
 ↓
Information Technology
 │
 │ GenericAll
 ↓
adunn
 │
 │ Replication rights
 ↓
DOMAIN
```

The important difference is:

```text
PowerView
    ↓
You manually build the chain

BloodHound
    ↓
Graph makes the chain visible
```

---

# 🧭 29. BloodHound Pre-Built Queries

BloodHound also provides pre-built queries that can help confirm important relationships.

In this example, the pre-built query can reveal that:

```text
adunn
  ↓
DS-Replication-Get-Changes
  +
DS-Replication-Get-Changes-In-Filtered-Set
  ↓
DOMAIN
```

This confirms that the final user in the chain has replication-related privileges relevant to **DCSync**.

---

# ⚖️ 30. Manual Enumeration vs BloodHound

|Aspect|PowerView / PowerShell|BloodHound|
|---|---|---|
|Primary role|Enumerate AD data|Correlate and visualize AD relationships|
|ACL discovery|Excellent|Excellent when data is collected|
|Filtering one user's SID|Very useful|Graph-based|
|GUID resolution|`-ResolveGUIDs`|Usually represented as named edges|
|Nested-group analysis|Manual commands|Visual relationship graph|
|Transitive attack paths|Must be followed manually|Can be visualized directly|
|Tool availability|PowerView or built-in PowerShell|Requires collector/data and BloodHound|
|Best mental model|🔎 **Inspect**|🕸️ **Correlate**|

---

# 🔄 31. PowerView → BloodHound Relationship

These tools can be used together:

```text
                    AD ENVIRONMENT
                          │
                          ↓
                    SharpHound
                          │
                          ↓
                  COLLECT RELATIONSHIPS
                          │
                          ↓
                      BloodHound
                          │
                          ↓
                 IDENTIFY ATTACK PATH
                          │
                          ↓
                    PowerView
                          │
                          ↓
              VERIFY / ENUMERATE DETAILS
                          │
                          ↓
                    ATTACK / ACCESS
```

Neither tool needs to replace the other.

> [!important]  
> **BloodHound is excellent for seeing the path. PowerView is excellent for interrogating the underlying AD objects and permissions.**

---

# 🧠 32. The `wley → DCSync` Attack Path

The complete example from this section can be visualized as:

```text
┌─────────────────────────────┐
│            wley             │
│       CURRENT CONTROL       │
└──────────────┬──────────────┘
               │
               │ ForceChangePassword
               ↓
┌─────────────────────────────┐
│          damundsen          │
└──────────────┬──────────────┘
               │
               │ GenericWrite
               ↓
┌─────────────────────────────┐
│     Help Desk Level 1       │
└──────────────┬──────────────┘
               │
               │ MemberOf
               ↓
┌─────────────────────────────┐
│   Information Technology    │
└──────────────┬──────────────┘
               │
               │ GenericAll
               ↓
┌─────────────────────────────┐
│           adunn             │
└──────────────┬──────────────┘
               │
               │ Replication Rights
               ↓
┌─────────────────────────────┐
│          DOMAIN             │
│   DCSync-relevant rights    │
└─────────────────────────────┘
```

### 🧠 Why this matters

No single step necessarily looks like:

```text
"Domain compromise"
```

Instead:

```text
Small permission
     ↓
User control
     ↓
Group control
     ↓
Nested group membership
     ↓
More powerful group
     ↓
Powerful user
     ↓
Domain-level rights
```

This is exactly why **ACL enumeration must be viewed as an attack-path problem**.

---

# 🧭 33. CPTS ACL Enumeration Decision Tree

```text
START
  │
  ↓
Do I control an AD user?
  │
  ├── NO → Obtain / identify valid context first
  │
  └── YES
       │
       ↓
   Convert username → SID
       │
       ↓
   Search ACLs for SID
       │
       ↓
   Found ACE?
       │
   ┌───┴────┐
   NO       YES
   │         │
   ↓         ↓
Continue   Identify
enum       ObjectAceType
              │
              ↓
         GUID or name?
          │        │
        GUID     Name
          │        │
          ↓        ↓
   Resolve GUID   Understand
          │        │
          └───┬────┘
              ↓
        Identify target
              │
              ↓
        USER / GROUP /
        COMPUTER / DOMAIN
              │
              ↓
        What does right
           allow?
              │
              ↓
       Re-enumerate target
              │
              ↓
       New group membership?
              │
              ├── YES → Follow nesting
              │
              └── NO
                   │
                   ↓
              New ACL rights?
                   │
                   ↓
              Follow relationship
                   │
                   ↓
             New attack path?
                   │
                   ↓
                 REPEAT
```

---

# 📊 34. Important PowerView Workflow

The most reusable PowerView pattern from this section is:

```powershell
# 1. Convert controlled principal to SID
$sid = Convert-NameToSid <USER_OR_GROUP>

# 2. Search ACLs for that SID
Get-DomainObjectACL -ResolveGUIDs -Identity * |
    ? {$_.SecurityIdentifier -eq $sid}
```

Then:

```text
Read result
    ↓
Identify target
    ↓
Identify right
    ↓
Ask "what can this right do?"
    ↓
If target is a group → inspect group nesting
    ↓
If target is a user → enumerate user's rights
    ↓
If target is a computer → inspect computer-specific paths
    ↓
If target is domain → inspect domain-level rights
```

> [!tip]  
> The most important command is not just `Get-DomainObjectACL`.
> 
> The reusable technique is:
> 
> **Convert principal → SID → search ACLs by `SecurityIdentifier` → follow every useful relationship.**

---

# 🧠 35. What the Important Properties Mean

When reading PowerView ACL output, focus on these fields:

|Property|Meaning|
|---|---|
|`ObjectDN`|The AD object that the ACE applies to.|
|`ObjectSID`|SID of the target object.|
|`SecurityIdentifier`|SID of the principal receiving the permission.|
|`ActiveDirectoryRights`|Rights granted by the ACE.|
|`ObjectAceType`|Object-specific/control-access right; may appear as a GUID without `-ResolveGUIDs`.|
|`AceType`|Type of ACE, such as `AccessAllowed` or an object-specific allowed ACE.|
|`AceQualifier`|Indicates the access-control classification.|
|`IsInherited`|Indicates whether the ACE was inherited.|
|`InheritanceFlags`|Describes inheritance behavior.|
|`AccessMask`|Numeric representation of the access rights.|
|`InheritedObjectAceType`|Specifies object types to which an inherited ACE can apply.|

### 🎯 Most important fields for attack-path enumeration

When reviewing output manually, prioritize:

```text
SecurityIdentifier
        ↓
WHO has the permission?

ObjectDN
        ↓
WHAT object do they control?

ActiveDirectoryRights
        ↓
WHAT kind of access?

ObjectAceType
        ↓
WHAT specific right?

IsInherited / InheritanceFlags
        ↓
WHERE does the permission apply?
```

---

# ⚠️ 36. Common Enumeration Mistakes

## ❌ Mistake 1 — Dumping Everything

```powershell
Find-InterestingDomainAcl
```

and then attempting to manually inspect thousands of results.

### Better:

```text
Start with current foothold
        ↓
Get SID
        ↓
Filter ACLs
```

---

## ❌ Mistake 2 — Ignoring GUIDs

Seeing:

```text
ObjectAceType : 00299570-246d-11d0-a768-00aa006e0529
```

and treating it as meaningless.

### Better:

```text
GUID
 ↓
Resolve
 ↓
Understand the actual right
```

Use:

```powershell
-ResolveGUIDs
```

when possible.

---

## ❌ Mistake 3 — Stopping at the First Object

Finding:

```text
wley → damundsen
```

and stopping.

### Better:

```text
wley
 ↓
damundsen
 ↓
What does damundsen control?
 ↓
Group?
 ↓
What is that group a member of?
 ↓
What does the parent group control?
 ↓
Repeat
```

---

## ❌ Mistake 4 — Ignoring Group Nesting

Finding:

```text
damundsen
 ↓
GenericWrite
 ↓
Help Desk Level 1
```

but never checking:

```powershell
Get-DomainGroup -Identity "Help Desk Level 1" | select memberof
```

### Better:

Always ask:

> **"What groups is this group a member of?"**

---

## ❌ Mistake 5 — Treating BloodHound as a Magic Button

BloodHound can reveal paths, but understanding **why the path works** remains essential.

```text
BloodHound says:
GenericAll
     ↓
You should know:
"What does GenericAll mean against this target?"
```

> [!important]  
> Use BloodHound to **accelerate understanding**, not replace understanding.

---

# 🧠 37. Final Mental Model

ACL enumeration is essentially **walking the control graph** of Active Directory.

```text
                         YOU
                          │
                          ↓
                    YOUR SID
                          │
                          ↓
                    ACL SEARCH
                          │
                          ↓
                 ┌────────┴────────┐
                 │                 │
               USER              GROUP
                 │                 │
                 ↓                 ↓
             USER ACLS       GROUP NESTING
                 │                 │
                 └────────┬────────┘
                          ↓
                     NEXT OBJECT
                          │
                          ↓
                     NEXT ACL
                          │
                          ↓
                   NEXT CAPABILITY
                          │
                          ↓
                    NEXT OBJECT
                          │
                         ...
                          ↓
                   ATTACK OBJECTIVE
```

The central question is always:

> **"Given the security principal I currently control, what objects can it control, and what does that control allow me to reach next?"**

---

# 🏁 38. Key Takeaways

> [!success]  
> **1. Start from what you control.**
> 
> Convert the controlled user's/group's name into a SID and search ACLs specifically for that SID.

> [!success]  
> **2. Prefer targeted enumeration.**
> 
> `Find-InterestingDomainAcl` can produce too much data. Filtering `Get-DomainObjectACL` by `SecurityIdentifier` is much more useful during a time-boxed assessment.

> [!success]  
> **3. Resolve GUIDs.**
> 
> `ObjectAceType` may initially be an unreadable GUID. `-ResolveGUIDs` translates it into a human-readable right.

> [!success]  
> **4. Know the fallback.**
> 
> `Get-ADUser` + `Get-Acl` + `Get-ADObject` can reproduce much of the workflow without PowerView, although less efficiently.

> [!success]  
> **5. Never stop at the first ACE.**
> 
> Re-enumerate every newly controlled user/group/object.

> [!success]  
> **6. Follow nested groups.**
> 
> A seemingly ordinary group can be nested into a more privileged group and therefore provide an indirect route to additional rights.

> [!success]  
> **7. PowerView and BloodHound are complementary.**
> 
> **PowerView = inspect AD permissions.**
> 
> **BloodHound = correlate those permissions into relationships and attack paths.**

> [!success]  
> **8. ACL enumeration is a graph problem.**
> 
> The real skill is turning individual relationships into a connected path:
> 
> ```text
> wley
>  ↓
> ForceChangePassword
>  ↓
> damundsen
>  ↓
> GenericWrite
>  ↓
> Help Desk Level 1
>  ↓
> MemberOf
>  ↓
> Information Technology
>  ↓
> GenericAll
>  ↓
> adunn
>  ↓
> Replication Rights
>  ↓
> DCSync-relevant access
> ```

> [!tip]  
> **CPTS exam mindset:**
> 
> When you obtain valid AD credentials, don't only ask:
> 
> **"What services can I log into?"**
> 
> Also ask:
> 
> **"What does this principal control?"**
> 
> Then repeatedly:
> 
> **SID → ACL → Right → Target → Re-enumerate → Follow the graph.**