# 🎯 1. The Concept of Attacks

> [!info]  
> **Core idea:** Instead of memorizing every attack as a completely separate technique, analyze vulnerabilities using the same four categories:
> 
> **`Source → Process → Privileges → Destination`**
> 
> This creates a reusable pattern for understanding attacks against different services such as **SSH, FTP, SMB, HTTP**, and others.

---

# 🧠 2. Why Use an Attack Concept?

Different vulnerabilities can look completely different on the surface, but they often follow the same underlying structure.

For example:

- HTTP injection
    
- Buffer overflow
    
- Malicious file processing
    
- Authentication attacks
    
- Remote code execution
    

Instead of asking:

> _"Which exploit do I use?"_

Think:

> **"What information enters the system, which process handles it, what privileges does that process have, and where does the result go?"**

This gives us a general methodology that can be applied across different services and vulnerabilities.

---

# 🔄 3. The Four-Part Attack Model

```text
                 ATTACK
                   │
                   ▼
             ┌───────────┐
             │  SOURCE   │
             │ Where does│
             │ input come │
             │ from?      │
             └─────┬─────┘
                   │
                   ▼
             ┌───────────┐
             │  PROCESS  │
             │ What code │
             │ handles it│
             └─────┬─────┘
                   │
                   ▼
             ┌───────────┐
             │ PRIVILEGES│
             │ Who/what  │
             │ executes  │
             │ it?       │
             └─────┬─────┘
                   │
                   ▼
             ┌───────────┐
             │DESTINATION│
             │ Where does│
             │ the result │
             │ go?       │
             └───────────┘
```

> [!important]  
> Memorize this as:
> 
> **SOURCE → PROCESS → PRIVILEGES → DESTINATION**
> 
> When analyzing an unknown vulnerability, walk through these four questions.

---

# 📥 4. Source — Where Does the Information Come From?

`Source` represents the information supplied to a process.

The important question is:

> **"What input can an attacker control or influence?"**

Information can reach a process through several mechanisms.

|**Information SourceDescription**|
|---|
|`Code`|
|`Libraries`|
|`Config`|
|`APIs`|
|`User Input`|

### 🔎 Think About Source Like This

```text
"What can I control?"
       │
       ├── HTTP parameter
       ├── HTTP header
       ├── File
       ├── API request
       ├── Configuration
       ├── User input
       └── Program/library data
```

> [!tip]  
> The protocol itself is not necessarily the important part.
> 
> **HTTP, SMB, FTP, SSH, etc. are delivery mechanisms.**
> 
> What matters is **what information eventually reaches the vulnerable process and how that information is interpreted.**

---

# 🧩 5. Process — What Handles the Input?

The `Process` is the component that receives and processes the information from the source.

The process determines **what happens to the input**.

Typical components include:

|**Process ComponentsDescription**|
|---|
|`PID`|
|`Input`|
|`Data processing`|
|`Variables`|
|`Logging`|

### 🧠 The Critical Question

Ask:

> **"How does the program interpret my input?"**

A vulnerability often exists because the program processes attacker-controlled information in an unsafe way.

```text
Attacker Input
      │
      ▼
   Process
      │
      ├── Expected interpretation
      │
      └── ❌ Unexpected interpretation
              │
              ▼
          Vulnerability
```

---

# 🔐 6. Privileges — Who Executes the Process?

`Privileges` determine what the process is allowed to do.

A vulnerability becomes much more dangerous when the vulnerable process has **high privileges**.

The source divides privileges into the following categories:

|**PrivilegesDescription**|
|---|
|`System`|
|`User`|
|`Groups`|
|`Policies`|
|`Rules`|

### ⚠️ Why Privileges Matter

The same vulnerability can have dramatically different impact depending on the process privileges.

```text
Vulnerable Process
       │
       ├── Runs as normal user
       │       ↓
       │   Limited impact
       │
       └── Runs as SYSTEM/root
               ↓
          Potentially total
          system compromise
```

> [!important]  
> **Always identify the privileges of the vulnerable process.**
> 
> Don't stop at:
> 
> `Can I execute code?`
> 
> Also ask:
> 
> `As whom does the code execute?`

---

# 🎯 7. Destination — Where Does the Result Go?

