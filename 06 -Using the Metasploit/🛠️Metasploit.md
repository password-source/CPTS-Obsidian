# 💥 Introduction to Metasploit

> [!info]+ 🎯 What Is Metasploit?  
> **Metasploit** is a **Ruby-based, modular penetration-testing platform** used to:
> 
> - 🔎 Enumerate systems and services
>     
> - 🧪 Test vulnerabilities
>     
> - 💥 Execute exploits
>     
> - 🐚 Deliver payloads
>     
> - 🔄 Manage sessions
>     
> - 🛠️ Develop and test exploit code
>     
> - ⚙️ Automate common penetration-testing tasks
>     
> 
> At its core, Metasploit provides a large collection of **modular tools and exploits** that can be combined with different payloads and targets.
> 
> ```text
> 🎯 TARGET
>     │
>     ▼
> 🔎 Identify Vulnerability
>     │
>     ▼
> 💥 Select Exploit
>     │
>     ▼
> 📦 Select Payload
>     │
>     ▼
> 🐚 Gain Access
>     │
>     ▼
> 🔄 Manage Session
> ```

---

# 🧩 1️⃣ The Metasploit Project

The **Metasploit Project** is the broader project surrounding Metasploit.

It has two major versions:

```text
                 💥 METASPLOIT PROJECT
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
     🆓 Metasploit Framework   💼 Metasploit Pro
       Open Source              Commercial
       Community Driven         Enterprise
       Free                     Paid
```

### 🆓 Metasploit Framework

The **Metasploit Framework (MSF)** is the open-source version commonly used by penetration testers.

It provides:

```text
💥 Exploits
📦 Payloads
🔎 Auxiliary modules
🔐 Post-exploitation
🛠️ Encoders
🛡️ Evasion modules
🧩 Plugins
⚙️ Tools
```

### 💼 Metasploit Pro

**Metasploit Pro** is the commercial, enterprise-oriented version.

It adds features designed for larger and more automated assessments.

---

# 💼 2️⃣ Metasploit Pro

Metasploit Pro includes additional capabilities such as:

```text
🧩 Task Chains
🌐 Web Interface
🎯 Vulnerability Validation
👥 Team Collaboration
📊 Reporting
🔐 Credential Management
🔄 Session Management
🧪 Exploitation Workflows
📡 VPN Pivoting
🕵️ Social Engineering
```

Other capabilities include:

|🧩 Infiltrate|📊 Collect Data|🛠️ Remediate|
|---|---|---|
|Manual Exploitation|Import & Scan Data|Bruteforce|
|Anti-virus Evasion|Discovery Scans|Task Chains|
|IPS/IDS Evasion|Meta-Modules|Exploitation Workflow|
|Proxy Pivot|Nexpose Integration|Session Rerun|
|Post-Exploitation||Task Replay|
|Session Cleanup||Project Sonar|
|Credential Reuse||Session Management|
|Social Engineering||Credential Management|
|Payload Generator||Team Collaboration|
|Quick Pen-testing||Web Interface|
|VPN Pivoting||Backup & Restore|
|Vulnerability Validation||Data Export|
|Phishing Wizard||Evidence Collection|
|Web App Testing||Reporting|
|Persistent Sessions||Tagging Data|

> [!tip]+ 🧠 CPTS Focus  
> For CPTS and hands-on HTB work, the **Metasploit Framework / `msfconsole`** is the part you will work with most frequently.

---

# 🖥️ 3️⃣ Metasploit Framework Console — `msfconsole`

The primary interface for the Metasploit Framework is:

```bash
msfconsole
```

It provides a centralized command-line interface for interacting with Metasploit.

```text
🖥️ Terminal
    │
    ▼
$ msfconsole
    │
    ▼
💥 Metasploit Framework
    │
    ├── Exploits
    ├── Payloads
    ├── Auxiliary
    ├── Post
    ├── Encoders
    ├── Evasion
    ├── Plugins
    └── Sessions
```

