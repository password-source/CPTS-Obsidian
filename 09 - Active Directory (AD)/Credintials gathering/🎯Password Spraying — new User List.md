# 🎯 1. Password Spraying — Making a Target User List

> [!info] Objective  
> Before performing a password spray, we need a list of **valid domain usernames**. The quality of this list directly affects the effectiveness of the spray.
> 
> The general workflow is:
> 
> ```text
> 🔎 ENUMERATE USERS
>        ↓
> 🧹 CLEAN / FILTER USERNAMES
>        ↓
> 🔐 ENUMERATE PASSWORD POLICY
>        ↓
> ⚠️ CHECK LOCKOUT STATE / THRESHOLD
>        ↓
> 🎯 BUILD SPRAY TARGET LIST
>        ↓
> 🔑 PASSWORD SPRAY
> ```

## ⚡ 2. Command Cheat Sheet

|Goal|Command|
|---|---|
|Enumerate users with enum4linux|`enum4linux -U 172.16.5.5`|
|Anonymous RPC connection|`rpcclient -U "" -N 172.16.5.5`|
|Enumerate users with rpcclient|`enumdomusers`|
|Enumerate users with CrackMapExec|`crackmapexec smb 172.16.5.5 --users`|
|Enumerate users with LDAP|`ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))"`|
|Anonymous LDAP users with windapsearch|`./windapsearch.py --dc-ip 172.16.5.5 -u "" -U`|
|Kerberos username enumeration|`kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt`|
|Credentialed user enumeration|`sudo crackmapexec smb 172.16.5.5 -u htb-student -p Academy_student_AD! --users`|

> [!important] CPTS focus  
> The important skill is not memorizing one tool. It is recognizing that **different levels of access give you different ways to build the same target-user list**.

---

# 🔎 3. Detailed User Enumeration

A successful password spray starts with **valid usernames**.

Possible sources include:

1. **SMB NULL session**
    
    - Retrieve domain users from a Domain Controller without valid credentials when the configuration permits it.
        
2. **LDAP anonymous bind**
    
    - Query Active Directory anonymously and retrieve users and attributes.
        
3. **Kerbrute**
    
    - Test a username wordlist against Kerberos.
        
    - Useful when we have no authenticated access and cannot use SMB NULL/LDAP anonymous enumeration.
        
4. **Credentials already obtained**
    
    - Credentials may come from:
        
        - Client-provided credentials
            
        - LLMNR/NBT-NS poisoning
            
        - A previous password spray
            
        - Other authorized assessment activity
            
5. **External information gathering**
    
    - Email harvesting
        
    - LinkedIn
        
    - Username-generation tools such as `linkedin2username`
        

### 🧠 Core Mental Model

```text
             ┌─────────────────────┐
             │  Need valid users   │
             └──────────┬──────────┘
                        ↓
          ┌─────────────────────────────┐
          │ What access do I have?      │
          └─────────────┬───────────────┘
                        │
       ┌────────────────┼────────────────┐
       ↓                ↓                ↓
  NULL SESSION    LDAP ANONYMOUS     CREDENTIALS
       │                │                │
       ↓                ↓                ↓
  enum4linux        ldapsearch       CME
  rpcclient         windapsearch      PowerView
       │                │                │
       └────────────────┼────────────────┘
                        ↓
                  USER LIST
                        ↑
                        │
                 No internal access?
                        │
                        ↓
                    Kerbrute
                        │
                        ↓
                External OSINT
```

---

# 🔐 4. Password Policy Must Come Before Spraying

Knowing the usernames is only half the problem.

We also need to understand the domain's password policy.

Important values include:

|Policy Value|Why It Matters|
|---|---|
|Minimum password length|Helps determine plausible passwords|
|Password complexity|Helps determine what password formats may satisfy policy|
|Lockout threshold|Determines how many failed attempts can trigger lockout|
|Lockout duration|Determines how long a locked account remains locked|
|Reset/bad-password timer|Determines when failed-attempt counters reset|

