# 🎯 Sessions

---

## 🧠 What Are Sessions?

A **Session** is an active communication channel between Metasploit and a target system created after successful exploitation or another module establishes communication.

Sessions allow `msfconsole` to:

- 🖥️ Maintain access to compromised systems
    
- 🔄 Switch between multiple compromised hosts
    
- 🧩 Run additional modules against an existing session
    
- 🌐 Perform post-exploitation activities
    
- 🔀 Keep sessions active while working with other Metasploit modules
    

### Mental Model

```text
Exploit
   │
   ▼
Target Compromised
   │
   ▼
Session Created
   │
   ├──► Interact with Session
   │
   ├──► Background Session
   │
   └──► Run Post-Exploitation Modules
```

> [!important] 🔑 Key Concept  
> **A session represents an active communication channel with a target.**

---

# 📋 Listing Sessions

Use:

```text
sessions
```

Example:

```text
msf6 > sessions

Active sessions
===============

  Id  Name  Type                     Information                 Connection
  --  ----  ----                     -----------                 ----------
  1         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ MS01  10.10.10.129:443 -> 10.10.10.205:50501
```

The session ID is:

```text
1
```

This ID is used to interact with the session.

---

# 🎮 Interacting With a Session

Use:

```text
sessions -i <ID>
```

Example:

```text
msf6 > sessions -i 1

[*] Starting interaction with 1...

meterpreter >
```

Now you are interacting directly with the Meterpreter session.

### Mental Model

```text
msfconsole
    │
    ├── Session 1 → Windows machine
    │
    ├── Session 2 → Linux machine
    │
    └── Session 3 → Another Windows machine
```

You can move between them:

```text
sessions -i 1
sessions -i 2
sessions -i 3
```

---

# ⏸️ Backgrounding a Session

You don't have to terminate a session when you want to return to `msfconsole`.

For Meterpreter, you can use:

```text
background
```

or:

```text
CTRL + Z
```

After backgrounding:

```text
meterpreter >
       │
       ▼
   background
       │
       ▼
msf6 >
```

The session **continues running** in the background.

You can verify it with:

```text
sessions
```

> [!warning] ⚠️ Background ≠ Terminate
> 
> Backgrounding a session does **not** close the session.
> 
> ```text
> background
>     ↓
> Session remains active
>     ↓
> You return to msfconsole
> ```
> 
> The session can still die later if the payload or communication channel fails.

---

# 🔄 Using Another Module Against an Existing Session

One of the biggest advantages of sessions is that you can use an existing compromised system as the starting point for further actions.

Typical workflow:

```text
1. Exploit target
       ↓
2. Obtain Session
       ↓
3. Background Session
       ↓
4. Search for another module
       ↓
5. Select the existing Session
       ↓
6. Run Post-Exploitation module
```

Many useful modules are located under:

```text
post/
```

Examples include modules for:

- 🔑 Credential gathering
    
- 🔎 Local enumeration
    
- 🧭 Network discovery
    
- ⬆️ Privilege escalation assistance
    
- 🖥️ System information gathering
    

The selected module may provide a `SESSION` option:

```text
show options
```

Then:

```text
set SESSION 1
```

This tells Metasploit:

> **Run this module against Session 1.**

---

# ⚙️ Sessions vs Jobs

This distinction is extremely important.

### 🟢 Session

A **Session** is primarily an active communication channel with a target.

```text
Exploit
   ↓
Target
   ↓
Session
   ↓
Interact with target
```

### 🔵 Job

A **Job** is a Metasploit task running in the background.

```text
Module
   ↓
Job
   ↓
Runs in background
```

### Comparison

||🟢 Session|🔵 Job|
|---|---|---|
|Represents|Communication channel|Background task|
|Connected to target?|Usually ✅|Not necessarily|
|Can interact with target?|✅|Depends on task|
|Can run in background?|✅|✅|
|Example|Meterpreter session|Reverse TCP handler|
|Main command|`sessions`|`jobs`|

🧠 **Easy way to remember:**

> **Session = connection**  
> **Job = background task**

---

# ⚙️ Jobs

Jobs allow Metasploit tasks to continue running in the background.

This is particularly useful for things such as:

- 🎧 Reverse handlers
    
- 🔄 Long-running modules
    
- 🌐 Listeners
    
- ⏳ Tasks that shouldn't block your current `msfconsole` workflow
    