### ⭐ Why `msfconsole` Is Important

`msfconsole`:

- 🖥️ Provides a command-line interface to MSF
    
- 🧩 Gives access to most Metasploit functionality
    
- 🔎 Provides module searching
    
- ⚙️ Allows module configuration
    
- 💥 Executes exploits
    
- 📦 Configures payloads
    
- 🐚 Manages sessions
    
- 🔄 Manages background jobs
    
- ⌨️ Supports tab completion and command history
    
- 🖥️ Can execute external commands
    

> [!important]+ 🧠 Remember  
> `msfconsole` is essentially the **central control interface** for the Metasploit Framework.

---

# 🧩 4️⃣ The Metasploit Module Concept

One of the most important concepts in Metasploit is the **module**.

A module is a self-contained piece of functionality designed for a specific purpose.

Examples:

```text
💥 Exploit module
📡 Auxiliary module
📦 Payload module
🛠️ Encoder module
🛡️ Evasion module
🔄 Post-exploitation module
```

Instead of writing everything from scratch, we can select an existing module and configure it for our target.

```text
                💥 MODULE
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Target      Options    Payload
        │          │          │
        └──────────┼──────────┘
                   ▼
                🚀 Execute
```

---

# 🗂️ 5️⃣ Metasploit Module Categories

The main module directories are:

```bash
ls /usr/share/metasploit-framework/modules
```

Output:

```text
auxiliary
encoders
evasion
exploits
nops
payloads
post
```

### 🔎 What They Mean

|Module|Purpose|
|---|---|
|`auxiliary`|Scanning, enumeration, fuzzing and other supporting functions|
|`encoders`|Encode payloads in supported contexts|
|`evasion`|Modules intended to help evade certain security controls|
|`exploits`|Exploit vulnerabilities|
|`nops`|No-operation generators used with certain exploit techniques|
|`payloads`|Code executed after successful exploitation|
|`post`|Post-exploitation functionality|

### 🧠 Mental Model

```text
💥 EXPLOIT
   │
   │ "How do I exploit the vulnerability?"
   ▼
📦 PAYLOAD
   │
   │ "What should happen after exploitation?"
   ▼
🐚 SESSION
   │
   │ "What can I do after gaining access?"
   ▼
🔄 POST
```

---

# 💥 6️⃣ Exploit vs Payload

This distinction is **extremely important**.

> [!important]+ 💥 Exploit  
> The **exploit** takes advantage of a vulnerability in the target.
> 
> ```text
> Vulnerability
>      ↓
>    EXPLOIT
>      ↓
> Gain execution/access
> ```

> [!success]+ 📦 Payload  
> The **payload** is what gets executed after successful exploitation.
> 
> ```text
> EXPLOIT
>    ↓
> PAYLOAD
>    ↓
> Desired action
> ```
> 
> For example, a payload could provide a shell or execute another action.

### 🔑 Remember

```text
EXPLOIT = HOW DO I GET IN?

PAYLOAD = WHAT DO I RUN AFTER I GET IN?
```

This distinction will become increasingly important when working with Metasploit.

---

# 🧩 7️⃣ Plugins

Metasploit also supports **plugins**.

Plugins extend the functionality of `msfconsole`.

They can provide:

```text
⚙️ Additional functionality
🤖 Automation
🔗 Integrations
📊 Data management
🔎 Additional assessment capabilities
```

The plugins are stored under:

```bash
/usr/share/metasploit-framework/plugins/
```

Example:

```text
aggregator.rb
alias.rb
auto_add_route.rb
db_tracker.rb
nessus.rb
nexpose.rb
openvas.rb
sqlmap.rb
wmap.rb
```

> [!tip]+ 🧠 Remember  
> **Modules** provide specific penetration-testing functionality.
> 
> **Plugins** extend the functionality of the Metasploit console itself.

---

# 📜 8️⃣ Scripts

Metasploit also contains scripts.

Location:

```bash
/usr/share/metasploit-framework/scripts/
```