`Destination` represents the location or system component where the process's result is stored, forwarded, or otherwise used.

Destinations can be:

- **Local**
    
- **Network**
    

The result can modify local files/records or be forwarded to another service, system, or network location.

|**DestinationDescription**|
|---|
|`Local`|
|`Network`|

### 🌐 Think of Destination as:

```text
                 Process Result
                      │
             ┌────────┴────────┐
             ▼                 ▼
          LOCAL             NETWORK
             │                 │
       ┌─────┴─────┐      ┌────┴─────┐
       │           │      │          │
      File       Record   Server    Service
```

---

# ☠️ 8. Example — Log4j (CVE-2021-44228)

Log4j is an excellent example because the vulnerability can be mapped cleanly onto the four categories.

Log4j is a **library** used for application logging. In the example, an attacker manipulates an HTTP `User-Agent` header so that Log4j interprets the input as a **JNDI lookup** instead of simply logging it.

### 🧬 Log4j Attack Model

```text
Attacker
   │
   │ Malicious User-Agent
   ▼
┌──────────────────┐
│ Vulnerable App   │
│                  │
│ Log4j Library    │
└────────┬─────────┘
         │
         │ Misinterprets input
         ▼
    JNDI Lookup
         │
         │ executed with
         │ application privileges
         ▼
┌──────────────────┐
│ Attacker Server  │
│                  │
│ Malicious Java   │
│ Class             │
└────────┬─────────┘
         │
         ▼
       RCE
```

---

# 🧩 9. Log4j — Mapping the Four Categories

## 🚀 9.1 Initiating the Attack

|**StepLog4jConcept of Attacks - Category**|||
|---|---|---|
|`1.`|The attacker manipulates the user agent with a JNDI lookup command.|`Source`|
|`2.`|The process misinterprets the assigned user agent, leading to the execution of the command.|`Process`|
|`3.`|The JNDI lookup command is executed with administrator privileges due to logging permissions.|`Privileges`|
|`4.`|This JNDI lookup command points to the server created and prepared by the attacker, which contains a malicious Java class containing commands designed by the attacker.|`Destination`|

### 🧠 Mental Model

```text
SOURCE
Malicious HTTP User-Agent
        │
        ▼
PROCESS
Log4j interprets the input
        │
        ▼
PRIVILEGES
Application/Admin privileges
        │
        ▼
DESTINATION
Attacker-controlled server
```

---

# 💥 10. Log4j — Triggering Remote Code Execution

The attack then effectively begins another cycle.

The malicious Java class retrieved from the attacker's server becomes the **new Source** for the next processing stage.

|**StepLog4jConcept of Attacks - Category**|||
|---|---|---|
|`5.`|After the malicious Java class is retrieved from the attacker's server, it is used as a source for further actions in the following process.|`Source`|
|`6.`|Next, the malicious code of the Java class is read in, which in many cases has led to remote access to the system.|`Process`|
|`7.`|The malicious code is executed with administrator privileges due to logging permissions.|`Privileges`|
|`8.`|The code leads back over the network to the attacker with the functions that allow the attacker to control the system remotely.|`Destination`|

---

# 🔄 11. The Attack Can Form Multiple Cycles

One important concept is that the four categories can be applied repeatedly.

The destination of one stage can become the source for another stage.

```text
              ATTACK STAGE 1
                   │
                   ▼
              ┌────────┐
              │ SOURCE │
              └───┬────┘
                  ↓
              ┌─────────┐
              │ PROCESS │
              └───┬─────┘
                  ↓
              ┌────────────┐
              │ PRIVILEGES │
              └─────┬──────┘
                    ↓
              ┌────────────┐
              │DESTINATION │
              └─────┬──────┘
                    │
                    │ becomes input
                    ▼
              ┌────────┐
              │ SOURCE │
              └───┬────┘
                  ↓
              NEXT PROCESS
                  ↓
              NEXT PRIVILEGES
                  ↓
              NEXT DESTINATION
```

> [!tip]  
> This is particularly useful when analyzing multi-stage exploits.
> 
> Don't think of an exploit as necessarily being one action.
> 
> Think:
> 
> **What is the current source? → What processes it? → With what privileges? → Where does it lead?**