---

# 📋 Viewing Jobs

Use:

```text
jobs -l
```

Example:

```text
msf6 > jobs -l

Jobs
====

 Id  Name                    Payload
 --  ----                    -------
 0   Exploit: multi/handler  generic/shell_reverse_tcp
```

The job ID here is:

```text
0
```

---

# 🚀 Running an Exploit as a Job

Normally:

```text
exploit
```

runs the exploit in the current context.

Adding:

```text
-j
```

runs it as a background job.

```text
exploit -j
```

Example:

```text
msf6 exploit(multi/handler) > exploit -j

[*] Exploit running as background job 0.
[*] Exploit completed, but no session was created.

[*] Started reverse TCP handler on 10.10.14.34:4444
```

The important part is:

```text
exploit -j
     │
     ▼
Background Job
     │
     ▼
Metasploit remains available
```

---

# 🛑 Managing Jobs

### List jobs

```text
jobs -l
```

### Kill a specific job

```text
jobs -k <job_id>
```

Example:

```text
jobs -k 0
```

### Kill all jobs

```text
jobs -K
```

### Get detailed job information

```text
jobs -i <job_id>
```

### Show help

```text
jobs -h
```

---

# 🚨 Why Jobs Matter

Suppose you start a reverse handler:

```text
exploit
```

and it occupies a listening port such as:

```text
4444
```

If you need to use that port for another handler/module, simply stopping your current interaction may not necessarily clean up the background task that owns the port.

Instead:

```text
jobs -l
```

Find the job:

```text
0   Exploit: multi/handler
```

Then terminate it:

```text
jobs -k 0
```

Now the port can be reused.

> [!tip] 💡 Practical Rule
> 
> If something is still running in the background and you need to stop it, check:
> 
> ```text
> jobs -l
> ```
> 
> before assuming the process is gone.

---

# 🧩 Session + Job Workflow

A common Metasploit workflow can look like this:

```text
                  EXPLOIT
                     │
                     ▼
              ┌─────────────┐
              │   TARGET    │
              └──────┬──────┘
                     │
                     ▼
                 SESSION 1
                     │
              ┌──────┴──────┐
              │             │
          interact       background
              │             │
              │             ▼
              │         msfconsole
              │             │
              │             ▼
              │       Use another module
              │             │
              │             ▼
              │       SESSION 1
              │
              ▼
          Target Access
```

Meanwhile, independent background tasks can exist:

```text
msfconsole
    │
    ├── Session 1
    ├── Session 2
    ├── Session 3
    │
    └── Jobs
         ├── Job 0 → Listener
         └── Job 1 → Background task
```

---

# 🧠 The Big Picture

Metasploit can maintain **multiple sessions and jobs simultaneously**.

```text
                         msfconsole
                             │
              ┌──────────────┼──────────────┐
              │              │              │
          SESSION 1      SESSION 2      SESSION 3
              │              │              │
          Windows         Linux          Windows
              │
              ▼
        Post-Exploitation


                         JOBS
                          │
             ┌────────────┼────────────┐
             │            │            │
           Job 0        Job 1        Job 2
          Handler       Scanner       Task
```

This is what allows Metasploit to operate efficiently during larger assessments without constantly restarting modules or losing existing connections.

---

> [!important] 🎯 CPTS Key Takeaways
> 
> ### Sessions
> 
> ```text
> sessions
> ```
> 
> List active sessions.
> 
> ```text
> sessions -i 1
> ```
> 
> Interact with Session 1.
> 
> ```text
> background
> ```
> 
> Background the current Meterpreter session while keeping it alive.
> 
> ---
> 
> ### Jobs
> 
> ```text
> jobs -l
> ```
> 
> List running background jobs.
> 
> ```text
> exploit -j
> ```
> 
> Run an exploit/module as a background job.
> 
> ```text
> jobs -k 0
> ```
> 
> Kill Job 0.
> 
> ```text
> jobs -K
> ```
> 
> Kill all jobs.
> 
> ---
> 
> ### ⭐ Remember This
> 
> ```text
> SESSION
>    ↓
> Active communication channel
>    ↓
> "I have access to the target"
> 
> JOB
>    ↓
> Background Metasploit task
>    ↓
> "Something is running in the background"
> ```
> 
> **Session = connection to target**  
> **Job = background task**