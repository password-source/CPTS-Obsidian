# 🔐 1. Enumerating & Retrieving Password Policies

> [!info]  
> **Objective:** Learn how to retrieve an Active Directory domain's password policy from **Linux and Windows**, both **with and without credentials**, using SMB/RPC, LDAP, built-in Windows commands, and PowerView.
> 
> The password policy is a critical piece of information before attempting **password spraying**, because it tells us how many authentication failures may be tolerated before accounts are locked.

---

# 🧾 2. Command Cheat Sheet

## 🐧 Linux — Credentialed

### CrackMapExec

```bash
crackmapexec smb 172.16.5.5 -u avazquez -p Password123 --pass-pol
```

Retrieve the domain password policy using valid SMB credentials.

> [!warning]  
> The original `CrackMapExec` project referenced by the source was archived in December 2023. For modern environments, treat it as legacy tooling and be aware of maintained alternatives.

---

## 🐧 Linux — SMB NULL Session

### rpcclient

```bash
rpcclient -U "" -N 172.16.5.5
```

Attempt an unauthenticated SMB/RPC connection.

```text
querydominfo
```

Retrieve domain information.

```text
getdompwinfo
```

Retrieve password-policy information.

`rpcclient` is part of the Samba suite and provides client-side MS-RPC functionality.

---

## 🔎 enum4linux

```bash
enum4linux -P 172.16.5.5
```

Enumerate password-policy information.

`-P` specifically requests password policy information.

---

## 🔎 enum4linux-ng

```bash
enum4linux-ng -P 172.16.5.5 -oA ilfreight
```

Enumerate the password policy and save results in multiple output formats.

```bash
cat ilfreight.json
```

Review the generated JSON.

> [!tip]  
> `-oA` is particularly useful because the structured JSON/YAML output can later be processed by scripts or other tooling. enum4linux-ng supports JSON/YAML export natively.

---

## 🪟 Windows — NULL Session

```cmd
net use \\DC01\ipc$ "" /u:""
```

Attempt to establish an unauthenticated IPC$ session.

---

## 🪟 Windows — Built-in Password Policy

```cmd
net accounts
```

Retrieve the local/domain password policy using a built-in Windows command.

---

## 🪟 Windows — PowerView

```powershell
import-module .\PowerView.ps1
```

Load PowerView.

```powershell
Get-DomainPolicy
```

Retrieve the domain policy.

---

## 🐧 Linux — LDAP Anonymous Bind

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

Query LDAP anonymously and extract the relevant password-policy attributes.

> [!warning]  
> Newer versions of `ldapsearch` deprecate `-h` in favor of `-H`.

---

# 🧠 3. Why Enumerate the Password Policy First?

Before password spraying, we need to answer:

```text
❓ How many bad passwords can we try?
❓ How long does an account remain locked?
❓ Does the account automatically unlock?
❓ What password length is required?
❓ Is complexity enabled?
❓ How frequently can we safely spray?
```

The password policy determines the **risk boundary** of password attacks.

### 🧠 Mental Model

```text
🔎 Obtain Password Policy
          ↓
📏 Minimum Password Length
          ↓
🔐 Complexity Requirements
          ↓
🚨 Lockout Threshold
          ↓
⏱️ Lockout Duration
          ↓
🔄 Reset/Observation Window
          ↓
🎯 Determine Safe Spraying Strategy
          ↓
👤 Build Target User List
          ↓
🔑 Password Spray
```

> [!important]  
> **Never start spraying blindly when you can safely obtain the policy first.**

---

# 🐧 4. Enumerating the Password Policy from Linux — Credentialed

If valid domain credentials are available, the easiest approach is often to query the Domain Controller remotely over SMB.

## 4.1 Using CrackMapExec

```bash
crackmapexec smb 172.16.5.5 -u avazquez -p Password123 --pass-pol
```