> [!warning] Never assume the policy  
> The exact lockout behavior must be determined from the target environment or client-provided rules. A threshold such as `5` does **not** mean that five attempts are universally safe. Distributed authentication and multiple Domain Controllers can affect behavior.

If the password policy is unavailable, the source recommends either:

- One highly targeted attempt as a last resort, after other foothold options have been exhausted.
    
- Very widely spaced attempts, such as one spray every few hours, to reduce the likelihood of lockout.
    

During an authorized penetration test, follow the **client-approved lockout limits first**.

---

# 📝 5. Maintain a Spray Activity Log

Regardless of how the usernames were obtained, maintain a detailed record.

At minimum, record:

- **Accounts targeted**
    
- **Domain Controller used**
    
- **Time of spray**
    
- **Date of spray**
    
- **Password(s) attempted**
    

### Why?

The log prevents duplicate attempts and provides a record that can be compared with the client's authentication logs.

```text
Your activity log
       │
       ├── Who was targeted?
       ├── Which DC?
       ├── When?
       └── Which password?
               │
               ↓
        Client SIEM / Logs
               │
               ↓
      Correlate suspicious
       authentication events
```

> [!success] Professional pentesting habit  
> A spray log is not just for organization. If an account becomes locked or the client detects suspicious authentication activity, your records allow the client to distinguish your authorized testing from unrelated activity.

---

# 🖥️ 6. SMB NULL Session → User List

If we are inside the network but **do not have valid domain credentials**, first look for:

- SMB NULL sessions
    
- LDAP anonymous binds
    

Both may provide an accurate domain-user list and password policy.

If we already have:

- A valid domain account, or
    
- `SYSTEM` access on a Windows host
    

then querying Active Directory becomes much easier.

### 🧠 Why can SYSTEM be useful?

A Windows computer account is treated as a domain security principal with important similarities to a domain user account.

`SYSTEM` can operate in the security context of the computer, allowing the host to authenticate as its computer account.

```text
Windows Host
     │
     └── SYSTEM
          │
          ↓
   Computer Account
          │
          ↓
   Domain Authentication
          │
          ↓
   Query Active Directory
```

> [!info] Important distinction  
> A computer account is not simply a normal human user account. It has different properties and behaviors, including differences around authentication across forest trusts. The important CPTS concept is that the computer itself has a domain identity.

If neither authenticated enumeration nor anonymous enumeration works, we can fall back to external information gathering:

```text
Company information
       ↓
Email addresses
       ↓
Naming convention
       ↓
Potential usernames
       ↓
Kerberos validation
```

---

# 🧰 7. SMB-Based Enumeration Tools

Useful tools include:

- [enum4linux](https://github.com/portcullislabs/enum4linux?utm_source=chatgpt.com)
    
- [rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html?utm_source=chatgpt.com)
    
- [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec?utm_source=chatgpt.com)
    

Regardless of the tool, the objective is:

```text
Raw enumeration output
        ↓
Extract usernames
        ↓
Remove unnecessary information
        ↓
username
username
username
...
```

---

# 👤 8. Using enum4linux

The `-U` option retrieves domain users.

```bash
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

Example output:

```text
administrator
guest
krbtgt
lab_adm
htb-student
avazquez
pfalcon
fanthony
wdillard
lbradford
sgage
asanchez
dbranch
ccruz
njohnson
mholliday

<SNIP>
```

### 🧠 What is happening?

The command is a small parsing pipeline:

```text
enum4linux -U
     │
     ↓
Enumerate domain users
     │
     ↓
grep "user:"
     │
     ↓
Keep only user records
     │
     ↓
cut -f2 -d"["
     │
     ↓
Extract username portion
     │
     ↓
cut -f1 -d"]"
     │
     ↓
Output clean usernames
```

The result is much more useful for subsequent tooling because each line contains only a username.

---

# 🖥️ 9. Using rpcclient

Connect anonymously:

```bash
rpcclient -U "" -N 172.16.5.5
```

Then enumerate domain users:

```text
rpcclient $> enumdomusers
user:[administrator] rid:[0x1f4]
user:[guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[lab_adm] rid:[0x3e9]
user:[htb-student] rid:[0x457]
user:[avazquez] rid:[0x458]

<SNIP>
```

### 🔍 Key command

```text
enumdomusers
```

asks the RPC service for the domain's user objects.

### 🧠 Mental Model

```text
SMB
 ↓
Anonymous RPC session
 ↓
RPC domain enumeration
 ↓
enumdomusers
 ↓
Domain users + RIDs
```

The RIDs are useful domain identifiers, but for password spraying our immediate objective is the **username**.

---

# 👥 10. CrackMapExec `--users`

[CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec?utm_source=chatgpt.com) can enumerate users with:

```bash
crackmapexec smb 172.16.5.5 --users
```

Example:

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] Enumerated domain user(s)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\administrator                  badpwdcount: 0 baddpwdtime: 2022-01-10 13:23:09.463228
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\guest                          badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\lab_adm                        badpwdcount: 0 baddpwdtime: 2021-12-21 14:10:56.859064
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\krbtgt                         badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\htb-student                    badpwdcount: 0 baddpwdtime: 2022-02-22 14:48:26.653366
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\avazquez                       badpwdcount: 0 baddpwdtime: 2022-02-17 22:59:22.684613

<SNIP>
```

## 🔎 `badpwdcount`

`badpwdcount` represents the number of invalid password attempts associated with the account as reported by the queried Domain Controller.

This is extremely useful when preparing a spray because an account already close to its lockout threshold deserves special attention.

Example:

```text
Account          badpwdcount
────────────────────────────
administrator        0
guest                0
lab_adm              0
avazquez            20
```

If the policy threshold is `5`, an account reporting `20` indicates that something about the environment or account state needs to be investigated rather than blindly attempting authentication.

> [!warning] Do not blindly calculate spray attempts  
> Treat `badpwdcount` as **DC-specific state**, not a universal organization-wide counter.

---

# ⏱️ 11. `baddpwdtime`

`baddpwdtime` records the date/time of the last bad-password attempt.

This can help determine how recently an invalid authentication occurred and how close the account may be to having its bad-password counter reset.

### Multiple Domain Controllers

In an environment with multiple DCs, the value is maintained separately on each DC.

Therefore:

```text
                Domain
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
      DC01       DC02      DC03
        │         │         │
   badpwdcount badpwdcount badpwdcount
        │         │         │
        └─────────┼─────────┘
                  ↓
        Not automatically one
        global counter
```

To determine the total accurately, the source explains that we would need to:

1. Query each Domain Controller and use the sum of the values, **or**
    
2. Query the Domain Controller holding the **PDC Emulator FSMO role**.
    

> [!important] CPTS concept  
> The PDC Emulator is particularly important for password-change and authentication-related operations. When you see password-attempt counters and multiple DCs, remember that **DC-local state matters**.

---

# 🌐 12. LDAP Anonymous Bind → User List

If LDAP anonymous binding is possible, we can retrieve users without valid credentials.

Useful tools include:

- `windapsearch`
    
- [ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch&utm_source=chatgpt.com)
    

LDAP queries require an appropriate search filter.

The source uses:

```text
(&(objectclass=user))
```

This asks LDAP to return objects whose `objectclass` includes `user`.

---

# 🔍 13. Using ldapsearch

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
```

Example:

```text
guest
ACADEMY-EA-DC01$
ACADEMY-EA-MS01$
ACADEMY-EA-WEB01$
htb-student
avazquez
pfalcon
fanthony
wdillard
lbradford
sgage
asanchez
dbranch

<SNIP>
```

### 🧠 Breakdown

```text
ldapsearch
   │
   ├── -h 172.16.5.5
   │      └── LDAP server
   │
   ├── -x
   │      └── Simple authentication
   │
   ├── -b "DC=INLANEFREIGHT,DC=LOCAL"
   │      └── Search base
   │
   ├── -s sub
   │      └── Search the subtree
   │
   └── "(&(objectclass=user))"
          └── LDAP search filter
```

Then:

```text
grep sAMAccountName:
        ↓
Keep account names
        ↓
cut
        ↓
Produce clean username list
```

> [!info] Computer accounts  
> Entries ending in `$`, such as `ACADEMY-EA-DC01$`, are **computer accounts**, not normal human user accounts. For a human-focused spray list, these would normally need to be treated separately.

---

# 🧰 14. Using windapsearch

[windapsearch](https://github.com/ropnop/windapsearch?utm_source=chatgpt.com) makes anonymous LDAP enumeration easier.

```bash
./windapsearch.py --dc-ip 172.16.5.5 -u "" -U
```

Example:

```text
[+] No username provided. Will try anonymous bind.
[+] Using Domain Controller at: 172.16.5.5
[+] Getting defaultNamingContext from Root DSE
[+] Found: DC=INLANEFREIGHT,DC=LOCAL
[+] Attempting bind
[+] ...success! Binded as: 
[+]  None

[+] Enumerating all AD users
[+] Found 2906 users: 

cn: Guest

cn: Htb Student
userPrincipalName: htb-student@inlanefreight.local

cn: Annie Vazquez
userPrincipalName: avazquez@inlanefreight.local

cn: Paul Falcon
userPrincipalName: pfalcon@inlanefreight.local

cn: Fae Anthony
userPrincipalName: fanthony@inlanefreight.local

cn: Walter Dillard
userPrincipalName: wdillard@inlanefreight.local

<SNIP>
```

### Important options

```text
--dc-ip 172.16.5.5
    Specify the Domain Controller.

-u ""
    Blank username → attempt anonymous bind.

-U
    Retrieve/enumerate users.
```

### Mental Model

```text
Root DSE
   ↓
Discover naming context
   ↓
DC=INLANEFREIGHT,DC=LOCAL
   ↓
Anonymous LDAP bind
   ↓
Enumerate AD users
   ↓
Clean target list
```

---

# ⚡ 15. Kerbrute Username Enumeration

When we have **no internal authentication access** and SMB NULL/LDAP anonymous enumeration is unavailable, Kerberos can provide another path for validating usernames.

[Kerbrute](https://github.com/ropnop/kerbrute?utm_source=chatgpt.com) uses Kerberos behavior to determine whether usernames exist.

### 🔑 Core concept

Kerberos Pre-Authentication provides a useful distinction during username enumeration.

The tool sends TGT requests to the Domain Controller.

Conceptually:

```text
Candidate username
        ↓
      KDC
        │
        ├── PRINCIPAL UNKNOWN
        │       ↓
        │   ❌ Invalid username
        │
        └── Pre-Authentication required
                ↓
             ✅ Valid username
```

The key idea is that the KDC responds differently depending on whether the principal exists.

> [!important] Why this is useful  
> Username validation can be performed without actually authenticating with a password, so this enumeration phase does not produce the same failed-logon behavior as blindly attempting passwords.

---

# 🧪 16. Kerbrute User Enumeration

The source uses the `jsmith.txt` list containing **48,705 possible usernames** in the `flast` format.

The wordlist comes from the [statistically-likely-usernames repository](https://github.com/insidetrust/statistically-likely-usernames?utm_source=chatgpt.com).

Command:

```bash
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

Example output:

```text
    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: dev (9cfb81e) - 02/17/22 - Ronnie Flathers @ropnop

2022/02/17 22:16:11 >  Using KDC(s):
2022/02/17 22:16:11 >   172.16.5.5:88

2022/02/17 22:16:11 >  [+] VALID USERNAME:   jjones@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   sbrown@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   tjohnson@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   jwilson@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   bdavis@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   njohnson@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   asanchez@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   dlewis@inlanefreight.local
2022/02/17 22:16:11 >  [+] VALID USERNAME:   ccruz@inlanefreight.local

<SNIP>
```

The source reports:

- More than **48,000 usernames** checked
    
- Approximately **12 seconds**
    
- More than **50 valid usernames** discovered
    

This demonstrates why Kerberos username enumeration can be extremely efficient.

---

# 🛡️ 17. Kerbrute Detection & Lockout Considerations

Kerberos username enumeration and password spraying are **not the same operation**.

### Username enumeration

```text
Candidate username
       ↓
KDC
       ↓
Does principal exist?
       ↓
Yes / No
```

This does **not** involve repeatedly submitting incorrect passwords.

### Password spraying

```text
Valid username
       +
Candidate password
       ↓
KDC authentication attempt
       ↓
Success / Failure
```

Failed authentication attempts can contribute toward account lockout.

> [!warning] Critical distinction  
> Kerberos-based **username enumeration** does not mean Kerberos-based **password spraying** is lockout-free.
> 
> Once switching to password attempts, failed Kerberos pre-authentication attempts can count toward failed-login/account-lockout mechanisms.

---

# 📊 18. Windows Event Visibility

Kerbrute username enumeration generates **Event ID 4768**:

> **A Kerberos authentication ticket (TGT) was requested**

The source notes that this event is only triggered if Kerberos event logging is enabled through Group Policy.

Therefore defenders can monitor for an unusual influx of Event ID `4768`.

```text
Many username guesses
        ↓
Many TGT requests
        ↓
Event ID 4768
        ↓
SIEM correlation
        ↓
Potential username-enumeration activity
```

> [!success] CPTS reporting insight  
> If a penetration test successfully demonstrates this technique, the observation can become a useful defensive recommendation: monitor and correlate unusual Kerberos TGT-request patterns rather than relying only on failed-logon Event ID `4625`.

---

# 🌐 19. External Username Discovery

If internal enumeration fails, return to external information gathering.

Possible sources:

- Company email addresses
    
- Company websites
    
- LinkedIn
    
- Public employee information
    

A tool such as [linkedin2username](https://github.com/initstring/linkedin2username?utm_source=chatgpt.com) can generate possible usernames from a company's LinkedIn information.

The generated names are **potential usernames**, not automatically valid accounts.

Therefore:

```text
External information
        ↓
Potential usernames
        ↓
Kerbrute validation
        ↓
Valid usernames
        ↓
Target list
```

---

# 🔑 20. Credentialed Enumeration

If we obtain valid credentials, user enumeration becomes significantly easier.

The source uses [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec?utm_source=chatgpt.com):

```bash
sudo crackmapexec smb 172.16.5.5 -u htb-student -p Academy_student_AD! --users
```

Example:

```text
[sudo] password for htb-student: 
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\htb-student:Academy_student_AD! 
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] Enumerated domain user(s)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\administrator                  badpwdcount: 1 baddpwdtime: 2022-02-23 21:43:35.059620
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\guest                          badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\lab_adm                        badpwdcount: 0 baddpwdtime: 2021-12-21 14:10:56.859064
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\krbtgt                         badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\htb-student                    badpwdcount: 0 baddpwdtime: 2022-02-22 14:48:26.653366
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\avazquez                       badpwdcount: 20 baddpwdtime: 2022-02-17 22:59:22.684613
SMB         172.16.5.5      445    ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\pfalcon                        badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58

<SNIP>
```

---

# 🧠 21. How the Enumeration Methods Fit Together

The most important CPTS skill is choosing the enumeration method based on your current access.

|Current Situation|Best Enumeration Path|
|---|---|
|Anonymous SMB/RPC available|`enum4linux` / `rpcclient`|
|LDAP anonymous bind available|`ldapsearch` / `windapsearch`|
|Valid domain credentials|`CrackMapExec --users` or other authenticated AD enumeration|
|`SYSTEM` on domain-joined Windows host|Use computer-account/domain context to query AD|
|No credentials + no anonymous SMB/LDAP|`Kerbrute` + username wordlist|
|No useful internal enumeration|External OSINT → potential usernames → validate|

### 🎯 Decision Tree

```text
                    START
                      │
                      ↓
             Need domain users
                      │
             ┌────────┴────────┐
             ↓                 ↓
       Have credentials?      NO
             │                 │
            YES                ↓
             │          SMB NULL available?
             │           ┌─────┴─────┐
             │          YES          NO
             │           │            │
             │           ↓            ↓
             │      enum4linux    LDAP anonymous?
             │      rpcclient      ┌───┴───┐
             │                     YES     NO
             │                      │       │
             │                      ↓       ↓
             │                 ldapsearch  Kerbrute
             │                 windapsearch     │
             │                              ↓
             └──────────────┬───────────────┘
                            ↓
                     Clean user list
                            ↓
                 Check password policy
                            ↓
                 Check account state
                            ↓
                   Build spray targets
```

---

# ⚠️ 22. Important CPTS Distinctions

### `badpwdcount` ≠ global failed-attempt count

With multiple DCs:

```text
DC01 → badpwdcount = X
DC02 → badpwdcount = Y
DC03 → badpwdcount = Z
```

These values can be maintained separately.

### Username enumeration ≠ password spraying

```text
Username enumeration
    → determine whether account exists

Password spraying
    → attempt a password against valid accounts
```

The first can avoid failed-password events; the second can cause lockouts.

### Valid user ≠ human user

LDAP/AD enumeration can return:

- Human accounts
    
- Service accounts
    
- Computer accounts
    
- Built-in accounts
    
- Special accounts
    

Therefore the raw enumeration result may require filtering before becoming a spray target list.

---

# 🧠 23. Strong CPTS Mental Model

```text
                    🔎 ENUMERATION
                         │
         ┌───────────────┼────────────────┐
         ↓               ↓                ↓
    SMB NULL        LDAP ANONYMOUS    CREDENTIALS
         │               │                │
    enum4linux       ldapsearch          CME
    rpcclient        windapsearch         │
         │               │                │
         └───────────────┼────────────────┘
                         ↓
                   👥 USER LIST
                         │
                         │
                 No internal access?
                         ↓
                     Kerbrute
                         │
                         ↓
                  VALIDATE USERS
                         │
                         ↓
                  🧹 FILTER LIST
                         │
                         ↓
                🔐 PASSWORD POLICY
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
       Length        Complexity      Lockout
          │              │              │
          └──────────────┼──────────────┘
                         ↓
                  🎯 SPRAY TARGETS
                         │
                         ↓
                  📝 LOG EVERYTHING
                         │
                         ↓
                 Authorized testing
```

> [!tip] The key question  
> **“What access do I have right now, and what does that access allow me to enumerate?”**
> 
> CPTS methodology is about adapting the next enumeration step to the access you currently possess rather than blindly running tools.

---

# ✅ 24. Key Takeaways

- A password spray requires a **valid target-user list**.
    
- **SMB NULL sessions** can expose domain users when permitted.
    
- `enum4linux -U` provides a convenient user-enumeration path.
    
- `rpcclient` can enumerate users with `enumdomusers`.
    
- `CrackMapExec --users` can provide usernames plus `badpwdcount` and `baddpwdtime`.
    
- `badpwdcount` is important when assessing account-lockout risk.
    
- In multi-DC environments, bad-password state can be **DC-specific**.
    
- **LDAP anonymous bind** can expose AD users through `ldapsearch` or `windapsearch`.
    
- LDAP filters such as `(&(objectclass=user))` are fundamental to understanding AD enumeration.
    
- **Kerbrute** can validate usernames through Kerberos behavior when authenticated enumeration is unavailable.
    
- Kerberos username enumeration can be much faster than conventional authentication attempts.
    
- Username enumeration and password spraying have **different lockout and logging implications**.
    
- Kerbrute username enumeration can generate **Event ID 4768** when relevant Kerberos logging is enabled.
    
- External OSINT can produce potential usernames when internal enumeration fails.
    
- Always enumerate or obtain the **password policy before spraying whenever possible**.
    
- Always maintain a **spray activity log**.
    
- Never assume that a particular number of attempts is universally safe; follow the target's actual policy and client-approved testing limits.