# 🔎 1. Finding Sensitive Information

> [!info]  
> **Core idea:** During a penetration test, think like a **detective**.
> 
> Every piece of information discovered during enumeration can become a clue that leads to another account, service, credential, or ultimately **Remote Code Execution (RCE)**.
> 
> Information that appears useless in isolation may become extremely valuable when combined with other discoveries.

---

# 🧠 2. The Detective Mindset

When attacking services, don't only look for an immediate exploit.

Instead:

```text
🔎 Discover Information
        ↓
🧩 Understand Its Meaning
        ↓
🔗 Correlate It With Other Services
        ↓
🔑 Try It Where Relevant
        ↓
📂 Obtain More Information
        ↓
💥 Reach the Objective
```

> [!important]  
> **Never dismiss information simply because it doesn't immediately give you access.**
> 
> A username, hostname, email address, filename, IP address, or configuration fragment may become the missing piece later.

---

# 🎯 3. Example Attack Chain

The source provides an excellent example of how seemingly insignificant information can lead to RCE.

### 🌐 Initial Target

The engagement targets:

- 📧 Email
    
- 📁 FTP
    
- 🗄️ Databases
    
- 💾 Storage
    

**Goal:** Obtain **RCE** on any of the services.

---

## 🔍 Step 1 — Enumerate Services

Anonymous access is tested against all services.

```text
Email   → ❌ No anonymous access
FTP     → ✅ Anonymous access
Database→ ❌
Storage → ❌
```

FTP allows anonymous access.

---

## 📂 Step 2 — Investigate FTP

An empty file is discovered:

```text
johnsmith
```

At first glance, this seems insignificant.

The tester makes a reasonable hypothesis:

```text
Username = johnsmith
Password = johnsmith
```

The FTP login fails.

> [!warning]  
> **Failure does not mean the information is useless.**
> 
> `johnsmith` is still a potentially valid username and should be retained as a clue.

---

## 📧 Step 3 — Reuse the Information Against Email

The discovered username is tested against the email service.

This time:

```text
johnsmith
   ↓
Email authentication
   ↓
✅ Successful login
```

Now the tester has access to John's mailbox.

---

## 🔎 Step 4 — Search the Mailbox

Since email frequently contains operational information, search for messages containing:

```text
password
```

Multiple results are discovered.

One email contains John's credentials for the **MSSQL database**.

```text
Email
  ↓
Search "password"
  ↓
MSSQL credentials
```

---

## 🗄️ Step 5 — Access MSSQL

The discovered credentials are used to access the database.

The database provides built-in functionality that allows commands to be executed.

```text
MSSQL Credentials
       ↓
   Database Access
       ↓
Command Execution
       ↓
      RCE 💥
```

The engagement goal is achieved.

---

# 🔗 4. The Complete Attack Chain

```text
             🔎 FTP Enumeration
                    │
                    ▼
          Anonymous FTP Access
                    │
                    ▼
          Empty file: johnsmith
                    │
                    ▼
       Treat "johnsmith" as a clue
                    │
                    ▼
          Try against Email
                    │
                    ▼
             📧 Email Access
                    │
                    ▼
        Search emails for "password"
                    │
                    ▼
        🗝️ MSSQL Credentials Found
                    │
                    ▼
             🗄️ MSSQL Access
                    │
                    ▼
        Built-in Command Execution
                    │
                    ▼
               💥 RCE
```

> [!success]
> 
> ### The lesson
> 
> **One service can provide the information needed to compromise another service.**
> 
> Think horizontally across the entire target environment rather than treating every service as an isolated target.

---

# 🧩 5. Why `johnsmith` Was Important

The original discovery was:

```text
johnsmith
```

It did **not** immediately provide:

- FTP access
    
- A password
    
- RCE
    
- Database access
    

But it provided a **username candidate**.

That username eventually led to:

```text
johnsmith
    ↓
Email account
    ↓
MSSQL credentials
    ↓
Database access
    ↓
Command execution
    ↓
RCE
```

### 🧠 Mental Model

```text
"Useless" Information
        │
        │ correlate
        ▼
Useful Information
        │
        │ reuse
        ▼
Credentials
        │
        │ authenticate
        ▼
New Access
        │
        │ enumerate
        ▼
More Information
        │
        ▼
💥 Compromise
```