Example:

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\avazquez:Password123 
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] Dumping password info for domain: INLANEFREIGHT
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Minimum password length: 8
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Password history length: 24
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Maximum password age: Not Set
SMB         172.16.5.5      445    ACADEMY-EA-DC01  
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Password Complexity Flags: 000001
SMB         172.16.5.5      445    ACADEMY-EA-DC01      Domain Refuse Password Change: 0
SMB         172.16.5.5      445    ACADEMY-EA-DC01      Domain Password Store Cleartext: 0
SMB         172.16.5.5      445    ACADEMY-EA-DC01      Domain Password Lockout Admins: 0
SMB         172.16.5.5      445    ACADEMY-EA-DC01      Domain Password No Clear Change: 0
SMB         172.16.5.5      445    ACADEMY-EA-DC01      Domain Password No Anon Change: 0
SMB         172.16.5.5      445    ACADEMY-EA-DC01      Domain Password Complex: 1
SMB         172.16.5.5      445    ACADEMY-EA-DC01  
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Minimum password age: 1 day 4 minutes 
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Reset Account Lockout Counter: 30 minutes 
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Locked Account Duration: 30 minutes 
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Account Lockout Threshold: 5
SMB         172.16.5.5      445    ACADEMY-EA-DC01  Forced Log off Time: Not Set
```

---

## 🔎 4.2 Interpreting the Policy

The important values are:

|Policy|Value|Why It Matters|
|---|--:|---|
|Minimum password length|8|Shorter passwords may be easier to guess/crack|
|Password history|24|Users cannot immediately reuse recent passwords|
|Maximum password age|Not Set|Passwords do not have a configured expiration|
|Password complexity|Enabled|Passwords must satisfy complexity requirements|
|Minimum password age|1 day 4 minutes|Users cannot immediately change passwords repeatedly|
|Reset account lockout counter|30 minutes|Failed-attempt counter resets after this period|
|Locked account duration|30 minutes|Locked accounts remain locked for this period|
|Account lockout threshold|5|Five failed attempts can trigger lockout|
|Forced logoff|Not Set|No configured forced logoff|

> [!important]  
> The **lockout threshold**, **lockout duration**, and **reset counter** are the most important values when determining how cautiously authentication testing must be performed.

---

# 🐧 5. SMB NULL Sessions

A different situation occurs when we have **no credentials**.

A misconfigured Domain Controller may allow an **SMB NULL session**.

## 🧠 What Is a NULL Session?

A NULL session is an unauthenticated SMB/RPC session.

Instead of:

```text
Username + Password
        ↓
Authentication
        ↓
SMB/RPC access
```

we have:

```text
Username: ""
Password: ""
        ↓
Unauthenticated session
        ↓
Potential SMB/RPC enumeration
```

Historically, older Windows Server configurations allowed anonymous access to certain resources. When those systems were upgraded in place, insecure legacy configurations could sometimes remain.

---

# 🎯 6. What Can a NULL Session Reveal?

Depending on the configuration, anonymous SMB/RPC access can expose information such as:

- Users
    
- Groups
    
- Computers
    
- User account attributes
    
- Domain information
    
- Password policy
    
- Other domain enumeration data
    

> [!warning]  
> A NULL session is **not guaranteed** to provide all of this information. Modern configurations commonly restrict anonymous access.

---

# 🧰 7. NULL Session Enumeration Tools

Common tools include:

- [rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html)
    
- [enum4linux](https://labs.portcullis.co.uk/tools/enum4linux/)
    
- [enum4linux-ng](https://github.com/cddmp/enum4linux-ng)
    
- [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec)
    

The original `enum4linux` is essentially a wrapper around Samba tools such as `smbclient`, `rpcclient`, `net`, and `nmblookup`.

---

# 🔌 8. rpcclient — Testing for an SMB NULL Session

Use:

```bash
rpcclient -U "" -N 172.16.5.5
```

### Important options

```text
-U ""
```

Supply an empty username.

```text
-N
```

Do not request a password.

The result is an attempt to establish an unauthenticated RPC session.

---

# 🔎 9. querydominfo — Confirming Domain Access

Once connected:

```text
rpcclient $> querydominfo
```

Output:

```text
Domain:     INLANEFREIGHT
Server:     
Comment:    
Total Users:    3650
Total Groups:   0
Total Aliases:  37
Sequence No:    1
Force Logoff:   -1
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:  0x1
```

### 🧠 What Does This Tell Us?

The key result is that an unauthenticated session was able to retrieve domain-level information.

We learn:

```text
Domain:       INLANEFREIGHT
Users:        3650
Aliases:      37
Server Role:  ROLE_DOMAIN_PDC
```

This confirms that the NULL session has useful enumeration capabilities.

---

# 🔐 10. getdompwinfo — Retrieve the Password Policy

Inside `rpcclient`:

```text
rpcclient $> getdompwinfo
```

Output:

```text
min_password_length: 8
password_properties: 0x00000001
    DOMAIN_PASSWORD_COMPLEX
```

### Interpretation

```text
Minimum password length = 8
Password complexity     = Enabled
```

The important point is that **we obtained this without valid credentials**.

---

# 🔎 11. enum4linux — Password Policy Enumeration

[enum4linux](https://labs.portcullis.co.uk/tools/enum4linux/) provides a convenient wrapper around several Samba utilities and supports password-policy enumeration with `-P`.

Run:

```bash
enum4linux -P 172.16.5.5
```

Output:

```text
 ================================================== 
