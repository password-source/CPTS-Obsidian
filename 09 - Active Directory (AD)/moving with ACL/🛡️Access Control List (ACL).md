# 🛡️ 1. Access Control List (ACL) Abuse Primer

> [!info]  
> **Objective:** Understand how Active Directory `ACLs`, `DACLs`, `SACLs`, and `ACEs` control access to AD objects, and how excessive or misconfigured permissions can create paths for **lateral movement, privilege escalation, and persistence**.
> 
> 🧠 **Core CPTS idea:** ACL abuse is often not about exploiting a software vulnerability. Instead, the attacker discovers that their current security principal has **more control over another AD object than it should**.

---
# 💻 2. Command Cheat Sheet

|Command|What it does|
|---|---|
|`Set-DomainUserPassword`|Resets the password of an AD user when the current principal has sufficient rights.|
|`Add-DomainGroupMember`|Adds a user or other principal to an AD group when permitted.|
|`Set-DomainObject`|Modifies writable attributes on an AD object.|
|`Set-DomainObjectOwner`|Changes the owner of an AD object when `WriteOwner` is available.|
|`Add-DomainObjectACL`|Modifies an object's ACL by adding an ACE/right when sufficient permissions exist.|
|`Get-DomainUser * -spn`|Enumerates domain users configured with SPNs.|
|`Get-DomainSPNTicket`|Requests/extracts Kerberos service tickets for SPNs using PowerView functionality.|
|`Add-DomainGroupMember`|Abuses group membership rights such as `AddMember` or `AddSelf`.|
|`Get-DomainObject`|Enumerates AD objects and their properties using PowerView.|
|`Get-DomainObjectAcl`|Enumerates ACLs/ACE information associated with AD objects using PowerView.|
|`Get-ObjectAcl`|Retrieves ACL information for AD objects in PowerView-based workflows.|
|`Get-DomainUser`|Enumerates domain user objects and their properties.|
|`Get-DomainGroup`|Enumerates domain groups and their properties.|

> [!important]  
> The source material names the abuse functions used for particular ACEs. The exact enumeration syntax can vary between PowerView versions and CPTS module examples, so preserve the **relationship between permission → abuse action** rather than memorizing one syntax in isolation.

