# 🎯 1. Internal Password Spraying — From Windows

> [!info] Objective  
> When we obtain a foothold on a **domain-joined Windows host**, we can perform internal password spraying directly from Windows.
> 
> The primary tool covered here is [DomainPasswordSpray](https://github.com/dafthack/DomainPasswordSpray?utm_source=chatgpt.com).
> 
> When authenticated to the domain, it can automate several steps that we previously performed manually from Linux:
> 
> ```text
> 👤 Current domain context
>          ↓
> 📋 Enumerate domain users
>          ↓
> 🔐 Query password policy
>          ↓
> ⚠️ Identify accounts near lockout
>          ↓
> 🧹 Remove unsafe targets
>          ↓
> 🎯 Password spray
>          ↓
> 💾 Save successful credentials
> ```

---

# ⚡ 2. Command Cheat Sheet

|Goal|Command|
|---|---|
|Load DomainPasswordSpray|`Import-Module .\DomainPasswordSpray.ps1`|
|Domain password spray|`Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue`|
|Kerbrute from Windows|`Kerbrute` can be used for the same enumeration/spraying workflow from the previous section|

---

# 🪟 3. Why Spray From Windows?

A Windows foothold can be particularly useful when the compromised system is already **domain joined**.

Possible scenarios:

- Client provides a managed Windows workstation for testing.
    
- Tester is physically on-site and uses a Windows VM.
    
- An initial compromise provides access to a domain-joined Windows host.
    
- We have authenticated to a domain host and want to obtain credentials for an account with greater privileges.
    

### 🧠 Mental Model

```text
Initial foothold
      ↓
Domain-joined Windows host
      ↓
Existing domain context
      ↓
Enumerate AD users
      ↓
Read password policy
      ↓
Remove unsafe accounts
      ↓
Password spray
      ↓
Higher-value credentials
      ↓
Credentialed domain enumeration
```

> [!important] Why this differs from Linux  
> On Linux, we previously had to build the user list and inspect the password policy ourselves.
> 
> When authenticated on a domain-joined Windows host, `DomainPasswordSpray` can automate much of this process.

---

# 🛠️ 4. DomainPasswordSpray

Import the PowerShell module:

```powershell
Import-Module .\DomainPasswordSpray.ps1
```

Then execute:

```powershell
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```

### Important arguments

```text
-Password Welcome1
    The single password to spray.

-OutFile spray_success
    Save successful results to a file.

-ErrorAction SilentlyContinue
    Suppress non-critical PowerShell errors/noise.
```

Because the Windows host is domain joined and authenticated, we do **not** need to provide `-UserList` in this example.

---

# 🔎 5. What DomainPasswordSpray Does Automatically

The tool reports:

```text
[*] Current domain is compatible with Fine-Grained Password Policy.
[*] Now creating a list of users to spray...
[*] The smallest lockout threshold discovered in the domain is 5 login attempts.
[*] Removing disabled users from list.
[*] There are 2923 total users found.
[*] Removing users within 1 attempt of locking out from list.
[*] Created a userlist containing 2923 users gathered from the current user's domain
[*] The domain password policy observation window is set to  minutes.
[*] Setting a  minute wait in between sprays.
```

The important workflow is:

```text
                  DomainPasswordSpray
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
        Find users     Read policy    Check state
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                  Remove unsafe users
                           ↓
                    Build spray list
```

### 🧠 Why this matters

The dangerous part of password spraying is not simply discovering a password.

The dangerous part is **causing account lockouts**.

Therefore, automatically identifying accounts close to the lockout threshold helps reduce accidental lockouts.

---

# 🔐 6. Fine-Grained Password Policies

The tool reports:

```text
[*] Current domain is compatible with Fine-Grained Password Policy.
```

This matters because Active Directory can have password policies that apply differently to different users/groups rather than relying exclusively on one domain-wide policy.

Therefore, when spraying, we should not blindly assume that:

```text
Every user
      ↓
Same password policy
```

Instead:

```text
Domain
  │
  ├── Policy A
  │      └── Group / users
  │
  ├── Policy B
  │      └── Group / users
  │
  └── Default domain policy
```

> [!important] CPTS concept  
> **Password policy enumeration is part of attack planning**, not just a box to check. Different policies can change lockout and password requirements for different accounts.

---

# 🧹 7. Automatic Target Filtering

The tool reports:

```text
[*] Removing disabled users from list.
```

Disabled accounts are not useful spray targets.

It then reports:

```text
[*] Removing users within 1 attempt of locking out from list.
```

This is particularly important.

### Example

Suppose:

```text
Lockout threshold = 5
```

and an account is already at:

```text
badpwdcount = 4
```

One more failed attempt could potentially trigger the lockout.

Therefore:

```text
Account
  ↓
1 attempt from lockout
  ↓
❌ Remove from spray list
```

---

# 📋 8. Automatically Generated User List

The example reports:

```text
[*] There are 2923 total users found.
```

and then:

```text
[*] Created a userlist containing 2923 users gathered from the current user's domain
```

The important difference from our Linux workflow is:

```text
🐧 Linux
   ↓
Manually obtain users
   ↓
Manually inspect policy
   ↓
Manually filter targets
   ↓
Spray


🪟 Windows + DomainPasswordSpray
   ↓
Automatically enumerate users
   ↓
Automatically inspect policy
   ↓
Automatically remove unsafe accounts
   ↓
Spray
```

---

# ⏱️ 9. Password Policy Observation Window

The tool also determines the domain's password-policy observation window:

```text
[*] The domain password policy observation window is set to  minutes.
[*] Setting a  minute wait in between sprays.
```

The observation window is relevant because failed-password counters can reset after the configured period.

Conceptually:

```text
Failed attempt
      ↓
bad password counter
      ↓
Observation window
      ↓
Counter reset
```

> [!warning] Do not blindly trust timing  
> The exact output shown in the source has blank minute values. Do not infer a safe timing value from this example. In a real assessment, follow the actual domain policy and client-approved spray limits.

---

# 🚨 10. Confirmation Before Spraying

Before starting, the tool asks:

```text
Confirm Password Spray
Are you sure you want to perform a password spray against 2923 accounts?
[Y] Yes  [N] No  [?] Help (default is "Y"): Y
```

This is an important safety checkpoint.

The tool explicitly tells the tester:

```text
[*] Password spraying has begun with  1  passwords
[*] This might take a while depending on the total number of users
[*] Now trying password Welcome1 against 2923 users. Current time is 2:57 PM
```

The important methodology is:

```text
User list
   ↓
Policy-aware filtering
   ↓
Explicit confirmation
   ↓
ONE password
   ↓
Many accounts
```

---

# ✅ 11. Successful Results

The source obtains:

```text
[*] Writing successes to spray_success
[*] SUCCESS! User:sgage Password:Welcome1
[*] SUCCESS! User:tjohnson Password:Welcome1

[*] Password spraying is complete
[*] Any passwords that were successfully sprayed have been output to spray_success
```

This gives us two valid credential sets:

```text
sgage    : Welcome1
tjohnson : Welcome1
```

and the successful results are stored in:

```text
spray_success
```

### 🧠 Why save the results?

Because a successful password spray is not the end of the assessment.

The credentials become inputs for the next phase:

```text
Valid credentials
       ↓
Credentialed enumeration
       ↓
Permissions / groups
       ↓
Accessible systems
       ↓
Lateral movement
       ↓
Privilege escalation
```

---

# 🐧 12. Kerbrute from Windows

The same general Kerberos enumeration and spraying techniques from the previous section can also be performed from Windows using [Kerbrute](https://github.com/ropnop/kerbrute?utm_source=chatgpt.com).

The source notes that `Kerbrute` is available in:

```text
C:\Tools
```

The methodology remains the same:

```text
Kerbrute
   ↓
Username enumeration
   ↓
Valid user list
   ↓
Password spraying
   ↓
Valid credentials
```

---

# 🛡️ 13. Mitigations Against Password Spraying

No single control completely eliminates password spraying.

The strongest defense is **defense in depth**.

|TechniqueDescription||
|---|---|
|`Multi-factor Authentication`|Multi-factor authentication can greatly reduce the risk of password spraying attacks. Many types of multi-factor authentication exist, such as push notifications to a mobile device, a rotating One Time Password (OTP) such as Google Authenticator, RSA key, or text message confirmations. While this may prevent an attacker from gaining access to an account, certain multi-factor implementations still disclose if the username/password combination is valid. It may be possible to reuse this credential against other exposed services or applications. It is important to implement multi-factor solutions with all external portals.|
|`Restricting Access`|It is often possible to log into applications with any domain user account, even if the user does not need to access it as part of their role. In line with the principle of least privilege, access to the application should be restricted to those who require it.|
|`Reducing Impact of Successful Exploitation`|A quick win is to ensure that privileged users have a separate account for any administrative activities. Application-specific permission levels should also be implemented if possible. Network segmentation is also recommended because if an attacker is isolated to a compromised subnet, this may slow down or entirely stop lateral movement and further compromise.|
|`Password Hygiene`|Educating users on selecting difficult to guess passwords such as passphrases can significantly reduce the efficacy of a password spraying attack. Also, using a password filter to restrict common dictionary words, names of months and seasons, and variations on the company's name will make it quite difficult for an attacker to choose a valid password for spraying attempts.|

---

# 🔐 14. Mitigation #1 — Multi-Factor Authentication

MFA adds an authentication factor beyond the password.

Examples include:

- Push notifications
    
- One-Time Passwords (OTP)
    
- RSA keys/tokens
    
- SMS confirmations
    

Conceptually:

```text
Password
   +
Second factor
   ↓
Authentication
```

This greatly reduces the usefulness of a discovered password.

> [!warning] MFA is not magic  
> Some MFA implementations may still reveal that the username/password combination was valid even when the second factor prevents access. Therefore, MFA should be combined with monitoring, access restrictions, and strong authentication design.

External portals should receive particular attention because they are directly reachable by attackers.

---

# 🚪 15. Mitigation #2 — Restricting Access

A common weakness is allowing any domain user to authenticate to an application even when that user does not require access.

The principle of least privilege says:

```text
User
  ↓
Does this role require access?
  │
 ┌┴─────────┐
 ↓          ↓
YES         NO
 ↓          ↓
Allow      Deny
```

Reducing the number of applications accessible with ordinary domain credentials reduces the attack surface available to a successful spray.

---

# 🧱 16. Mitigation #3 — Reduce the Impact of Successful Exploitation

Even if an attacker obtains valid credentials, we want to limit what those credentials can do.

### Separate administrative accounts

Instead of:

```text
User account
    +
Daily activities
    +
Administrative privileges
```

use:

```text
Normal account
    ↓
Daily work


Separate admin account
    ↓
Administrative operations
```

This limits the damage if the ordinary user credential is compromised.

### Application-specific permissions

Do not automatically give every authenticated domain user access to every application.

### Network segmentation

Segmentation can restrict lateral movement:

```text
Compromised subnet
       │
       ├── Allowed → Required services
       │
       └── Blocked → Sensitive network
```

This can slow or stop further compromise even after valid credentials are obtained.

---

# 🔑 17. Mitigation #4 — Password Hygiene

Users should choose passwords that are difficult to guess.

Passphrases can significantly reduce the effectiveness of common password spraying attempts.

Organizations can also use password filters to prevent common predictable choices such as:

- Dictionary words
    
- Months
    
- Seasons
    
- Company names
    
- Variations of company names
    

### Why?

Attackers often build candidate passwords from predictable organizational patterns.

```text
Company: ExampleCorp

Potential weak patterns:
ExampleCorp1
ExampleCorp2026
Welcome1
Winter2026
Summer2026
```

Blocking predictable patterns reduces the probability that one common password will work against many accounts.

---

# ⚠️ 18. Lockout Policy and Denial of Service

Password lockout policies themselves can introduce a security problem.

If the policy is extremely restrictive and requires an administrator to manually unlock accounts:

```text
Attacker
   ↓
Password spray
   ↓
Many failed attempts
   ↓
Many accounts locked
   ↓
Users unable to authenticate
   ↓
Administrative intervention
```

This can become a **denial-of-service condition**.

> [!important] CPTS perspective  
> A penetration tester must consider both sides of the security control:
> 
> - Too weak → password spraying becomes easier.
>     
> - Too restrictive → attackers may abuse lockout to disrupt users.
>     
> 
> The goal is not simply "maximum lockout."

---

# 🔎 19. Detecting Password Spraying

Password spraying produces recognizable patterns.

Potential indicators include:

- Many account lockouts within a short period
    
- Many authentication attempts
    
- Authentication attempts against valid users
    
- Authentication attempts against nonexistent users
    
- Large numbers of requests to one application or URL
    

### Generic detection pattern

```text
Many accounts
     ↓
Similar authentication attempt
     ↓
Short time window
     ↓
Repeated failures
     ↓
🚨 Possible password spray
```

---

# 🪟 20. Windows Event ID 4625

On a Domain Controller, many instances of:

**Event ID 4625 — An account failed to log on**

over a short period may indicate password spraying.

Conceptually:

```text
User A → failed login
User B → failed login
User C → failed login
User D → failed login
User E → failed login
       ↓
Short time window
       ↓
SIEM correlation
       ↓
🚨 Potential password spray
```

Organizations should configure detection rules that correlate many failures within a defined time interval rather than treating every failed login independently.

---

# 🔐 21. Kerberos Detection — Event ID 4771

A more sophisticated attacker may avoid SMB-based spraying and use Kerberos/LDAP-related authentication paths.

Defenders should also monitor:

**Event ID 4771 — Kerberos pre-authentication failed**

The source notes that Kerberos logging must be enabled for this detection approach.

### Detection model

```text
Kerberos authentication attempts
             ↓
Pre-authentication failures
             ↓
Event ID 4771
             ↓
SIEM correlation
             ↓
Potential password spraying
```

> [!tip] Defensive correlation  
> Monitoring only `4625` is not enough. Detection should account for the authentication protocol and service being targeted.

---

# 📊 22. Detection vs Attack Technique

|Attack Activity|Useful Defensive Signal|
|---|---|
|SMB/password authentication failures|Event ID `4625`|
|Kerberos pre-authentication failures|Event ID `4771`|
|Kerberos username enumeration|Event ID `4768` when appropriate logging is enabled|
|Many accounts targeted in a short period|SIEM correlation|
|Many account lockouts|Account-lockout monitoring|
|Many requests to one external application|Application/web logs|

The strongest detection comes from **correlating events**, not simply looking at one event in isolation.

---

# 🌍 23. External Password Spraying

Password spraying is also commonly used from the internet to obtain an initial foothold.

This is outside the primary scope of the internal-spraying section, but the same basic concept applies:

```text
Internet
   ↓
External authentication portal
   ↓
Many usernames
   +
One common password
   ↓
Potential valid account
```

Common external targets include:

- Microsoft 365
    
- Outlook Web Exchange
    
- Exchange Web Access
    
- Skype for Business
    
- Lync Server
    
- Microsoft Remote Desktop Services (RDS) portals
    
- Citrix portals using AD authentication
    
- VDI implementations using AD authentication, such as VMware Horizon
    
- VPN portals using AD authentication, including Citrix, SonicWall, OpenVPN, and Fortinet
    
- Custom web applications using AD authentication
    

---

# 🔄 24. Internal vs External Password Spraying

||Internal Spraying|External Spraying|
|---|---|---|
|Starting position|Inside the network|Internet|
|Typical target|Domain Controller / internal service|Public-facing authentication portal|
|User list source|AD enumeration / Kerberos|OSINT / previously obtained lists|
|Main goal|Domain credentials / lateral movement|Initial foothold|
|Important controls|Lockout policy, AD monitoring, segmentation|MFA, conditional access, external authentication monitoring|

### 🧠 Mental Model

```text
EXTERNAL
Internet
   ↓
Public authentication portal
   ↓
Valid credential
   ↓
Initial foothold
   ↓
Internal enumeration


INTERNAL
Existing foothold
   ↓
Domain user enumeration
   ↓
Password spray
   ↓
Higher-privilege credential
   ↓
Credentialed enumeration
   ↓
Lateral movement
```

---

# 🚀 25. Moving Deeper

Once several valid credential sets have been obtained, the engagement moves beyond basic password spraying.

The next objective is **credentialed domain enumeration**.

```text
                    PASSWORD SPRAY
                          ↓
                   VALID CREDENTIALS
                          ↓
                ┌─────────┴─────────┐
                ↓                   ↓
           Credential 1        Credential 2
                │                   │
                └─────────┬─────────┘
                          ↓
                 🔎 DOMAIN ENUMERATION
                          ↓
                Users / Groups / PCs
                          ↓
                Sessions / Shares
                          ↓
                 ACLs / Permissions
                          ↓
                🔄 Lateral Movement
                          ↓
                ⬆️ Privilege Escalation
                          ↓
                    🎯 END GOAL
```

> [!important] CPTS methodology  
> Password spraying is a **credential-acquisition phase**, not the final objective.
> 
> Once valid credentials are obtained, the question changes from:
> 
> **“Can I authenticate?”**
> 
> to:
> 
> **“What can this identity access, and where can it take me?”**

---

# 🧠 26. Complete Windows Spraying Mental Model

```text
             🪟 DOMAIN-JOINED WINDOWS HOST
                          │
                          ↓
                  Existing domain context
                          │
                          ↓
                DomainPasswordSpray
                          │
             ┌────────────┼────────────┐
             ↓            ↓            ↓
        👥 Users       🔐 Policy    ⚠️ Lockout
             │            │            │
             └────────────┼────────────┘
                          ↓
                  🧹 Filter targets
                          │
                          ↓
                  🎯 ONE PASSWORD
                          │
                          ↓
                 👥 MANY ACCOUNTS
                          │
                          ↓
                   Authentication
                          │
                 ┌────────┴────────┐
                 ↓                 ↓
               FAIL              SUCCESS
                                   │
                                   ↓
                            💾 Save creds
                                   │
                                   ↓
                       🔎 Credentialed enum
                                   │
                                   ↓
                       🔄 Lateral movement
                                   │
                                   ↓
                       ⬆️ Privilege escalation
```

---

# 🎯 27. Key Takeaways

- A **domain-joined Windows host** provides an excellent position from which to perform internal password spraying.
    
- `DomainPasswordSpray` can automatically:
    
    - Enumerate domain users
        
    - Query the password policy
        
    - Identify the smallest lockout threshold
        
    - Remove disabled accounts
        
    - Remove accounts close to lockout
        
    - Determine appropriate spray timing
        
    - Record successful credentials
        
- `-UserList` is unnecessary when the tool can generate the domain user list from the authenticated domain context.
    
- `-Password` specifies the password being sprayed.
    
- `-OutFile` saves successful results for later use.
    
- `Kerbrute` can also perform the username-enumeration and password-spraying workflow from Windows.
    
- **MFA** greatly reduces the risk of password spraying but should be deployed with other controls.
    
- **Least privilege** reduces the number of applications and systems accessible with ordinary domain credentials.
    
- Separate administrative accounts reduce the impact of compromised user accounts.
    
- **Network segmentation** can limit lateral movement after credential compromise.
    
- Strong password hygiene and password filtering reduce predictable spray targets.
    
- Excessively restrictive lockout policies can themselves create a **denial-of-service risk**.
    
- Event ID **4625** can help detect failed authentication patterns.
    
- Event ID **4771** can help detect Kerberos pre-authentication failures.
    
- Event ID **4768** can be relevant when detecting Kerberos TGT-request patterns.
    
- Detection should correlate authentication events across **users, time windows, services, and protocols**.
    
- External password spraying targets public-facing authentication portals such as Microsoft 365, VPNs, Citrix, RDS, VDI, and custom AD-backed applications.
    
- Once credentials are obtained, the engagement moves into **credentialed enumeration**, lateral movement, and privilege escalation.