|    Password Policy Information for 172.16.5.5    |
 ================================================== 

[+] Attaching to 172.16.5.5 using a NULL share
[+] Trying protocol 139/SMB...

    [!] Protocol failed: Cannot request session (Called Name:172.16.5.5)

[+] Trying protocol 445/SMB...
[+] Found domain(s):

    [+] INLANEFREIGHT
    [+] Builtin

[+] Password Info for Domain: INLANEFREIGHT

    [+] Minimum password length: 8
    [+] Password history length: 24
    [+] Maximum password age: Not Set
    [+] Password Complexity Flags: 000001

        [+] Domain Refuse Password Change: 0
        [+] Domain Password Store Cleartext: 0
        [+] Domain Password Lockout Admins: 0
        [+] Domain Password No Clear Change: 0
        [+] Domain Password No Anon Change: 0
        [+] Domain Password Complex: 1

    [+] Minimum password age: 1 day 4 minutes 
    [+] Reset Account Lockout Counter: 30 minutes 
    [+] Locked Account Duration: 30 minutes 
    [+] Account Lockout Threshold: 5
    [+] Forced Log off Time: Not Set

[+] Retieved partial password policy with rpcclient:

Password Complexity: Enabled
Minimum Password Length: 8

enum4linux complete on Tue Feb 22 17:39:29 2022
```

---

# 🧠 12. Understanding the enum4linux Output

Notice:

```text
[+] Trying protocol 139/SMB...
[!] Protocol failed
```

followed by:

```text
[+] Trying protocol 445/SMB...
```

and successful enumeration.

### Mental Model

```text
enum4linux
    │
    ├── Try SMB/NetBIOS
    │       │
    │       └── 139/TCP → failed
    │
    └── Try SMB
            │
            └── 445/TCP → successful
                    │
                    ▼
              NULL session
                    │
                    ▼
              RPC enumeration
                    │
                    ▼
             Password policy
```

> [!important]  
> A failed connection on one SMB transport does **not** mean SMB enumeration is impossible. Here, port `139/TCP` failed while `445/TCP` succeeded.

---

# 📊 13. Common Samba Enumeration Tools & Ports

|ToolPorts||
|---|---|
|nmblookup|137/UDP|
|nbtstat|137/UDP|
|net|139/TCP, 135/TCP, TCP and UDP 135 and 49152-65535|
|rpcclient|135/TCP|
|smbclient|445/TCP|

> [!warning]  
> Preserve this table as a source reference, but remember that modern SMB/RPC deployments can involve additional transport behavior and dynamic RPC ports. For example, current Samba documentation describes `rpcclient` transports over SMB named pipes and direct DCERPC/TCP bindings.

---

# 🚀 14. enum4linux-ng

[enum4linux-ng](https://github.com/cddmp/enum4linux-ng) is a Python rewrite of the original enum4linux.

It provides additional functionality such as:

- YAML output
    
- JSON output
    
- Colored output
    
- Multiple authentication methods
    
- SMBv1 support
    
- IPC signing detection
    
- Smart enumeration
    
- Timeout support
    
- SMB dialect checks
    
- Experimental IPv6 support
    
- Native LDAP-related enumeration functionality
    

---

# 🔎 15. Using enum4linux-ng

```bash
enum4linux-ng -P 172.16.5.5 -oA ilfreight
```

The important pieces are:

```text
-P
```

Password-policy enumeration.

```text
-oA ilfreight
```

Save the enumeration results in the supported structured output formats using the specified base name.

---

# 🧪 16. enum4linux-ng NULL Session Detection

Output:

```text
 =======================================
|    RPC Session Check on 172.16.5.5    |
 =======================================
[*] Check for null session
[+] Server allows session using username '', password ''
[*] Check for random user session
[-] Could not establish random user session: STATUS_LOGON_FAILURE
```

This is an important distinction.

```text
NULL credentials
      ↓
Accepted
      ↓
Anonymous session possible
```

while:

```text
Random username
      ↓
STATUS_LOGON_FAILURE
      ↓
Not simply accepting arbitrary credentials
```

---

# 🌐 17. Domain Information from enum4linux-ng

```text
 =================================================
|    Domain Information via RPC for 172.16.5.5    |
 =================================================
[+] Domain: INLANEFREIGHT
[+] SID: S-1-5-21-3842939050-3880317879-2865463114
[+] Host is part of a domain (not a workgroup)
```

SMB information:

```text
 =========================================================
|    Domain Information via SMB session for 172.16.5.5    |
 =========================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: ACADEMY-EA-DC01
NetBIOS domain name: INLANEFREIGHT
DNS domain: INLANEFREIGHT.LOCAL
FQDN: ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

### 🧠 Information We Now Have