**Official references:** [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) · [Microsoft ACL documentation](https://learn.microsoft.com/en-us/windows/win32/secauthz/access-control-lists)

---

# 🧭 3. Tool Roles & Differences

ACL attacks involve several different concepts and tools that can easily become mixed together.

|Tool / Concept|Main purpose|Mental model|
|---|---|---|
|**ACL**|Defines access-control information for an object|📜 "Who can do what?"|
|**ACE**|Individual permission/audit entry inside an ACL|🧩 "One permission rule"|
|**DACL**|Defines who is allowed/denied access|🚪 "Who gets through the door?"|
|**SACL**|Defines which access attempts are audited|📹 "What gets recorded?"|
|[PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)|Enumerates and interacts with AD objects and permissions|🔎 "Show me the AD security information"|
|[BloodHound](https://bloodhound.specterops.io/)|Visualizes relationships and attack paths involving AD permissions|🕸️ "How are these permissions connected?"|
|ADUC|Graphical Windows interface for viewing/modifying object permissions|🖥️ "Show me the permissions manually"|
|GMSAPasswordReader|Can retrieve gMSA password material when appropriate rights exist|🔑 "Turn a permission into credential access"|

### 🧠 The Big Picture

```text
                         ACTIVE DIRECTORY
                                │
                                ↓
                         SECURITY OBJECTS
                                │
                    ┌───────────┴───────────┐
                    │                       │
                  USERS                   GROUPS
                    │                       │
                    └───────────┬───────────┘
                                ↓
                              ACLs
                                │
                    ┌───────────┴───────────┐
                    │                       │
                  DACL                    SACL
                    │                       │
              Allow / Deny              Auditing
                    │
                    ↓
                  ACEs
                    │
              "Who can do what?"
                    │
                    ↓
             BloodHound / PowerView
                    │
                    ↓
              Find excessive rights
                    │
                    ↓
              Identify abuse path
                    │
             ┌──────┼───────┐
             ↓      ↓       ↓
          Password  Group   Object
           Reset   Member   Control
             │      │       │
             └──────┴───────┘
                    ↓
          Lateral Movement /
          Privilege Escalation /
               Persistence
```

### 🔎 What Each Component Is For

#### 🧩 ACL

An **Access Control List** is a collection of permission entries associated with a securable object.

It answers:

> **"Which security principals have which rights over this object?"**

An AD object can have multiple ACEs because multiple users/groups may have different permissions over the same object. Microsoft describes an ACL as an ordered collection of ACEs, with each ACE defining access or auditing for a specific trustee.

---

#### 🚪 DACL

A **Discretionary Access Control List** determines which security principals are **allowed or denied** access to an object.

Mental model:

```text
DACL
 │
 ├── Alice → Allow → Read
 ├── Bob   → Allow → Write
 ├── Group → Allow → Full Control
 └── Eve   → Deny  → Write
```

The DACL is therefore the part of the security descriptor that directly answers:

> **"Can this principal perform this operation?"**

If an object has **no DACL**, Windows grants full access to everyone. If a DACL exists but contains no ACEs, access is denied.

---

#### 📹 SACL

A **System Access Control List** is used for **auditing**, rather than deciding ordinary access.

Mental model:

```text
DACL → "Can you do it?"
SACL → "Should Windows record that you tried?"
```

A SACL can generate audit events when specified access attempts succeed, fail, or both.

---

#### 🧩 ACE

An **Access Control Entry** is an individual rule inside an ACL.

Think:

```text
ACL
 │
 ├── ACE #1 → Alice → Allow → Read
 ├── ACE #2 → Bob   → Allow → Write
 └── ACE #3 → Eve   → Deny  → Write
```

An ACE identifies a security principal and specifies the rights that apply to that principal.

Microsoft identifies three major ACE types relevant here:

|**ACE**|**Description**|
|---|---|
|`Access denied ACE`|Used within a DACL to show that a user or group is explicitly denied access to an object|
|`Access allowed ACE`|Used within a DACL to show that a user or group is explicitly granted access to an object|
|`System audit ACE`|Used within a SACL to generate audit logs when a user or group attempts to access an object. It records whether access was granted or not and what type of access occurred|

---

### 🎯 When Would I Use Which?

|Question I am trying to answer|Tool / Technique|
|---|---|
|"Who has permissions over this object?"|ACL / ACE enumeration|
|"Who can modify this user?"|DACL / ACE analysis|
|"Who can add members to this group?"|Group membership ACE analysis|
|"Who can reset this user's password?"|`ForceChangePassword` / `GenericAll` / relevant extended rights|
|"What relationships exist between principals and objects?"|BloodHound|
|"How can my current account reach a privileged object?"|BloodHound attack-path analysis|
|"How can I enumerate AD permissions from PowerShell?"|PowerView|
|"What access attempts are being logged?"|SACL / auditing|

---

# 🧠 4. Core ACL Mental Model

## 🔐 4.1 Think "Object → Principal → Right"

The easiest way to understand AD ACL abuse is to stop thinking of permissions as isolated labels.

Instead, think:

```text
OBJECT
  │
  ↓
WHO has a relationship with it?
  │
  ↓
SECURITY PRINCIPAL
  │
  ↓
WHAT RIGHT do they have?
  │
  ↓
WHAT can that right become?
  │
  ↓
ATTACK PATH
```

For example:

```text
forend
  │
  │ GenericAll
  ↓
Administrator
  │
  ↓
Full control over object
  │
  ├── Password manipulation
  ├── Attribute modification
  └── Other object-specific control
```

The important CPTS skill is therefore not simply recognizing `GenericAll`.

It is recognizing:

> **"I control X, X has this right over Y, therefore I may be able to manipulate Y."**

---

# 🧬 5. ACL Structure

## 5.1 Security Principals

An ACE can apply to a **security principal**, such as:

- User
    
- Group
    
- Computer
    
- Process/security context
    

A security principal is represented internally by a **SID**.

The principal is the "who" in:

```text
WHO → CAN DO WHAT → TO WHICH OBJECT?
```

---

## 5.2 The Four Major ACE Components

Each ACE contains information describing:

1. **Security Identifier (SID)**
    
    - Identifies the user/group/security principal.
        
2. **ACE type**
    
    - Allow
        
    - Deny
        
    - System audit
        
3. **Inheritance flags**
    
    - Determine whether the permission can apply to child objects/containers.
        
4. **Access mask**
    
    - A 32-bit value representing the rights granted or controlled by the ACE.
        

Microsoft's access-control documentation describes ACEs as containing the trustee/security identifier and access rights, while directory-service ACEs can also specify inheritance and object-specific rights.

### 🧠 Mental Model

```text
ACE
 │
 ├── WHO?
 │    └── SID / Principal
 │
 ├── WHAT TYPE?
 │    └── Allow / Deny / Audit
 │
 ├── WHERE DOES IT APPLY?
 │    └── Object / Child objects
 │
 └── WHAT CAN THEY DO?
      └── Access mask / rights
```

---

# ⚙️ 6. How DACL Access Checks Work

When a principal attempts to access an object, Windows evaluates the object's security information against the user's security context/token.

Simplified:

```text
User attempts action
        ↓
Windows obtains user's security token
        ↓
Token contains user's SID + group SIDs
        ↓
Windows checks object's DACL
        ↓
Relevant ACEs are evaluated
        ↓
Required rights allowed?
      /       \
    YES        NO
     ↓          ↓
  ACCESS      DENIED
```

Microsoft documents that access checks compare the trustees represented in the access token against the ACEs in the object's DACL.

> [!warning]  
> **ACE order matters.** Windows evaluates ACEs in sequence, and deny entries can affect whether access is granted. In canonical DACL ordering, explicit entries precede inherited entries, with deny entries ordered before grant entries within the applicable categories.

---

# 🧩 7. DACL vs SACL vs ACE

|Component|Purpose|Question it answers|
|---|---|---|
|**ACL**|Collection of access-control entries|"What security rules exist?"|
|**DACL**|Controls allowed/denied access|"Who can access this object and how?"|
|**SACL**|Controls auditing|"Which access attempts should be logged?"|
|**ACE**|Individual rule inside an ACL|"What does this specific principal get to do?"|

### 🧠 Remember

```text
ACL
 │
 ├── DACL
 │     └── Controls ACCESS
 │
 │       └── ACE
 │            └── Allow / Deny
 │
 └── SACL
       └── Controls AUDITING
             │
             └── ACE
                  └── Audit
```

---

# 🎯 8. Why ACLs Matter to Penetration Testers

ACLs are particularly interesting during an AD assessment because an account can have **unexpected control over another object** without being an obvious administrator.

This can create paths such as:

```text
LOW-PRIVILEGE USER
       ↓
Excessive ACE
       ↓
CONTROL OVER OBJECT
       ↓
Modify / Reset / Add / Delegate
       ↓
MORE PRIVILEGES
       ↓
LATERAL MOVEMENT / PRIVILEGE ESCALATION
```

> [!important]  
> ACL weaknesses may not look like traditional software vulnerabilities. The issue can simply be **an incorrect relationship between two legitimate AD objects**.

Large AD environments can accumulate these permissions through delegation, software installations, legacy configuration, or administrative convenience.

---

# 🕸️ 9. BloodHound + PowerView: Finding ACL Abuse

Two important tools in this workflow have different roles.

## 🩸 [BloodHound](https://bloodhound.specterops.io/)

**Primary role:** relationship and attack-path analysis.

Mental model:

```text
AD DATA
   ↓
Relationships
   ↓
Permissions
   ↓
Graph
   ↓
Attack Paths
```

BloodHound can represent relationships such as:

```text
User
 ↓
MemberOf
 ↓
Group
 ↓
AdminTo
 ↓
Computer
```

and ACL relationships such as:

```text
User
 ↓
GenericAll
 ↓
Target
```

The important point is that BloodHound helps answer:

> **"How does the permission I have connect me to something valuable?"**

---

## 🛠️ [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)

**Primary role:** AD enumeration and interaction from PowerShell.

PowerView provides PowerShell functionality for gaining situational awareness of Windows domains, including domain enumeration, user/group discovery, trust enumeration, user hunting, and ACL-related information.

Mental model:

```text
PowerView
    ↓
QUERY ACTIVE DIRECTORY
    ↓
Objects / Users / Groups / ACLs / Relationships
    ↓
Information
```

### 🧠 BloodHound vs PowerView

```text
PowerView
   ↓
ENUMERATE
   ↓
"Give me AD information"

BloodHound
   ↓
CORRELATE / VISUALIZE
   ↓
"Show me how this information connects"
```

They are therefore **complementary**, not simply competing tools.

---

# 🔓 10. Common Abusable AD Rights

The source identifies several permissions that can become useful attack primitives.

|Permission / ACE|Potential abuse|
|---|---|
|`ForceChangePassword`|`Set-DomainUserPassword`|
|`Add Members`|`Add-DomainGroupMember`|
|`GenericAll`|`Set-DomainUserPassword` / `Add-DomainGroupMember`|
|`GenericWrite`|`Set-DomainObject`|
|`WriteOwner`|`Set-DomainObjectOwner`|
|`WriteDACL`|`Add-DomainObjectACL`|
|`AllExtendedRights`|`Set-DomainUserPassword` / `Add-DomainGroupMember`|
|`AddSelf`|`Add-DomainGroupMember`|

### 🧠 The Important Pattern

Do not memorize these as random combinations.

Think:

```text
PERMISSION
    ↓
WHAT DOES IT LET ME MODIFY?
    ↓
WHAT SECURITY PRINCIPAL / OBJECT IS TARGETED?
    ↓
WHAT USEFUL STATE CAN I CREATE?
    ↓
HOW DOES THAT MOVE ME TOWARD MY OBJECTIVE?
```

---

# 🔐 11. ForceChangePassword

`ForceChangePassword` means a principal has the ability to **reset another user's password without knowing the user's current password**.

```text
YOUR ACCOUNT
     │
     │ ForceChangePassword
     ↓
TARGET USER
     │
     ↓
Reset password
     │
     ↓
Authenticate as target
```

The source associates this permission with:

```powershell
Set-DomainUserPassword
```

### ⚠️ Operational consideration

Changing a user's password is a **destructive modification**.

In a real penetration test:

- Confirm authorization.
    
- Consider client approval before resetting a password.
    
- Record the original state where possible.
    
- Document the change.
    
- Restore the environment when appropriate.
    

---

# 👥 12. Add Members / AddSelf

## `Add Members`

If a principal has permission to modify membership of a group, they may be able to add another security principal to that group.

```text
YOU
 │
 │ Add Members
 ↓
GROUP
 │
 ↓
Add controlled account
 │
 ↓
Controlled account inherits group's privileges
```

---

## `AddSelf`

`AddSelf` specifically represents the ability for a user to **add themselves to a security group**.

```text
YOUR USER
   │
   │ AddSelf
   ↓
TARGET GROUP
   │
   ↓
Add yourself
   │
   ↓
Receive group's permissions
```

This can be extremely important when the target group grants privileged access.

### 🧠 Difference

```text
Add Members
     ↓
Can manage membership

AddSelf
     ↓
Can add YOURSELF
```

---

# 🧨 13. GenericAll

`GenericAll` represents **full control over the target object**.

The exact practical abuse depends heavily on **what object GenericAll applies to**.

```text
                    GenericAll
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
        USER          GROUP        COMPUTER
          │             │             │
          ↓             ↓             ↓
   Modify user     Modify group    Computer-specific
      object       membership       attack paths
```

### 👤 GenericAll over a User

Potential actions include:

- Password manipulation
    
- Attribute modification
    
- Other operations permitted by full control
    

The source specifically highlights targeted Kerberoasting possibilities through attribute manipulation.

---

### 👥 GenericAll over a Group

Potentially:

```text
GenericAll
    ↓
GROUP
    ↓
Modify membership
    ↓
Add controlled principal
    ↓
Obtain group's permissions
```

---

### 💻 GenericAll over a Computer

Computer-object abuse can lead to different attack paths depending on the environment.

The source highlights **LAPS** as one possible example.

If LAPS is deployed and the required permissions exist:

```text
GenericAll over Computer
          ↓
Access relevant computer-object information
          ↓
LAPS password exposure
          ↓
Local Administrator access
          ↓
Lateral movement / privilege escalation
```

> [!warning]  
> Do not interpret `GenericAll` as meaning the same exact attack against every object. **The target object's type matters.**

---

# ✏️ 14. GenericWrite

`GenericWrite` gives the principal the ability to write to **non-protected attributes** of the target object.

Mental model:

```text
GenericWrite
     ↓
"I can modify attributes"
     ↓
What object?
     │
 ┌───┼──────────┐
 ↓   ↓          ↓
User Group   Computer
```

## 👤 GenericWrite over a User

The source highlights assigning an SPN to the user.

Conceptually:

```text
GenericWrite
     ↓
User attribute modification
     ↓
Assign SPN
     ↓
Request service ticket
     ↓
Extract ticket material
     ↓
Offline password cracking
     ↓
Potential account compromise
```

This creates a connection between **ACL abuse** and **Kerberoasting**.

> [!important]  
> GenericWrite itself is not "Kerberoasting." It is the **permission that enables an attribute modification** that can create a Kerberoasting opportunity.

---

## 👥 GenericWrite over a Group

Potentially allows modification that can result in group membership manipulation, depending on the exact rights and object configuration.

The source associates group abuse with:

```powershell
Add-DomainGroupMember
```

---

## 💻 GenericWrite over a Computer

The source notes that this can enable a **resource-based constrained delegation (RBCD)** attack.

That technique is outside the scope of this primer, but the CPTS mental model is:

```text
GenericWrite
     ↓
Computer object
     ↓
Modify relevant computer-object attributes
     ↓
RBCD opportunity
```

---

# 👑 15. WriteOwner

`WriteOwner` allows changing the **owner** of an object when the required permissions are available.

Source abuse function:

```powershell
Set-DomainObjectOwner
```

Mental model:

```text
Object
  │
  ↓
Current Owner
  │
  │ WriteOwner
  ↓
New Owner
  │
  ↓
Potentially gain additional control
```

> [!important]  
> `WriteOwner` is not the same thing as `GenericAll`.
> 
> It is a specific control relationship involving **ownership**. Ownership can then become useful because object ownership is tied to control over the object's security descriptor and permissions.

---

# 📝 16. WriteDACL

`WriteDACL` provides the ability to modify the object's **DACL**.

Source abuse function:

```powershell
Add-DomainObjectACL
```

Mental model:

```text
WriteDACL
    ↓
Modify DACL
    ↓
Grant yourself another right
    ↓
Obtain useful control
    ↓
Abuse the newly granted right
```

This makes `WriteDACL` particularly interesting because it can be used as a **permission-to-permission escalation**.

Example concept:

```text
YOU
 │
 │ WriteDACL
 ↓
TARGET
 │
 ↓
Modify DACL
 │
 ↓
Grant yourself useful ACE
 │
 ↓
New permission
 │
 ↓
Abuse new permission
```

---

# 🔑 17. AllExtendedRights

`AllExtendedRights` represents broad extended-rights access over the target.

The source associates this with potential abuse through:

```powershell
Set-DomainUserPassword
Add-DomainGroupMember
```

The important CPTS lesson is:

> **Do not treat every permission name as self-explanatory.**

When BloodHound exposes an unfamiliar edge:

```text
Unknown / unfamiliar edge
          ↓
Identify the exact right
          ↓
Determine target object type
          ↓
Research what operation the right permits
          ↓
Find a valid abuse primitive
          ↓
Test safely in the lab
```

---

# 🧬 18. Other ACL Rights You May Encounter

The source specifically warns that the four major rights covered in this primer are **not the complete universe of AD ACL abuse**.

You may encounter other BloodHound edges or Active Directory extended rights.

Examples include:

- `ReadGMSAPassword`
    
- `Unexpire-Password`
    
- `Reanimate-Tombstones`
    

### 🔑 ReadGMSAPassword

A user may have a relationship allowing them to read the password of a **Group Managed Service Account (gMSA)**.

Conceptually:

```text
Controlled User
      │
      │ ReadGMSAPassword
      ↓
    gMSA
      │
      ↓
Password material
      │
      ↓
Service-account access
```

The source mentions [GMSAPasswordReader](https://github.com/rvazarkar/GMSAPasswordReader) as one tool that can be used when appropriate access exists.

---

# 🧠 19. Unknown ACE → Research → Abuse

This is one of the most important CPTS skills from this section.

You will eventually encounter a permission you have not memorized.

Do **not** stop at:

> "I don't know what this edge means."

Instead:

```text
BloodHound shows unfamiliar edge
             ↓
        Identify edge
             ↓
   Determine target object
             ↓
   Understand granted right
             ↓
    Research the operation
             ↓
   Find an abuse primitive
             ↓
     Validate in context
             ↓
        Execute safely
             ↓
      Document / revert
```

Useful references include the [BloodHound edge documentation](https://bloodhound.specterops.io/resources/edges/overview) and Microsoft's documentation for [Active Directory extended rights](https://learn.microsoft.com/en-us/windows/win32/adschema/extended-rights).

> [!tip]  
> **CPTS exam mindset:** You do not need to memorize every obscure ACE. You need to understand the methodology for determining **what a permission grants and how that permission could change your attack path**.

---

# 🧭 20. ACL Abuse in the CPTS Methodology

ACL abuse fits naturally into the broader AD attack workflow.

```text
                    INITIAL ACCESS
                          ↓
                 Obtain AD credentials
                          ↓
                    ENUMERATION
                          ↓
             Users / Groups / Computers
                          ↓
                    ACL ENUMERATION
                          ↓
              Identify excessive rights
                          ↓
                  BLOODHOUND GRAPH
                          ↓
                 Find useful relationships
                          ↓
              ┌───────────┼────────────┐
              ↓           ↓            ↓
        Password Reset   Group     Object Control
                          │
                          ↓
                    New Privilege
                          │
                          ↓
                Lateral Movement /
                Privilege Escalation
                          │
                          ↓
                    Re-enumerate
                          │
                          ↓
                    Repeat / Pivot
```

### 🧠 The Key Principle

After obtaining new privileges:

```text
NEW ACCESS
    ↓
RE-ENUMERATE
    ↓
NEW ACL RELATIONSHIPS?
    ↓
NEW USERS / GROUPS / COMPUTERS?
    ↓
NEW ATTACK PATH?
```

ACL abuse should therefore be viewed as a **recursive attack-path problem**, not a one-time enumeration step.

---

# 🎯 21. Common Real-World ACL Attack Scenarios

The source identifies three particularly common scenarios.

|Attack|Description|
|---|---|
|`Abusing forgot password permissions`|Help Desk and other IT users are often granted permissions to perform password resets and other privileged tasks. If we can take over an account with these privileges (or an account in a group that confers these privileges on its users), we may be able to perform a password reset for a more privileged account in the domain.|
|`Abusing group membership management`|It's also common to see Help Desk and other staff that have the right to add/remove users from a given group. It is always worth enumerating this further, as sometimes we may be able to add an account that we control into a privileged built-in AD group or a group that grants us some sort of interesting privilege.|
|`Excessive user rights`|We also commonly see user, computer, and group objects with excessive rights that a client is likely unaware of. This could occur after some sort of software install (Exchange, for example, adds many ACL changes into the environment at install time) or some kind of legacy or accidental configuration that gives a user unintended rights. Sometimes we may take over an account that was given certain rights out of convenience or to solve a nagging problem more quickly.|

---

# 🚨 22. Why ACL Misconfigurations Are Dangerous

An ACL misconfiguration can effectively transform:

```text
LOW PRIVILEGE ACCOUNT
        │
        │ Unexpected ACE
        ↓
HIGHER-VALUE OBJECT
        │
        ↓
CONTROL / MODIFICATION
        │
        ↓
NEW PRIVILEGE
        │
        ↓
DOMAIN-WIDE CONSEQUENCES
```

The danger comes from **chained permissions**.

For example:

```text
User A
  │
  │ GenericWrite
  ↓
User B
  │
  │ MemberOf
  ↓
Privileged Group
  │
  ↓
Administrative Access
```

The first account does not need to be privileged itself.

It only needs a permission relationship that eventually leads to something privileged.

---

# 🧹 23. Operational Safety & Cleanup

ACL attacks can modify the environment.

Examples include:

- Resetting passwords
    
- Adding group members
    
- Changing ownership
    
- Modifying DACLs
    
- Modifying object attributes
    

These changes can be **destructive** or disruptive.

> [!warning]  
> In an authorized penetration test, do not treat "technically possible" as equivalent to "safe to execute."

### Recommended assessment workflow

```text
IDENTIFY
   ↓
UNDERSTAND IMPACT
   ↓
CONFIRM AUTHORIZATION
   ↓
EXECUTE
   ↓
DOCUMENT CHANGE
   ↓
VERIFY RESULT
   ↓
REVERT CHANGE
   ↓
VERIFY CLEANUP
   ↓
REPORT
```

The source specifically recommends obtaining client approval when in doubt, documenting attacks from start to finish, and reverting modifications so the client can verify that the environment was restored.

---

# 🧠 24. ACL Abuse Mental Model

The entire section can be reduced to one question:

> **"What can the account I control do to another AD object?"**

Then:

```text
                 CURRENT ACCOUNT
                        │
                        ↓
                WHAT CAN I CONTROL?
                        │
                        ↓
                 ACL / ACE ENUM
                        │
                        ↓
              ┌─────────┴─────────┐
              │                   │
           TARGET USER          GROUP
              │                   │
              ↓                   ↓
       Password / SPN        Membership
              │                   │
              └─────────┬─────────┘
                        ↓
                  NEW ACCESS
                        │
                        ↓
                  RE-ENUMERATE
                        │
                        ↓
                   NEW PATH?
                        │
                        ↓
                  ATTACK / PIVOT
```

---

# 🧩 25. Permission → Object → Abuse Cheat Sheet

|Permission|Target|What to think about|
|---|---|---|
|`ForceChangePassword`|User|Can I reset this user's password?|
|`AddSelf`|Group|Can I add myself to this group?|
|`Add Members`|Group|Can I add a controlled principal to this group?|
|`GenericAll`|User|What full-control operations can I perform against this user?|
|`GenericAll`|Group|Can I manipulate membership?|
|`GenericAll`|Computer|What computer-specific control paths are available?|
|`GenericWrite`|User|Can I modify an attribute to create another attack opportunity?|
|`GenericWrite`|Group|Can I manipulate useful group attributes/membership?|
|`GenericWrite`|Computer|Could this enable RBCD?|
|`WriteOwner`|Object|Can I change ownership and then leverage ownership-based control?|
|`WriteDACL`|Object|Can I modify the DACL to grant myself a more useful right?|
|`AllExtendedRights`|User/Group|Which extended operations does this relationship permit?|
|`ReadGMSAPassword`|gMSA|Can I obtain the gMSA's password material?|

---

# 🔄 26. The ACL Abuse Decision Tree

```text
                I CONTROL USER X
                       │
                       ↓
             Enumerate ACL relationships
                       │
                       ↓
              Does X control Y?
                 /           \
               NO             YES
               │               │
               ↓               ↓
         Continue AD       Identify right
         enumeration            │
                                ↓
                        What is target type?
                         /      |       \
                      USER    GROUP   COMPUTER
                        │       │        │
                        ↓       ↓        ↓
                    Password   Member   Computer
                    / SPN      control  specific
                        │       │        │
                        └───────┴────────┘
                                ↓
                        Determine abuse path
                                │
                                ↓
                       Is modification required?
                          /              \
                        YES              NO
                         │                │
                   Check authorization    │
                         │                │
                         ↓                ↓
                     Execute          Access
                         │
                         ↓
                    Re-enumerate
                         │
                         ↓
                  New attack path?
```

---

# 🏁 27. Key Takeaways

> [!success]  
> **ACL = collection of ACEs.**
> 
> **ACE = individual permission/audit rule.**
> 
> **DACL = controls access.**
> 
> **SACL = audits access.**

> [!important]  
> The most important CPTS mental model is:
> 
> **Current principal → ACE → target object → granted capability → abuse primitive → new access → re-enumerate.**

### Remember the major relationships

```text
ForceChangePassword
        ↓
Password reset

AddSelf / Add Members
        ↓
Group membership

GenericAll
        ↓
Full control over target object

GenericWrite
        ↓
Modify writable attributes

WriteOwner
        ↓
Change ownership

WriteDACL
        ↓
Modify permissions

AllExtendedRights
        ↓
Broad extended-right capability

ReadGMSAPassword
        ↓
Potential gMSA credential access
```

### Final CPTS mental model

```text
🔎 ENUMERATE
     ↓
🧩 FIND ACL / ACE
     ↓
🎯 IDENTIFY TARGET OBJECT
     ↓
🧠 UNDERSTAND THE RIGHT
     ↓
🔓 DETERMINE WHAT CONTROL IT PROVIDES
     ↓
🛠️ MAP TO AN ABUSE TECHNIQUE
     ↓
📈 GAIN NEW ACCESS
     ↓
🔄 RE-ENUMERATE
     ↓
🎯 CONTINUE THE ATTACK PATH
```

> [!tip]  
> **Do not memorize ACL edges as isolated names.**
> 
> Learn to translate:
> 
> **"I have this permission over this type of object"**
> 
> into:
> 
> **"Therefore, these are the operations I can potentially perform, and this is how they could change my attack path."**