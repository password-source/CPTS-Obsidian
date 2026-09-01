# 🕵️ Meterpreter

---

## 🎯 What Is Meterpreter?

**Meterpreter** is an advanced Metasploit payload designed primarily for **post-exploitation**.

Instead of giving us only a basic OS shell, Meterpreter provides a powerful interactive environment with functionality for:

- 🔎 System enumeration
    
- 🔐 Credential collection
    
- ⬆️ Privilege escalation
    
- 🔄 Process migration
    
- 🧭 Network discovery & pivoting
    
- 🧩 Loading additional extensions
    
- 🗂️ File and process interaction
    
- 🔑 Token manipulation
    
- 🛠️ Running post-exploitation modules
    

🧠 **Mental Model:**

> **Exploit → Meterpreter → Enumerate → Escalate → Loot → Pivot**

Meterpreter is often described as the **"Swiss Army knife" of Metasploit** because it provides many post-exploitation capabilities from one session.

---

## 🧬 How Meterpreter Works

A typical Meterpreter connection involves several stages:

```text
Target
  │
  ▼
Initial Stager
  │
  ▼
Reflective DLL Loader
  │
  ▼
Meterpreter Core
  │
  ▼
Encrypted Communication
  │
  ▼
Meterpreter Extensions
  │
  ├── stdapi
  └── priv (when privileges allow)
```

### 🔹 1. Initial Stager

The target first executes a small **stager**.

Its purpose is to establish communication and load the larger Meterpreter payload.

Examples include:

```text
bind
reverse
findtag
passivex
```

### 🔹 2. Reflective DLL Loading

The stager loads the Meterpreter DLL using a **Reflective DLL Loader**.

This allows the payload to be loaded directly into memory.

### 🔹 3. Meterpreter Core

The Meterpreter core initializes and establishes communication with Metasploit.

Modern Meterpreter communication can use **AES encryption**.

### 🔹 4. Extensions

Meterpreter can dynamically load extensions.

Common examples:

```text
stdapi
priv
```

`stdapi` provides many standard interaction capabilities.

`priv` provides additional functionality when the session has sufficient privileges.

---

# 🥷 Meterpreter Design Goals

Meterpreter was designed around three major principles:

|Goal|Meaning|
|---|---|
|🥷 **Stealthy**|Minimize artifacts and detection|
|💪 **Powerful**|Provide extensive post-exploitation functionality|
|🧩 **Extensible**|Add functionality dynamically through extensions|

---

## 🥷 1. Stealthy

Historically, Meterpreter was designed to operate primarily from **memory** rather than installing a traditional executable on disk.

It can also perform **process migration**, moving the Meterpreter server into another process.

Conceptually:

```text
Initial Compromised Process
          │
          ▼
     Meterpreter
          │
          │ migrate
          ▼
   Another Process
```

Modern endpoint security can still detect Meterpreter through behavioral analysis, memory inspection, network monitoring, and other techniques.

⚠️ **Important:**

> "Fileless" or memory-resident does **not** mean undetectable.

---

## 🔐 Encrypted Communication

Meterpreter can communicate with Metasploit using encrypted traffic.

This helps protect the contents of the communication channel from simple network inspection.

```text
Target
  │
  │ Encrypted Meterpreter Traffic
  │
  ▼
Attacker / msfconsole
```

🧠 **Key Point:**

Encryption protects the **communication contents**. It does not automatically hide the fact that suspicious communication is occurring.

---

# 💪 2. Powerful

Meterpreter uses a **channelized communication system**.

This allows Meterpreter to create additional communication channels for things such as:

```text
Meterpreter
    │
    ├── Meterpreter commands
    │
    ├── OS shell
    │
    ├── File interaction
    │
    └── Other channels
```

This gives us functionality beyond a traditional shell.

---

# 🧩 3. Extensible

Meterpreter is modular.

Additional functionality can be loaded at runtime without rebuilding the entire payload.

This makes it possible to extend Meterpreter's capabilities during an engagement.

---

# 🖥️ Running Meterpreter

Meterpreter is selected as a **payload** when configuring an exploit.

For example:

```text
show payloads
```

Then select an appropriate Meterpreter payload for the target OS and connection type.

Example:

```text
windows/meterpreter/reverse_tcp
```

After successful exploitation:

```text
[*] Meterpreter session 1 opened
```

We receive:

```text
meterpreter >
```

This is our Meterpreter session.

---

# 📖 Meterpreter Help

Immediately after obtaining a Meterpreter session:

```text
meterpreter > help
```

Useful commands include:

|Command|Purpose|
|---|---|
|`help`|Display help|
|`background`|Background current session|
|`sessions`|Manage/switch sessions|
|`shell`|Open target OS shell|
|`ps`|List processes|
|`migrate`|Move Meterpreter into another process|
|`getuid`|Identify current user|
|`getsystem`|Attempt privilege escalation|
|`load`|Load an extension|
|`run`|Execute Meterpreter scripts/modules|
|`exit`|Terminate session|