---

# 🧭 12. Universal Vulnerability Analysis Method

When you encounter a new service or vulnerability, use this checklist mentally:

```text
🔎 1. SOURCE
   ↓
   What input can I influence?

   ↓

🧩 2. PROCESS
   ↓
   Which application/library/function processes it?
   How is the input interpreted?

   ↓

🔐 3. PRIVILEGES
   ↓
   Which account/context executes the process?
   What can that context access?

   ↓

🎯 4. DESTINATION
   ↓
   Where does the processed result go?
   Local file? Database? Service? Remote host?

   ↓

💥 IMPACT
   ↓
   What can I achieve from this chain?
```

---

# 🛠️ 13. Applying the Model to Different Services

The model is deliberately **service-independent**.

For example:

```text
SSH
 │
 ├── Source      → credentials / user input
 ├── Process     → SSH daemon
 ├── Privileges  → service/user context
 └── Destination → local/remote system

FTP
 │
 ├── Source      → commands / uploaded files
 ├── Process     → FTP server
 ├── Privileges  → FTP service account
 └── Destination → filesystem/network

SMB
 │
 ├── Source      → SMB requests / files
 ├── Process     → SMB service
 ├── Privileges  → service/security context
 └── Destination → files / shares / remote system

HTTP
 │
 ├── Source      → request parameters / headers / files
 ├── Process     → web server/application/library
 ├── Privileges  → application account
 └── Destination → database / filesystem / network
```

> [!important]  
> The **specific attack technique changes**, but the **analytical framework remains the same**.

---

# 🧠 14. The Most Important Questions to Ask

|Category|Question|
|---|---|
|📥 **Source**|What information can I control or influence?|
|⚙️ **Process**|What program, function, library, or service processes it?|
|🔐 **Privileges**|Under which user, group, policy, or security context does it execute?|
|🎯 **Destination**|Where is the result stored, forwarded, or executed?|
|💥 **Impact**|What does the complete chain allow me to achieve?|

---

# 🧠 15. CPTS Mental Model

> [!success]
> 
> ### Remember: **SPPD**
> 
> **S — Source**  
> _Where does the information come from?_
> 
> **P — Process**  
> _What handles/interprets it?_
> 
> **P — Privileges**  
> _With what permissions does it run?_
> 
> **D — Destination**  
> _Where does the result go?_

```text
             ┌───────────────┐
             │    SOURCE     │
             │ "What enters?"│
             └───────┬───────┘
                     ↓
             ┌───────────────┐
             │    PROCESS    │
             │ "What handles │
             │     it?"      │
             └───────┬───────┘
                     ↓
             ┌───────────────┐
             │  PRIVILEGES   │
             │ "Who executes │
             │     it?"      │
             └───────┬───────┘
                     ↓
             ┌───────────────┐
             │  DESTINATION  │
             │ "Where does   │
             │ it lead?"     │
             └───────┬───────┘
                     ↓
                  💥 IMPACT
```

---

# 🏆 16. Key Takeaways

- **`Source → Process → Privileges → Destination`** is a generalized model for understanding vulnerabilities.
    
- **Source** identifies where the attacker-controlled or relevant information originates.
    
- **Process** identifies the application, function, library, or service that handles the information.
    
- **Privileges** determine the permissions available to the vulnerable process.
    
- **Destination** identifies where the resulting data, request, or execution goes.
    
- The same framework can be applied to many different services and vulnerabilities.
    
- Vulnerabilities often occur because a process **interprets input differently from what was intended**.
    
- High process privileges can significantly increase vulnerability impact.
    
- Multi-stage attacks can be understood as **repeating the same Source → Process → Privileges → Destination pattern**.
    
- Use the model not only for exploitation, but also for **source-code analysis, exploit development, debugging, and understanding attack impact**.
    

> [!important]  
> **CPTS Exam Mindset:**
> 
> Don't memorize only _"this vulnerability uses this exploit."_
> 
> Learn to recognize:
> 
> **📥 What can I control? → ⚙️ What processes it? → 🔐 What privileges does it have? → 🎯 Where can I make it go?**
> 
> Once you can answer those four questions, you can reason about unfamiliar vulnerabilities instead of relying entirely on memorized attack recipes.