---

# 📋 6. Types of Sensitive Information

Sensitive information can include, but is not limited to:

```text
👤 Usernames
📧 Email Addresses
🔑 Passwords
🌐 DNS Records
📍 IP Addresses
💻 Source Code
⚙️ Configuration Files
🧑 PII (Personally Identifiable Information)
```

### 🎯 Why Each Can Matter

|Information|Potential Value|
|---|---|
|👤 **Username**|Can identify valid accounts and become a credential-testing target|
|📧 **Email Address**|May reveal usernames, accounts, organizational structure, or useful correspondence|
|🔑 **Password**|Direct authentication to one or more services|
|🌐 **DNS Record**|Reveals hosts, naming conventions, and infrastructure|
|📍 **IP Address**|Identifies systems/services to investigate|
|💻 **Source Code**|May reveal vulnerabilities, credentials, API keys, or application logic|
|⚙️ **Configuration File**|May contain connection details, usernames, passwords, keys, or service settings|
|🧑 **PII**|Can reveal sensitive personal/organizational information and provide additional attack context|

> [!important]  
> The value of information depends on **context**.
> 
> A username alone may not be enough.
> 
> A username + email + password + service name can be extremely valuable.

---

# 🌐 7. Think Across Services

A critical pentesting mindset is to **correlate discoveries between services**.

Don't use this approach:

```text
FTP → attack FTP
Email → attack Email
MSSQL → attack MSSQL
Storage → attack Storage
```

Instead:

```text
FTP
 │
 └── Information
       ↓
     Email
       │
       └── Credentials
             ↓
           MSSQL
             │
             └── Command Execution
                    ↓
                   RCE
```

### 🧠 Cross-Service Correlation

```text
SERVICE A
   │
   │ leaks information
   ▼
SERVICE B
   │
   │ provides credentials
   ▼
SERVICE C
   │
   │ exposes functionality
   ▼
OBJECTIVE
```

> [!tip]  
> Whenever you discover something, ask:
> 
> **"Where else could this information be useful?"**

---

# 🔎 8. Understanding What to Look For

Not every target will expose the same information.

Every target is unique.

Before deciding what information is valuable, understand:

- The target's **business model**
    
- Its **purpose**
    
- Its **processes**
    
- Its **procedures**
    
- How its services interact
    
- What information is important to the organization
    

```text
Target
  │
  ├── 🏢 Business Model
  ├── ⚙️ Processes
  ├── 📋 Procedures
  ├── 🎯 Purpose
  └── 🌐 Services
          │
          ▼
   What information
   would be valuable?
```

> [!important]  
> **There is no universal "sensitive file" or "sensitive keyword" that works against every target.**
> 
> You need to understand what the organization actually does and determine what information would help you achieve your objective.

---

# 🧠 9. Two Key Elements

The source identifies **two fundamental requirements** for finding sensitive information:

## 1️⃣ Understand the Service

You need to understand:

- What the service does
    
- How it works
    
- What data it stores
    
- What users interact with
    
- Where information is likely to appear
    
- How the service communicates with other services
    

```text
Understand Service
        ↓
Understand Data Flow
        ↓
Know Where Information May Exist
```

---

## 2️⃣ Know What You're Looking For

Once you understand the service, determine what information is relevant to the engagement.

Examples:

```text
🔑 Credentials
👤 Usernames
📧 Email addresses
🗄️ Database connection strings
🌐 Internal hosts
⚙️ Configuration
💻 Source code
🧑 PII
```

The important distinction is:

> **Don't search blindly. Search with a hypothesis.**

---

# 🧭 10. Sensitive Information Discovery Methodology

```text
                 🎯 TARGET
                    │
                    ▼
          🔎 Enumerate Services
                    │
                    ▼
        🧠 Understand Each Service
                    │
                    ▼
          📋 Identify What Matters
                    │
                    ▼
          🔍 Search for Information
                    │
                    ▼
          🧩 Correlate Discoveries
                    │
                    ▼
       🔐 Test Information Against
          Relevant Services
                    │
                    ▼
             📂 Gain Access
                    │
                    ▼
        🔎 Enumerate Again
                    │
                    ▼
          🔗 Build New Attack Path
                    │
                    ▼
               💥 OBJECTIVE
```