🧠 **Don't try to memorize everything.**

Use:

```text
help
```

and:

```text
<command> -h
```

whenever you forget syntax.

---

# 🔄 Meterpreter Sessions

A Meterpreter session can be placed into the background:

```text
meterpreter > background
```

or:

```text
meterpreter > bg
```

We return to:

```text
msf6 >
```

The session continues running.

We can view sessions with:

```text
sessions
```

Example:

```text
Active sessions
===============

Id   Type
--   ----
1    meterpreter x86/windows
```

Interact with a specific session:

```text
sessions -i 1
```

Mental model:

```text
msfconsole
   │
   ├── Session 1 → Target A
   │
   ├── Session 2 → Target B
   │
   └── Session 3 → Target C
```

This allows Metasploit to manage multiple compromised systems simultaneously.

---

# 🔎 Meterpreter Post-Exploitation Workflow

Once a Meterpreter session is obtained, think systematically:

```text
             Meterpreter
                  │
                  ▼
             Identify User
                  │
                  ▼
          Enumerate Processes
                  │
                  ▼
        Determine Privileges
                  │
          ┌───────┴────────┐
          ▼                ▼
      Sufficient        Insufficient
      privileges        privileges
          │                │
          │                ▼
          │        Privilege Escalation
          │                │
          └────────┬───────┘
                   ▼
                Loot
                   │
                   ▼
              Enumerate
               Network
                   │
                   ▼
                Pivot
```

---

# 👤 Identifying the Current User

Use:

```text
meterpreter > getuid
```

Example:

```text
Server username: NT AUTHORITY\NETWORK SERVICE
```

This tells us **which account the Meterpreter process is currently running under**.

🧠 Always check this early.

You need to know:

> **"Who am I?"**

before deciding what to do next.

---

# 🔍 Process Enumeration

Use:

```text
meterpreter > ps
```

This displays running processes.

Example:

```text
PID    Name       User
---    ----       ----
1836   wmiprvse   NT AUTHORITY\NETWORK SERVICE
3552   w3wp       NT AUTHORITY\NETWORK SERVICE
```

Processes are useful because they can provide:

- 👤 Different security contexts
    
- 🔑 Different privileges
    
- 🧩 Potential migration targets
    
- 🔎 Information about applications/services
    

---

# 🔄 Process Migration

Meterpreter can migrate into another process:

```text
meterpreter > migrate <PID>
```

The general idea is:

```text
Meterpreter
    │
    │ migrate
    ▼
Target Process
    │
    ▼
New Process Context
```

Another related technique is token manipulation.

Example:

```text
meterpreter > steal_token <PID>
```

This attempts to impersonate the security token associated with another process.

Example:

```text
meterpreter > steal_token 1836

Stolen token with username:
NT AUTHORITY\NETWORK SERVICE
```

Then verify:

```text
meterpreter > getuid
```

⚠️ **Important distinction:**

**Process migration** and **token impersonation** are related but not identical.

```text
Migration
    ↓
Move Meterpreter into another process

Token Impersonation
    ↓
Use another process's security token
```

---

# ⬆️ Privilege Escalation

If:

```text
getuid
```

shows a low-privileged account, we should investigate privilege escalation.

One useful Metasploit post-exploitation module is:

```text
post/multi/recon/local_exploit_suggester
```

First background the session:

```text
meterpreter > bg
```

Search:

```text
msf6 > search local_exploit_suggester
```

Use it:

```text
msf6 > use post/multi/recon/local_exploit_suggester
```

Set the compromised session:

```text
msf6 post(...) > set SESSION 1
```

Run:

```text
msf6 post(...) > run
```

The module checks the target and suggests local exploits that **may** work.

Example output might contain:

```text
exploit/windows/local/ms14_058_track_popup_menu
exploit/windows/local/ms14_070_tcpip_ioctl
exploit/windows/local/ms15_051_client_copy_image
```

### 🧠 Critical Mental Model

The Local Exploit Suggester does **not** automatically mean:

> "This exploit will work."

Instead:

```text
Local Exploit Suggester
          │
          ▼
Potentially applicable exploits
          │
          ▼
Validate candidate
          │
          ▼
Attempt exploitation
          │
          ▼
Verify privileges
```

Always verify the result with:

```text
getuid
```

For example:

```text
Server username: NT AUTHORITY\SYSTEM
```

indicates that privilege escalation succeeded to **SYSTEM** on Windows.

---

# 🏆 SYSTEM Privileges

On Windows,:

```text
NT AUTHORITY\SYSTEM
```

represents a highly privileged local security context.

A successful escalation might therefore look like:

```text
NETWORK SERVICE
      │
      │ Local Privilege Escalation
      ▼
NT AUTHORITY\SYSTEM
```

Once SYSTEM-level access is obtained, substantially more post-exploitation functionality may become available.

---

# 🔐 Credential & Hash Collection

After obtaining sufficiently high privileges, Meterpreter can provide credential-related functionality.

For example:

```text
meterpreter > hashdump
```

This can retrieve password hashes from the Windows SAM database when the necessary privileges are available.

Conceptually:

```text
Windows SAM
    │
    ▼
Password Hashes
    │
    ├── Administrator
    ├── Guest
    └── Other local accounts
```

⚠️ These are **password hashes**, not plaintext passwords.

The hashes can potentially be used for further credential analysis in an authorized assessment.

---

# 🧠 SAM vs LSA Secrets

Two important Windows credential stores/functions encountered with Meterpreter are:

### SAM

The **Security Account Manager (SAM)** contains information associated with local Windows accounts, including password hashes.

Meterpreter:

```text
hashdump
```

### LSA Secrets

**LSA Secrets** can contain various sensitive information stored by Windows and installed services.

Meterpreter:

```text
lsa_dump_secrets
```

Example output may include service-related credentials or other secrets.

🧠 Mental model:

```text
              Windows Credentials
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
         SAM              LSA Secrets
          │                     │
     Account hashes       Stored secrets
```

---

# 🧭 Pivoting

Once we compromise a machine, it may provide access to networks that our attacking machine cannot directly reach.

This creates a potential pivot point:

```text
Attacker
   │
   ▼
Compromised Host
   │
   │ Internal Network
   ▼
Internal Host
   │
   ▼
Internal Services
```

Credentials, tokens, network information, and other loot obtained from the compromised system may also help with further authorized access.

🧠 **The compromised host can become a bridge into another network.**

---

# ⚠️ Important Operational Lesson

The example demonstrates an important concept:

> **Getting a Meterpreter session is not the end of exploitation. It is the beginning of post-exploitation.**

A basic workflow is:

```text
1️⃣ Initial Access
       ↓
2️⃣ Meterpreter Session
       ↓
3️⃣ getuid
       ↓
4️⃣ Process Enumeration
       ↓
5️⃣ Privilege Assessment
       ↓
6️⃣ Privilege Escalation
       ↓
7️⃣ Credential / Hash Collection
       ↓
8️⃣ Internal Enumeration
       ↓
9️⃣ Pivoting
```

---

# 🧠 CPTS Mental Model

Think of Meterpreter as a **post-exploitation toolbox**, not simply another shell.

```text
                 💻 TARGET
                    │
                    ▼
              🎯 EXPLOITATION
                    │
                    ▼
             🕵️ METERPRETER
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
     👤 USER     🔍 ENUM       🔄 PROCESS
        │           │            │
        ▼           ▼            ▼
   getuid /     ps / info    migrate /
   privileges                 tokens
        │
        ▼
      ⬆️ PRIVESC
        │
        ▼
   SYSTEM / HIGH PRIV
        │
        ├───────────────┐
        ▼               ▼
    🔐 CREDENTIALS    🧭 NETWORK
        │               │
        ▼               ▼
      LOOT            PIVOT
```

> [!important] **Core CPTS Takeaway**
> 
> When you obtain a Meterpreter session, don't immediately start randomly running commands.
> 
> Ask:
> 
> **Who am I? → What can I access? → What processes exist? → Can I escalate? → What information can I collect? → What networks can I reach?**

---

# 📌 Essential Commands

|Command|Purpose|
|---|---|
|`help`|Meterpreter help menu|
|`getuid`|Identify current user|
|`ps`|List processes|
|`migrate <PID>`|Migrate to another process|
|`steal_token <PID>`|Attempt token impersonation|
|`shell`|Open OS shell|
|`background` / `bg`|Background session|
|`sessions`|List active sessions|
|`sessions -i <ID>`|Interact with session|
|`load <extension>`|Load Meterpreter extension|
|`run <module>`|Run Meterpreter script/post module|
|`hashdump`|Dump local account hashes|
|`lsa_dump_secrets`|Dump LSA secrets when privileged|
|`exit`|Terminate session|

---

> [!tip] **Remember This**
> 
> **Meterpreter = Post-Exploitation Framework inside Metasploit**
> 
> The most important workflow to remember:
> 
> ```text
> Meterpreter
>      ↓
> getuid
>      ↓
> ps
>      ↓
> Enumerate
>      ↓
> Privilege Escalation
>      ↓
> SYSTEM
>      ↓
> Credential / Hash Collection
>      ↓
> Network Enumeration
>      ↓
> Pivot
> ```
> 
> Don't memorize every Meterpreter command.
> 
> **Understand what information you need at each stage and use `help` to find the command.**