```text
Host:
ACADEMY-EA-DC01

NetBIOS Domain:
INLANEFREIGHT

DNS Domain:
INLANEFREIGHT.LOCAL

FQDN:
ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL

Domain SID:
S-1-5-21-3842939050-3880317879-2865463114
```

---

# 🔐 18. Password Policy from enum4linux-ng

```text
 =======================================
|    Policies via RPC for 172.16.5.5    |
 =======================================
[*] Trying port 445/tcp
[+] Found policy:
domain_password_information:
  pw_history_length: 24
  min_pw_length: 8
  min_pw_age: 1 day 4 minutes
  max_pw_age: not set
  pw_properties:
  - DOMAIN_PASSWORD_COMPLEX: true
  - DOMAIN_PASSWORD_NO_ANON_CHANGE: false
  - DOMAIN_PASSWORD_NO_CLEAR_CHANGE: false
  - DOMAIN_PASSWORD_LOCKOUT_ADMINS: false
  - DOMAIN_PASSWORD_PASSWORD_STORE_CLEARTEXT: false
  - DOMAIN_PASSWORD_REFUSE_PASSWORD_CHANGE: false
domain_lockout_information:
  lockout_observation_window: 30 minutes
  lockout_duration: 30 minutes
  lockout_threshold: 5
domain_logoff_information:
  force_logoff_time: not set
```

This structured format is much easier to consume programmatically than raw terminal output.

---

# 📦 19. Saving Structured Enumeration Results

The source uses:

```bash
enum4linux-ng -P 172.16.5.5 -oA ilfreight
```

Then:

```bash
cat ilfreight.json
```

Example:

```json
{
    "target": {
        "host": "172.16.5.5",
        "workgroup": ""
    },
    "credentials": {
        "user": "",
        "password": "",
        "random_user": "yxditqpc"
    },
    "services": {
        "SMB": {
            "port": 445,
            "accessible": true
        },
        "SMB over NetBIOS": {
            "port": 139,
            "accessible": true
        }
    },
    "smb_dialects": {
        "SMB 1.0": false,
        "SMB 2.02": true,
        "SMB 2.1": true,
        "SMB 3.0": true,
        "SMB1 only": false,
        "Preferred dialect": "SMB 3.0",
        "SMB signing required": true
    },
    "sessions_possible": true,
    "null_session_possible": true,

<SNIP>
```

### 🧠 Why Structured Output Matters

Instead of manually copying information from terminal output:

```text
Terminal
   ↓
JSON/YAML
   ↓
Script processing
   ↓
Other enumeration tools
   ↓
Attack graph / notes / reporting
```

This becomes especially useful during larger assessments.

---

# 🪟 20. Enumerating a NULL Session from Windows

Performing this type of enumeration from Windows is less common, but it is possible.

Use:

```cmd
net use \\DC01\ipc$ "" /u:""
```

Example:

```text
The command completed successfully.
```

### 🧠 What Happened?

```text
Windows host
    ↓
net use
    ↓
\\DC01\ipc$
    ↓
Empty username/password
    ↓
IPC$ connection
    ↓
NULL session established
```

This is particularly useful when your foothold is Windows and you do not want or cannot transfer Linux tools.

---

# 🧪 21. Windows Authentication Errors

The `net use` command can also reveal useful information from authentication failures.

---

## ❌ 21.1 Account Disabled

```cmd
net use \\DC01\ipc$ "" /u:guest
```

Output:

```text
System error 1331 has occurred.

This user can't sign in because this account is currently disabled.
```

This tells us that the account exists but is disabled.

---

## ❌ 21.2 Incorrect Password

```cmd
net use \\DC01\ipc$ "password" /u:guest
```

Output:

```text
System error 1326 has occurred.

The user name or password is incorrect.
```

This indicates invalid authentication credentials.

---

## 🔒 21.3 Account Locked Out

```cmd
net use \\DC01\ipc$ "password" /u:guest
```

Output:

```text
System error 1909 has occurred.

The referenced account is currently locked out and may not be logged on to.
```

This confirms that the account is currently locked.

> [!important]  
> These error messages are useful during enumeration because authentication failures can reveal **account state**, not simply "wrong password."

---

# 🐧 22. LDAP Anonymous Bind

Another unauthenticated enumeration path is **LDAP anonymous bind**.

LDAP anonymous binds can expose information such as:

- Users
    
- Groups
    
- Computers
    
- User attributes
    
- Domain information
    
- Password policy
    

Historically, anonymous LDAP access was more permissive, and legacy configurations may still exist.

> [!warning]  
> Do not assume anonymous LDAP access is available simply because the environment uses Active Directory. Modern Windows Server configurations generally restrict unauthenticated LDAP operations, but legacy or application-specific configurations can reintroduce anonymous access.