Example:

```text
meterpreter
ps
resource
shell
```

These contain scripts used for:

```text
🐚 Shell functionality
🔄 Meterpreter functionality
⚙️ Automation
📜 Resource scripts
```

---

# 🛠️ 9️⃣ Tools

Metasploit includes additional command-line utilities.

Location:

```bash
/usr/share/metasploit-framework/tools/
```

Example:

```text
context
docs
hardware
modules
payloads
dev
exploit
memdump
password
recon
```

These tools can be called directly from the command line and support different parts of the penetration-testing workflow.

---

# 📂 🔟 Metasploit File Structure

On the lab environment, the main Metasploit Framework directory is:

```text
/usr/share/metasploit-framework
```

A simplified view:

```text
/usr/share/metasploit-framework/
│
├── 📊 data/
├── 📚 documentation/
├── 📦 lib/
│
├── 🧩 modules/
│   ├── auxiliary/
│   ├── encoders/
│   ├── evasion/
│   ├── exploits/
│   ├── nops/
│   ├── payloads/
│   └── post/
│
├── 🔌 plugins/
│
├── 📜 scripts/
│
└── 🛠️ tools/
```

> [!important]+ 🧠 Why Know the File Structure?  
> Knowing where Metasploit stores its components becomes useful when:
> 
> - 📥 Importing a module
>     
> - 🛠️ Creating your own module
>     
> - 🔎 Locating existing modules
>     
> - 🐛 Troubleshooting Metasploit
>     
> - 📚 Reading module source code
>     
> - 🔄 Updating or managing the framework
>     

---

# 🧠 1️⃣1️⃣ Understanding the Architecture

It's important not to treat Metasploit as a magic button.

Instead, understand the relationship between its components:

```text
                         💥 METASPLOIT
                              │
                ┌─────────────┴─────────────┐
                │                           │
             MODULES                     CONSOLE
                │                           │
     ┌──────────┼──────────┐                │
     ▼          ▼          ▼                ▼
  Exploit    Auxiliary   Payload       msfconsole
     │          │          │                │
     └──────────┴──────────┴────────────────┘
                              │
                              ▼
                           TARGET
                              │
                              ▼
                           SESSION
                              │
                              ▼
                         POST-EXPLOITATION
```

---

# 🔎 1️⃣2️⃣ Metasploit in a CPTS Workflow

Metasploit should normally come **after enumeration and vulnerability identification**, not before.

A useful workflow is:

```text
1️⃣ 🔎 ENUMERATE
      │
      ▼
   Nmap / Other Tools
      │
      ▼
2️⃣ 🧠 IDENTIFY
      │
      ▼
   Service + Version
      │
      ▼
3️⃣ 🔍 RESEARCH
      │
      ▼
   CVE / PoC / Exploit
      │
      ▼
4️⃣ 💥 METASPLOIT
      │
      ├── search
      ├── use
      ├── info
      ├── options
      ├── set
      └── run/exploit
      │
      ▼
5️⃣ 🐚 SESSION
      │
      ▼
6️⃣ 🔄 POST-EXPLOITATION
```

> [!success]+ 🎯 Important CPTS Mindset  
> **Don't start with Metasploit and randomly search for exploits.**
> 
> Start with:
> 
> **"What did my enumeration tell me?"**
> 
> Then use Metasploit to investigate whether it has a suitable module.

---

# 🧭 1️⃣3️⃣ Basic Metasploit Navigation

Once inside `msfconsole`, a common workflow is:

```text
🔎 search
    ↓
🧩 identify module
    ↓
🎯 use module
    ↓
ℹ️ info
    ↓
⚙️ show options
    ↓
📝 set required options
    ↓
🚀 run / exploit
    ↓
🐚 session
```

Example:

```text
msf6 > search <keyword>

msf6 > use <module>

msf6 > info

msf6 > show options

msf6 > set <OPTION> <VALUE>

msf6 > run
```