---

# ♻️ 11. Enumeration Is Iterative

One of the most important concepts is that enumeration does **not** necessarily end after the initial scan.

New information can reveal new targets.

```text
Initial Enumeration
        ↓
Information Found
        ↓
New Username / Host / Credential
        ↓
New Service Access
        ↓
New Information
        ↓
More Enumeration
        ↓
New Attack Path
```

> [!important]  
> **Every new piece of information can change what you should enumerate next.**
> 
> This is why pentesting is an iterative process rather than a simple linear checklist.

---

# 🧰 12. Services Covered in This Topic

The source focuses on three major areas where useful information can commonly be discovered:

### 📁 File Shares

Potential sources of:

- Credentials
    
- Configuration files
    
- Documents
    
- Usernames
    
- Scripts
    
- Sensitive organizational data
    

### 📧 Email

Potential sources of:

- Passwords
    
- Credentials
    
- Internal communication
    
- Service information
    
- Usernames
    
- Database connection details
    

### 🗄️ Databases

Potential sources of:

- Application data
    
- User information
    
- Credentials
    
- Configuration information
    
- Potential command-execution functionality
    

```text
📁 File Shares
      │
      ▼
   Information
      │
      ├──────────────┐
      ▼              ▼
   📧 Email      🗄️ Database
      │              │
      ▼              ▼
 Credentials     Sensitive Data
      │              │
      └──────┬───────┘
             ▼
        💥 Attack Path
```

---

# ⚡ 13. CPTS "Finding Sensitive Information" Cheat Sheet

|Step|Question|Goal|
|---|---|---|
|1️⃣|🔎 What services exist?|Build the initial attack surface|
|2️⃣|👤 Can I access them anonymously?|Find unauthenticated information|
|3️⃣|📂 What files/data can I access?|Discover clues|
|4️⃣|🧩 What does each discovery mean?|Convert raw data into useful information|
|5️⃣|🔗 Where else can I use it?|Correlate information between services|
|6️⃣|🔑 Does it provide credentials?|Obtain authenticated access|
|7️⃣|🌐 Does it reveal another host/service?|Expand the attack surface|
|8️⃣|🔎 What can the new access reveal?|Continue enumeration|
|9️⃣|💥 Can the chain reach the objective?|Achieve RCE / access / other engagement goal|

---

# 🧠 14. The Golden Rule

> [!success]
> 
> ### **Treat Every Discovery as a Clue**
> 
> Don't immediately classify information as:
> 
> ❌ "Useless"  
> ❌ "Just a username"  
> ❌ "Just an empty file"  
> ❌ "Just an email address"
> 
> Instead ask:
> 
> **"What can this lead to?"**

```text
             🔎 DISCOVERY
                  │
                  ▼
           "What is this?"
                  │
                  ▼
           "What does it tell me?"
                  │
                  ▼
       "Where else can I use it?"
                  │
                  ▼
       "What new access can it give?"
                  │
                  ▼
       "What can I enumerate now?"
                  │
                  ▼
            💥 NEW PATH
```

---

# 🏆 15. Key Takeaways

> [!important]
> 
> - **Every piece of information can be valuable.**
>     
> - A discovery that does not immediately provide access may still unlock another service.
>     
> - Always **correlate information across services**.
>     
> - Understand the **target, business model, processes, and purpose** before deciding what information is valuable.
>     
> - Two key skills are:
>     
>     1. **Understand how the service works.**
>         
>     2. **Know what information you are looking for.**
>         
> - Enumeration should be **iterative**: new information creates new enumeration opportunities.
>     
> - Think like a detective: **collect → interpret → correlate → reuse → enumerate again**.
>     
> - The example demonstrates a complete chain:
>     
> 
> ```text
> FTP
>  ↓
> Username
>  ↓
> Email
>  ↓
> MSSQL Credentials
>  ↓
> MSSQL
>  ↓
> Command Execution
>  ↓
> RCE 💥
> ```
> 
> **The biggest mindset shift:**  
> **Don't hunt only for vulnerabilities. Hunt for information that can lead you to vulnerabilities, credentials, access, and ultimately your objective.**