---

# 🔎 23. LDAP Enumeration Tools

Common tools include:

- [ldapsearch](https://www.openldap.org/)
    
- `windapsearch.py`
    
- `ad-ldapdomaindump.py`
    

The key concept is:

```text
No credentials
      ↓
LDAP anonymous bind
      ↓
LDAP directory query
      ↓
Domain objects / policy
```

---

# 🔍 24. ldapsearch

The source uses:

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

### Important options

```text
-h 172.16.5.5
```

LDAP server address.

> Newer `ldapsearch` versions deprecate `-h` in favor of `-H`.

```text
-x
```

Use simple authentication rather than SASL.

```text
-b "DC=INLANEFREIGHT,DC=LOCAL"
```

Set the LDAP search base.

```text
-s sub
```

Search the base and its subtree.

```text
"*"
```

Request the available attributes.

```text
grep -m 1 -B 10 pwdHistoryLength
```

Find the relevant password-history attribute while including surrounding lines.

---

# 📋 25. LDAP Password Policy Output

```text
forceLogoff: -9223372036854775808
lockoutDuration: -18000000000
lockOutObservationWindow: -18000000000
lockoutThreshold: 5
maxPwdAge: -9223372036854775808
minPwdAge: -864000000000
minPwdLength: 8
modifiedCountAtLastProm: 0
nextRid: 1002
pwdProperties: 1
pwdHistoryLength: 24
```

### 🔎 Important Values

```text
minPwdLength: 8
```

Minimum password length is 8.

```text
lockoutThreshold: 5
```

Five failed attempts trigger lockout.

```text
pwdProperties: 1
```

Password complexity is enabled.

```text
pwdHistoryLength: 24
```

Twenty-four previous passwords are retained in password history.

---

# 🪟 26. Enumerating the Password Policy from Windows

If we can authenticate to the domain from a Windows host, we do not necessarily need to transfer additional tooling.

Windows includes built-in commands capable of retrieving useful policy information.

This is important when:

- You land on a Windows system.
    
- You have domain authentication.
    
- Tool transfer is restricted.
    
- The client supplied the Windows attack system.
    
- AV/EDR makes third-party tooling undesirable.
    
- You want to minimize your tooling footprint.
    

---

# 🧰 27. net accounts

Use:

```cmd
net accounts
```

Output:

```text
Force user logoff how long after time expires?:       Never
Minimum password age (days):                          1
Maximum password age (days):                          Unlimited
Minimum password length:                              8
Length of password history maintained:                24
Lockout threshold:                                    5
Lockout duration (minutes):                           30
Lockout observation window (minutes):                 30
Computer role:                                        SERVER
The command completed successfully.
```

---

# 🧠 28. Interpreting net accounts

The output gives us:

### 🔐 Password Length

```text
Minimum password length: 8
```

An 8-character minimum is relatively weak by modern standards and leaves more candidate passwords available for password spraying.

### ⏳ Password Expiration

```text
Maximum password age: Unlimited
```

Passwords do not expire according to this policy.

### 🚨 Lockout Threshold

```text
Lockout threshold: 5
```

Five failed authentication attempts can trigger lockout.

### ⏱️ Lockout Duration

```text
Lockout duration: 30
```

A locked account remains locked for 30 minutes in this policy.

### 🔄 Observation Window

```text
Lockout observation window: 30
```

The failed-attempt counter resets after 30 minutes.

---

# 🧨 29. Why This Policy Is Interesting for Password Spraying

The source identifies this policy as favorable for password spraying.

Policy:

```text
Minimum password length:     8
Lockout threshold:           5
Lockout duration:           30 min
Observation window:          30 min
Maximum password age:        Unlimited
```

This tells us that weak but policy-compliant passwords may exist, for example:

```text
Welcome1
Password1
```

> [!warning]  
> Password complexity does **not** necessarily mean a password is strong.
> 
> A password such as `Welcome1` can satisfy common complexity rules while remaining highly predictable.

---

# 🧠 30. Password Complexity Does Not Equal Strong Passwords

The source explains that complexity generally requires a password to contain **3 of 4** character categories:

```text
1. Uppercase
2. Lowercase
3. Number
4. Special character
```

For example:

```text
Password1
Welcome1
```

can satisfy complexity requirements while remaining poor passwords.

### Mental Model

```text
Complexity Requirement
        ≠
Strong Password
```

A policy can enforce:

```text
Upper + Lower + Number
```

while users still choose predictable patterns.

---

# 🐚 31. PowerView

[PowerView](https://github.com/PowerShellMafia/PowerSploit) is another option when operating from Windows.

Load it:

```powershell
import-module .\PowerView.ps1
```

Then:

```powershell
Get-DomainPolicy
```

Example:

```text
Unicode        : @{Unicode=yes}
SystemAccess   : @{MinimumPasswordAge=1; MaximumPasswordAge=-1; MinimumPasswordLength=8; PasswordComplexity=1;
                 PasswordHistorySize=24; LockoutBadCount=5; ResetLockoutCount=30; LockoutDuration=30;
                 RequireLogonToChangePassword=0; ForceLogoffWhenHourExpire=0; ClearTextPassword=0;
                 LSAAnonymousNameLookup=0}
KerberosPolicy : @{MaxTicketAge=10; MaxRenewAge=7; MaxServiceAge=600; MaxClockSkew=5; TicketValidateClient=1}
Version        : @{signature="$CHICAGO$"; Revision=1}
RegistryValues : @{MACHINE\System\CurrentControlSet\Control\Lsa\NoLMHash=System.Object[]}
Path           : \\INLANEFREIGHT.LOCAL\sysvol\INLANEFREIGHT.LOCAL\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHI
                 NE\Microsoft\Windows NT\SecEdit\GptTmpl.inf
GPOName         : {31B2F340-016D-11D2-945F-00C04FB984F9}
GPODisplayName : Default Domain Policy
```

---

# 🔎 32. Understanding PowerView Output

The important values are:

```text
MinimumPasswordLength=8
```

Minimum length = 8.

```text
PasswordComplexity=1
```

Complexity enabled.

```text
PasswordHistorySize=24
```

24 previous passwords remembered.

```text
LockoutBadCount=5
```

Lockout threshold = 5.

```text
ResetLockoutCount=30
```

Failed-attempt counter reset = 30 minutes.

```text
LockoutDuration=30
```

Lockout duration = 30 minutes.

```text
MaximumPasswordAge=-1
```

Password does not expire.

---

# 🧩 33. Why Multiple Enumeration Methods Matter

You may encounter the same information through multiple paths:

```text
                    PASSWORD POLICY
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
     Linux             Windows           LDAP
        │                 │                 │
        ▼                 ▼                 ▼
  CrackMapExec       net accounts     ldapsearch
  rpcclient           PowerView
  enum4linux
  enum4linux-ng
```

The goal is not to memorize one command.

The goal is to recognize:

> **"I need the domain password policy. What access do I currently have, and which available interface can expose it?"**

---

# 🧭 34. Methodology Decision Tree

```text
             🔎 Need Password Policy
                       │
          ┌────────────┴────────────┐
          │                         │
     Have credentials?          No credentials
          │                         │
       ┌──┴──┐                ┌─────┴─────┐
       │     │                │           │
      YES    NO               SMB         LDAP
       │                      NULL        Anonymous
       ▼                        │           │
    SMB/RPC                     ▼           ▼
       │                    rpcclient   ldapsearch
       │                    enum4linux
       │                    enum4linux-ng
       ▼
   CrackMapExec
   rpcclient
       │
       └──────────────┬──────────────┘
                      ▼
             Analyze Policy
                      │
                      ▼
             Assess Spray Risk
                      │
                      ▼
             Build User List
```

---

# 📊 35. Linux vs Windows Enumeration

|Situation|Recommended Approach|
|---|---|
|Linux + valid credentials|CrackMapExec / `rpcclient`|
|Linux + no credentials|NULL session / anonymous LDAP|
|Linux + SMB NULL session|`rpcclient`, `enum4linux`, `enum4linux-ng`|
|Linux + LDAP anonymous bind|`ldapsearch` / LDAP enumeration tools|
|Windows + no additional tools|`net use`, `net accounts`|
|Windows + PowerShell tooling available|PowerView|
|Windows + restricted tool transfer|Built-in Windows commands|
|Need structured output|enum4linux-ng|

---

# 📊 36. Password Policy — Source Environment

|Policy|Value|CPTS Relevance|
|---|--:|---|
|Minimum password length|8|Weak passwords may still be policy-compliant|
|Password history|24|Prevents recent password reuse|
|Maximum password age|Not Set / Unlimited|Passwords do not expire|
|Password complexity|Enabled|Does not guarantee strong passwords|
|Minimum password age|1 day 4 minutes|Limits rapid password changes|
|Lockout threshold|5|Critical spraying constraint|
|Lockout duration|30 minutes|Determines how long accounts remain locked|
|Reset lockout counter|30 minutes|Determines when failed attempts reset|
|Forced logoff|Not Set|No forced logoff configured|

---

# 📜 37. Default Active Directory Password Policy

The source provides the following default policy for a newly created domain:

|PolicyDefault Value||
|---|---|
|Enforce password history|24 days|
|Maximum password age|42 days|
|Minimum password age|1 day|
|Minimum password length|7|
|Password must meet complexity requirements|Enabled|
|Store passwords using reversible encryption|Disabled|
|Account lockout duration|Not set|
|Account lockout threshold|0|
|Reset account lockout counter after|Not set|

> [!warning]  
> The table above reflects the **source's stated default-policy values**. Do not assume every current Active Directory environment uses these values. Always enumerate the actual target domain's policy.

---

# 🧠 38. Analyzing the Password Policy

The source domain uses:

### 📏 Minimum Length

```text
8
```

Eight characters is common but relatively weak by modern standards.

Some organizations use:

```text
10–14+ characters
```

A longer minimum reduces some candidate passwords, but **does not eliminate password spraying**.

---

### 🚨 Lockout Threshold

```text
5
```

Five failed attempts can trigger lockout.

Other environments may use:

```text
3
```

or even:

```text
0
```

where no lockout threshold is configured.

> [!important]  
> Never assume `5` is universal.

---

### ⏱️ Lockout Duration

```text
30 minutes
```

If an account is locked, the policy indicates that it remains locked for 30 minutes.

---

### 🔄 Automatic Unlock

This environment automatically unlocks accounts after the configured duration.

However, some organizations require administrative intervention.

> [!warning]  
> A manual-unlock environment dramatically increases the operational impact of a password-spraying mistake.

Imagine accidentally locking:

```text
500 users
```

If an administrator has to manually unlock every account, the penetration test can cause a significant operational incident.

---

# 🧨 39. Password Spraying Safety

The source gives a very important penetration-testing principle:

> **We do not want to be the pentester that locks out every account in the organization!**

The password policy should be obtained **before** spraying whenever possible.

### Safe Methodology

```text
🔎 Obtain policy
      ↓
📊 Determine lockout threshold
      ↓
⏱️ Determine reset/lockout timing
      ↓
👤 Enumerate valid users
      ↓
🔐 Select candidate password
      ↓
🎯 Spray conservatively
      ↓
📊 Monitor results
      ↓
⏳ Respect timing window
      ↓
🔁 Continue only when safe
```

---

# ⚠️ 40. What If We Cannot Obtain the Policy?

Sometimes:

- External password spraying is being performed.
    
- SMB NULL sessions are disabled.
    
- LDAP anonymous binds are disabled.
    
- Credentials are unavailable.
    
- The client does not provide the policy.
    

In that case, the risk is significantly higher.

The source recommends:

- Exercise **extreme caution**.
    
- If appropriate, perform **one, maximum two** spraying attempts.
    
- Wait **over an hour** between attempts if two attempts are performed.
    
- Never assume the organization uses a five-attempt threshold.
    
- Never assume accounts automatically unlock.
    

> [!important]  
> The source's conservative guidance is intentionally stricter than simply calculating `lockout threshold - 1`.
> 
> **Real-world password policies vary.**

For example:

```text
Organization A
Threshold = 5
Duration = 30 min
Automatic unlock

Organization B
Threshold = 3
Manual administrator unlock
```

The second environment makes careless spraying far more dangerous.

---

# 🧠 41. Why Waiting Over an Hour?

Suppose we know nothing about the target policy.

We do **not** know:

```text
Lockout threshold
Lockout duration
Observation window
Automatic vs manual unlock
Existing failed-attempt count
```

Therefore:

```text
Unknown policy
     ↓
Unknown lockout state
     ↓
Unknown safe retry interval
     ↓
Higher risk
     ↓
Use a highly conservative interval
```

> [!warning]  
> Waiting does not magically make spraying safe. It is simply a conservative measure when the actual policy cannot be determined.

---

# 🎯 42. Password Policy → Spray Strategy

The important relationship is:

```text
Password Policy
      │
      ├── Minimum Length
      │
      ├── Complexity
      │
      ├── Lockout Threshold
      │
      ├── Lockout Duration
      │
      └── Observation Window
              │
              ▼
       Attack Constraints
              │
              ▼
       Password Candidate
              │
              ▼
         User Target List
              │
              ▼
       Controlled Spray
```

The password policy is therefore not just a piece of documentation.

It directly affects **how the next attack should be performed**.

---

# 👤 43. Next Step — Build the Target User List

Once the policy has been obtained:

```text
Password Policy
      ↓
Understand lockout constraints
      ↓
Enumerate users
      ↓
Build target list
      ↓
Select password candidates
      ↓
Controlled password spraying
```

Potential sources for usernames include:

- SMB/RPC enumeration
    
- NULL sessions
    
- LDAP enumeration
    
- Active Directory enumeration
    
- Previously captured usernames
    
- Application/user enumeration
    
- Other assessment findings
    

> [!important]  
> **Password policy enumeration and user enumeration are complementary.**
> 
> Knowing the policy tells you **how safely** you can test.
> 
> Knowing the users tells you **who** you can test.

---

# 🧠 44. The CPTS Mental Model

```text
                    🏢 ACTIVE DIRECTORY
                           │
                           ▼
                  🔎 ENUMERATION PHASE
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
       👤 USER LIST                🔐 PASSWORD POLICY
             │                           │
             │                    ┌──────┴──────┐
             │                    │             │
             │                Threshold      Timing
             │                    │             │
             └────────────┬───────┴─────────────┘
                          ▼
                  🎯 SPRAY DECISION
                          │
                          ▼
                 🔑 Candidate Password
                          │
                          ▼
                   👤 Target Users
                          │
                          ▼
                 🚨 LOCKOUT SAFETY
                          │
                          ▼
                   Controlled Test
                          │
                          ▼
                  🔓 Valid Credentials
                          │
                          ▼
                 🔁 Further Enumeration
```

---

# 🔗 45. Tool Reference Links

- [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) — SMB/Windows network enumeration and execution framework. The referenced repository is archived.
    
- [rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) — Samba MS-RPC client.
    
- [Samba](https://www.samba.org/) — Suite containing `rpcclient`, `smbclient`, `nmblookup`, `net`, and other SMB tooling.
    
- [enum4linux](https://labs.portcullis.co.uk/tools/enum4linux/) — Windows/Samba enumeration wrapper.
    
- [enum4linux-ng](https://github.com/cddmp/enum4linux-ng) — Python rewrite with structured JSON/YAML output and additional enumeration functionality.
    
- [ldapsearch / OpenLDAP](https://www.openldap.org/) — LDAP command-line tooling.
    
- [PowerSploit / PowerView](https://github.com/PowerShellMafia/PowerSploit) — PowerShell post-exploitation/enumeration framework containing PowerView.
    
- [nmblookup](https://www.samba.org/samba/docs/current/man-html/nmblookup.1.html) — Samba NetBIOS name lookup utility.
    
- [smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) — Samba SMB/CIFS client.
    

---

# 🧠 46. Key Takeaways

> [!success]  
> **CPTS Exam Mental Model**
> 
> - **Password policy enumeration should happen before password spraying whenever possible.**
>     
> - With valid credentials, use SMB/RPC-based methods such as `crackmapexec --pass-pol` or `rpcclient`.
>     
> - Without credentials, test for:
>     
>     - **SMB NULL sessions**
>         
>     - **LDAP anonymous binds**
>         
> - `rpcclient -U "" -N <IP>` tests for unauthenticated SMB/RPC access.
>     
> - `querydominfo` confirms what domain information the RPC session can expose.
>     
> - `getdompwinfo` retrieves core password-policy information.
>     
> - `enum4linux -P <IP>` is a convenient password-policy enumeration method.
>     
> - `enum4linux-ng -P <IP> -oA <name>` provides structured output that is easier to process.
>     
> - On Windows, `net use \\DC01\ipc$ "" /u:""` can establish a NULL IPC$ session.
>     
> - `net accounts` provides password and lockout policy information using built-in Windows functionality.
>     
> - `Get-DomainPolicy` provides similar information through PowerView.
>     
> - LDAP policy attributes such as `minPwdLength`, `lockoutThreshold`, `pwdProperties`, and `pwdHistoryLength` are important.
>     
> - **Password complexity does not mean passwords are strong.**
>     
> - The most important spraying values are:
>     
> 
> ```text
> 🚨 Lockout Threshold
> ⏱️ Lockout Duration
> 🔄 Observation Window
> 🔐 Password Complexity
> 📏 Minimum Password Length
> ```
> 
> - Never assume the policy is `5 attempts / 30 minutes`.
>     
> - Never assume accounts automatically unlock.
>     
> - If the policy cannot be obtained, **extreme caution** is required.
>     
> - The overall CPTS workflow is:
>     
> 
> ```text
> 🔎 ENUMERATE
>      ↓
> 🔐 RETRIEVE PASSWORD POLICY
>      ↓
> 🚨 UNDERSTAND LOCKOUT RISK
>      ↓
> 👤 ENUMERATE USERS
>      ↓
> 🎯 BUILD TARGET LIST
>      ↓
> 🔑 CONTROLLED PASSWORD SPRAY
>      ↓
> 🔓 VALID CREDENTIALS
>      ↓
> 🔁 CONTINUE ENUMERATION / LATERAL MOVEMENT
> ```