> [!tip]+ 🧠 Your Metasploit Habit  
> Whenever you find multiple modules, **don't immediately pick one randomly**.
> 
> Check:
> 
> ```text
> 🎯 Target application
> 🔢 Version
> 💥 Vulnerability
> 🖥️ Target OS
> 📡 Service
> 📦 Payload compatibility
> ✅ Check support
> ```
> 
> This is the same methodology you were using during your earlier SMB/rConfig work.

---

# 📌 1️⃣4️⃣ Quick Reference

### 🖥️ Start Metasploit

```bash
msfconsole
```

### 📂 Main Framework Directory

```text
/usr/share/metasploit-framework
```

### 🧩 Modules

```bash
ls /usr/share/metasploit-framework/modules
```

```text
auxiliary
encoders
evasion
exploits
nops
payloads
post
```

### 🔌 Plugins

```bash
ls /usr/share/metasploit-framework/plugins/
```

### 📜 Scripts

```bash
ls /usr/share/metasploit-framework/scripts/
```

### 🛠️ Tools

```bash
ls /usr/share/metasploit-framework/tools/
```

---

# 🧠 1️⃣5️⃣ Metasploit Cheat Sheet

|Command|Purpose|
|---|---|
|`msfconsole`|Start Metasploit Framework console|
|`search <term>`|Search for modules|
|`use <module>`|Select a module|
|`info`|Display module information|
|`show options`|Display module options|
|`set <option> <value>`|Configure an option|
|`run`|Execute the selected module|
|`exploit`|Execute an exploit module|

### 🔑 Core Vocabulary

```text
💥 Exploit
= Takes advantage of a vulnerability

📦 Payload
= Code/action executed after exploitation

🧩 Module
= Individual piece of Metasploit functionality

🔌 Plugin
= Extends Metasploit functionality

🐚 Session
= Connection/interface obtained after successful exploitation

🖥️ msfconsole
= Main command-line interface
```

---

# 🎯 1️⃣6️⃣ Key Takeaways

- 💥 **Metasploit** is a modular penetration-testing and exploit-development platform.
    
- 🆓 **Metasploit Framework** is the open-source/community-driven version.
    
- 💼 **Metasploit Pro** is the commercial enterprise-oriented version.
    
- 🖥️ **`msfconsole`** is the primary interface you'll use with the Framework.
    
- 🧩 Metasploit is organized into different **module categories**.
    
- 💥 **Exploit modules** target vulnerabilities.
    
- 📦 **Payloads** define what happens after exploitation.
    
- 🔎 **Auxiliary modules** perform supporting tasks such as enumeration and scanning.
    
- 🔄 **Post modules** provide post-exploitation functionality.
    
- 🛠️ **Encoders, NOPs, and evasion modules** serve specialized purposes.
    
- 🔌 **Plugins** extend the functionality of Metasploit.
    
- 📜 **Scripts** provide additional automation and functionality.
    
- 🛠️ **Tools** provide supporting command-line utilities.
    
- 📂 The main Framework directory is `/usr/share/metasploit-framework`.
    
- 🧠 Understanding the architecture is more valuable than memorizing commands.
    

> [!success]+ 🧠 CPTS Mental Model
> 
> ```text
> 🔎 ENUMERATION
>       │
>       ▼
> 🧠 IDENTIFY SERVICE + VERSION
>       │
>       ▼
> 🔍 RESEARCH VULNERABILITY
>       │
>       ▼
> 💥 FIND METASPLOIT MODULE
>       │
>       ▼
> 🧩 SELECT EXPLOIT
>       │
>       ▼
> 📦 CONFIGURE PAYLOAD
>       │
>       ▼
> ⚙️ SET OPTIONS
>       │
>       ▼
> 🚀 EXECUTE
>       │
>       ▼
> 🐚 SESSION
>       │
>       ▼
> 🔄 POST-EXPLOITATION
> ```
> 
> ### ⭐ Golden Rule
> 
> **Enumeration tells you what to attack.  
> Metasploit helps you attack it.**