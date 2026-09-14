# 🕸️ Antak Webshell

> [!info]+ 🎯 What Is Antak?  
> **Antak** is an **ASPX web shell** built with ASP.NET and PowerShell.
> 
> It is part of the **Nishang** penetration-testing framework and allows us to interact with a Windows target through a web browser.
> 
> ```text
> 🌐 Browser
>      │
>      ▼
> 🕸️ Antak ASPX Web Shell
>      │
>      ▼
> 💙 PowerShell
>      │
>      ▼
> 🪟 Windows Operating System
> ```
> 
> Antak is particularly useful when:
> 
> - 🎯 The target is a Windows server.
>     
> - 🌐 The web server supports ASP.NET/ASPX.
>     
> - 📤 We have a way to upload an ASPX file.
>     
> - 💻 We want PowerShell-based command execution.
>     

---

# 📚 1️⃣ Learning ASPX Web Shells

Before working with Antak, it is useful to understand **how ASPX web shells work**.

One excellent resource for studying real-world penetration-testing techniques is **IPPSEC's** collection of walkthroughs.

[IPPSEC Rocks](https://ippsec.rocks/?utm_source=chatgpt.com)

The site allows you to search for concepts such as:

```text
ASPX
Web Shell
PowerShell
File Upload
RCE
```

It then identifies relevant sections of IPPSEC's videos where those concepts are demonstrated.

> [!tip]+ 🧠 Why This Is Useful for CPTS  
> Reading gives you the **concept**.
> 
> Watching a real engagement gives you the **workflow**.
> 
> Hands-on practice gives you the **skill**.
> 
> ```text
> 📖 Read
>    ↓
> 🎥 Watch
>    ↓
> 🧪 Practice
>    ↓
> 🧠 Understand
>    ↓
> 💻 Apply
> ```

---

# 🪟 2️⃣ What Is ASPX?

**ASPX** stands for **Active Server Page Extended**.

It is a file type used by Microsoft's **ASP.NET** framework.

An ASPX page is processed by the web server and can generate HTML that is returned to the browser.

Conceptually:

```text
🌐 Browser
    │
    │ HTTP Request
    ▼
🪟 ASP.NET Web Server
    │
    │ Processes .aspx
    ▼
📄 ASPX Code
    │
    ▼
🌐 HTML Response
    │
    ▼
🖥️ Browser
```

However, because ASPX code executes on the server, a malicious ASPX file can potentially interact with the underlying operating system.

That is what makes an **ASPX web shell** useful during a penetration test.

---

# 🧠 3️⃣ ASPX Web Shell Concept

The important concept is:

```text
📄 ASPX File
      │
      ▼
⚙️ ASP.NET Processes It
      │
      ▼
💻 Server-Side Code Executes
      │
      ▼
🪟 Windows OS
```

A specially crafted ASPX file can provide a web interface that allows us to execute commands on the underlying Windows host.

Therefore:

> [!important]+ 💡 Core Concept  
> **ASPX is not inherently malicious.**
> 
> ASPX is a legitimate web technology.
> 
> The security issue arises when an attacker can **upload or otherwise place an executable ASPX file on a server and then access it**.
> 
> ```text
> 📤 Upload ASPX
>       +
> ⚙️ Server executes ASPX
>       ↓
> 💻 Server-side code execution
>       ↓
> 🕸️ Web Shell
> ```

---

# 🕸️ 4️⃣ What Is Antak?

**Antak** is an ASP.NET web shell included in the **Nishang** project.

[Nishang on GitHub](https://github.com/samratashok/nishang?utm_source=chatgpt.com)

Nishang is an offensive PowerShell framework containing tools and scripts useful throughout a penetration test.

Antak specifically focuses on providing a **PowerShell-based web shell for Windows**.

```text
🧰 Nishang
    │
    └── 🕸️ Antak
           │
           ├── ASP.NET / ASPX
           │
           └── PowerShell
```

> [!success]+ 🎯 Why Antak?  
> Antak is useful because it combines:
> 
> ```text
> 🌐 Web Interface
>       +
> 📄 ASPX
>       +
> 💙 PowerShell
>       ↓
> 🪟 Windows Command Execution
> ```

---

# 📁 5️⃣ Locating Antak

On Kali/Parrot, Antak can be found at:

```text
/usr/share/nishang/Antak-WebShell
```

We can inspect the directory with:

```bash
ls /usr/share/nishang/Antak-WebShell
```

Example:

```text
antak.aspx
Readme.md
```

So our main web shell is:

```text
/usr/share/nishang/Antak-WebShell/antak.aspx
```

---

# 💙 6️⃣ How Antak Works

Antak is designed to behave somewhat like a **PowerShell console through a web interface**.

The basic flow is:

```text
🌐 Browser
    │
    │ HTTP Request
    ▼
🕸️ Antak.aspx
    │
    ▼
💙 PowerShell
    │
    ▼
🎯 Windows Target
    │
    ▼
📤 Command Output
    │
    ▼
🌐 Browser
```

Unlike a normal interactive PowerShell terminal, Antak executes commands through the web application.

> [!important]+ 🧠 Important Detail  
> Antak executes each command as a **new process**.
> 
> Therefore, it should not be thought of as exactly the same thing as having a persistent interactive PowerShell terminal.

---

# ⚙️ 7️⃣ Antak Capabilities

Antak provides several useful capabilities through its web interface.

It can:

```text
💻 Execute PowerShell commands
📜 Execute scripts
📤 Upload files
📥 Download files
🔐 Encode commands
⚙️ Execute encoded commands
🔎 Parse web.config
🗄️ Execute SQL queries
```

Conceptually:

```text
                  🕸️ ANTAK
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
   💻 Commands    📤 Upload     📥 Download
       │
       ├──────────────► 📜 Scripts
       │
       ├──────────────► 🔐 Encode
       │
       └──────────────► 🗄️ SQL
```

> [!tip]+ 🧠 Remember  
> Antak is more than simply a command textbox.
> 
> It provides several functions for interacting with a Windows web server through PowerShell.

---

# 📄 8️⃣ Make a Working Copy

Just like with Laudanum, we should avoid modifying the original file directly.

Create a copy:

```bash
cp /usr/share/nishang/Antak-WebShell/antak.aspx /home/administrator/Upload.aspx
```

Now:

```text
📦 Original
/usr/share/nishang/Antak-WebShell/antak.aspx
        │
        │ COPY
        ▼
📄 Working File
/home/administrator/Upload.aspx
```

> [!tip]+ 🧠 Good Habit  
> Keep the original payload untouched and modify a working copy.
> 
> This makes it easy to return to the original if you make a mistake or need another copy later.

---

# 🔐 9️⃣ Configure Antak Credentials

Antak allows us to configure credentials that must be supplied before accessing the web shell.

The material instructs us to modify **line 14** and specify:

```text
👤 Username
🔑 Password
```

Conceptually:

```text
🌐 Browse to Antak
        │
        ▼
🔐 Login Prompt
        │
        ├── ❌ Incorrect → No access
        │
        └── ✅ Correct
                │
                ▼
          🕸️ Antak Shell
```

> [!important]+ 🔐 Why Add Credentials?  
> If the uploaded web shell is accessible through a predictable URL, anyone who can reach it may attempt to use it.
> 
> Requiring credentials adds an additional layer of protection around the web shell.
> 
> ```text
> Web shell URL
>      +
> Authentication
>      ↓
> Restricted access
> ```

---

# 🧹 🔟 Remove Unnecessary Content

As with Laudanum, it can be useful to remove unnecessary:

```text
🎨 ASCII art
💬 Comments
```

from the payload.

Known payload content can potentially contribute to detection by:

```text
🛡️ Antivirus
🛡️ EDR
🔎 Security scanners
```

Conceptually:

```text
📄 Payload
   │
   ├── Known strings
   ├── Known comments
   └── Known patterns
          │
          ▼
       🛡️ Detection
```

> [!warning]+ 🛡️ CPTS Perspective  
> This is an important lesson about payloads:
> 
> **Payload functionality and payload detectability are separate concerns.**
> 
> A payload may work perfectly from a technical perspective while still being detected by endpoint security.

---

# 📤 1️⃣1️⃣ Upload Antak

The lab uses the same web application from the previous Laudanum exercise.

The target is a **Windows host**, making ASPX + PowerShell an appropriate combination.

```text
🎯 Windows Server
       │
       ▼
🌐 ASP.NET
       │
       ▼
📄 ASPX
       │
       ▼
🕸️ Antak
       │
       ▼
💙 PowerShell
```

We upload:

```text
Upload.aspx
```

through the application's file-upload functionality.

---

# 📁 1️⃣2️⃣ Find the Uploaded File

The application stores uploaded files in:

```text
/files/
```

Therefore our uploaded shell becomes accessible through something similar to:

```text
status.inlanefreight.local/files/upload.aspx
```

The process is:

```text
📄 Upload.aspx
      │
      ▼
📤 Upload Function
      │
      ▼
📁 /files/
      │
      ▼
📄 upload.aspx
      │
      ▼
🌐 Browser
```

> [!warning]+ ⚠️ Don't Assume the Path  
> The `/files/` location is specific to this lab.
> 
> In another environment, uploaded files may:
> 
> - Have randomized names.
>     
> - Be stored outside the web root.
>     
> - Require authentication.
>     
> - Be inaccessible directly.
>     
> - Be blocked from execution.
>     
> 
> Always **enumerate and verify** the actual upload behavior.

---

# 🔑 1️⃣3️⃣ Antak Login

When we navigate to the uploaded ASPX file, Antak presents an authentication prompt.

```text
🌐 Browser
     │
     ▼
🕸️ Antak.aspx
     │
     ▼
🔐 Username + Password
     │
     ▼
      ? 
     │
 ┌───┴────┐
 ▼        ▼
❌        ✅
Denied    Shell
```

Enter the credentials configured inside the ASPX file.

If they are correct, access to the Antak interface is granted.

---

# 🖥️ 1️⃣4️⃣ Antak Interface

After successful authentication, we receive a PowerShell-themed web interface.

The interface provides functionality such as:

```text
💻 Submit
📁 Browse
📤 Upload the File
🔐 Encode and Execute
📥 Download
⚙️ Parse web.config
🗄️ Execute SQL Query
```

This gives us a much more capable web shell than a simple command-execution page.

---

# 💻 1️⃣5️⃣ Executing PowerShell Commands

Once authenticated, we can issue PowerShell commands.

For example:

```powershell
Get-ChildItem
```

or:

```powershell
Get-Location
```

The command is sent through the web interface and the output is returned to the browser.

```text
🌐 Browser
     │
     │ PowerShell command
     ▼
🕸️ Antak
     │
     ▼
💙 PowerShell
     │
     ▼
🪟 Windows
     │
     ▼
📤 Output
     │
     ▼
🌐 Browser
```

> [!success]+ 🎯 What We Have Achieved  
> We now have **PowerShell command execution on the Windows target through a web shell**.
> 
> ```text
> 🌐 Browser
>      ↓
> 🕸️ Antak
>      ↓
> 💙 PowerShell
>      ↓
> 🪟 Windows
> ```

---

# 📜 1️⃣6️⃣ Executing Scripts

Antak can also work with PowerShell scripts.

Conceptually:

```text
📜 PowerShell Script
        │
        ▼
🕸️ Antak
        │
        ▼
💙 PowerShell
        │
        ▼
🎯 Windows Target
```

This is useful because sometimes a single command is not enough.

Instead, we may need to execute a series of commands or a larger PowerShell payload.

---

# 🔐 1️⃣7️⃣ Encode and Execute

Antak provides an **Encode and Execute** option.

Conceptually:

```text
💻 PowerShell Command
        │
        ▼
🔐 Encode
        │
        ▼
📤 Send
        │
        ▼
🎯 Target
        │
        ▼
💙 PowerShell
```

Encoding changes how the command is represented.

> [!warning]+ 🧠 Encoding ≠ Encryption ≠ Guaranteed Evasion  
> Encoding should **not** be confused with encryption.
> 
> Encoding simply changes the representation of data.
> 
> ```text
> Encoding
>    ≠
> Encryption
>    ≠
> Guaranteed AV bypass
> ```
> 
> Modern security products can inspect encoded PowerShell commands and detect malicious behavior.

---

# 📤 1️⃣8️⃣ File Upload and Download

Antak also provides file-transfer functionality.

### 📤 Upload

```text
🖥️ Attack Box
      │
      │ File
      ▼
🕸️ Antak
      │
      ▼
🎯 Windows Target
```

### 📥 Download

```text
🎯 Windows Target
      │
      │ File
      ▼
🕸️ Antak
      │
      ▼
🖥️ Attack Box
```

This can be useful when moving tools or payloads during a penetration test.

> [!important]+ 💡 Remember  
> A web shell can be more than just:
> 
> ```text
> 💻 Execute command
> ```
> 
> It can also provide mechanisms for:
> 
> ```text
> 📤 File upload
> 📥 File download
> 📜 Script execution
> 🔐 Command encoding
> ```

---

# 🔎 1️⃣9️⃣ `help` Is Your Friend

If you're unsure what Antak supports, the interface provides a `help` command.

```powershell
help
```

This is a useful general penetration-testing habit:

> [!tip]+ 🧠 Don't Guess — Enumerate the Tool  
> When you land on a new shell or tool:
> 
> ```text
> ❓ What can it do?
>       ↓
> 💻 help
>       ↓
> 📋 Available functionality
>       ↓
> 🧠 Choose the appropriate feature
> ```
> 
> The same principle applies to:
> 
> - Metasploit
>     
> - Meterpreter
>     
> - PowerShell
>     
> - Linux shells
>     
> - Web shells
>     
> - Other post-exploitation tools
>     

---

# 🆚 2️⃣0️⃣ Antak vs Laudanum

Both Antak and Laudanum can provide web-shell functionality, but they have different focuses.

||🕸️ Laudanum|💙 Antak|
|---|---|---|
|Primary use|Web shell|Web shell|
|ASPX support|✅|✅|
|Windows focused|✅|✅|
|PowerShell based|❌|✅|
|Authentication|Depends on shell|✅ Built-in|
|File upload|Depends on shell|✅|
|File download|Depends on shell|✅|
|Command encoding|Limited/varies|✅|
|Project|Laudanum|Nishang|

### 🕸️ Laudanum

```text
📄 ASPX
   ↓
🕸️ Web Shell
   ↓
💻 Windows Commands
```

### 💙 Antak

```text
📄 ASPX
   ↓
🕸️ Antak
   ↓
💙 PowerShell
   ↓
🪟 Windows
```

> [!important]+ 🧠 Main Difference  
> **Antak is specifically designed around PowerShell**, making it especially useful when targeting Windows systems where PowerShell is available.

---

# 🔗 2️⃣1️⃣ From Web Shell to Callback

A web shell can also act as a stepping stone toward a more traditional shell or command-and-control connection.

The general concept is:

```text
📤 Upload Web Shell
        ↓
🕸️ Web Shell
        ↓
💻 Command Execution
        ↓
📜 Execute PowerShell Payload
        ↓
🔄 Callback
        ↓
🖥️ Attack Machine
        ↓
🐚 More Interactive Session
```

> [!important]+ 🎯 CPTS Concept  
> Think of the web shell as **initial command execution**.
> 
> It doesn't necessarily have to be your final shell.
> 
> You may use it to:
> 
> ```text
> 🔎 Enumerate
> 📤 Upload tools
> 📜 Execute scripts
> 🔄 Establish a callback
> 🐚 Upgrade your access
> ```

---

# 🧠 2️⃣2️⃣ The Important Technology Chain

One of the most important things to understand from this section is the relationship between the technologies.

```text
🪟 Windows
   │
   ▼
🌐 IIS / ASP.NET
   │
   ▼
📄 ASPX
   │
   ▼
🕸️ Antak
   │
   ▼
💙 PowerShell
   │
   ▼
💻 Windows Command Execution
```

This gives us a methodology for selecting a web shell.

> [!success]+ 🧠 Payload Selection Mental Model
> 
> ```text
> 🔎 Identify Target OS
>          ↓
> 🔎 Identify Web Server
>          ↓
> 🔎 Identify Web Technology
>          ↓
> 📄 Select Compatible Shell
>          ↓
> 📤 Upload
>          ↓
> 🌐 Access
>          ↓
> 💻 Execute
> ```
> 
> **Don't choose the payload first. Identify the environment first.**

---

# 🔍 2️⃣3️⃣ General ASPX Web-Shell Methodology

When you discover a Windows web server and a potential file-upload vulnerability, think through the following:

```text
1️⃣ 🔎 What OS is the target running?
             ↓
2️⃣ 🌐 What web server is running?
             ↓
3️⃣ ⚙️ Is ASP.NET available?
             ↓
4️⃣ 📤 Is there a file-upload function?
             ↓
5️⃣ 📁 Where are uploaded files stored?
             ↓
6️⃣ ⚙️ Are uploaded ASPX files executed?
             ↓
7️⃣ 🕸️ Can we deploy a compatible web shell?
             ↓
8️⃣ 🔐 Can access be protected?
             ↓
9️⃣ 💻 Can we execute commands?
             ↓
🔟 🔎 What can we enumerate from the shell?
```

This is much more valuable than simply memorizing:

```text
"Use Antak."
```

Because the real CPTS skill is understanding **why and when** to use it.

---

# 🧩 2️⃣4️⃣ Web Shell Attack Chain

The complete attack chain looks like:

```text
                    🎯 WINDOWS WEB SERVER
                             │
                             ▼
                    🔎 ENUMERATION
                             │
                             ▼
                   🌐 IDENTIFY ASP.NET
                             │
                             ▼
                    📤 FIND UPLOAD
                             │
                             ▼
                   📄 PREPARE ANTАK
                             │
                             ▼
                    ✏️ CONFIGURE
                             │
                             ▼
                      🔐 ADD CREDS
                             │
                             ▼
                       📤 UPLOAD
                             │
                             ▼
                     📁 /files/
                             │
                             ▼
                   🌐 ACCESS ASPX
                             │
                             ▼
                     🔐 AUTHENTICATE
                             │
                             ▼
                       🕸️ ANTAK
                             │
                             ▼
                      💙 POWERSHELL
                             │
                             ▼
                     💻 COMMAND EXEC
                             │
                             ▼
                       🔎 ENUMERATE
                             │
                             ▼
                  🔄 FURTHER ACCESS
```

---

# 📌 2️⃣5️⃣ Quick Reference

### 📁 Antak Location

```text
/usr/share/nishang/Antak-WebShell/antak.aspx
```

### 📄 Create Working Copy

```bash
cp /usr/share/nishang/Antak-WebShell/antak.aspx /home/administrator/Upload.aspx
```

### ✏️ Configure

```text
👤 Username
🔑 Password
```

### 📤 Upload

```text
Upload.aspx
      ↓
Target file-upload function
```

### 🌐 Access

```text
https://status.inlanefreight.local/files/upload.aspx
```

### 💻 PowerShell

```powershell
Get-ChildItem
```

### 🆘 Help

```powershell
help
```

---

# 🎯 2️⃣6️⃣ Key Takeaways

- 🕸️ **Antak** is an ASPX web shell designed for Windows.
    
- 💙 Antak uses **PowerShell** to interact with the target.
    
- 📦 Antak is included in the **Nishang** project.
    
- 📁 On Kali/Parrot, it can be found under `/usr/share/nishang/Antak-WebShell`.
    
- 📄 Make a copy of the original `antak.aspx` before modifying it.
    
- 🔐 Antak allows credentials to be configured for web-shell access.
    
- 🌐 ASPX is a technology associated with **Microsoft ASP.NET**.
    
- 📤 A vulnerable file-upload function can potentially allow an ASPX web shell to be deployed.
    
- ⚙️ The target must be capable of **processing/executing ASPX** for the shell to work.
    
- 💻 Antak provides PowerShell-based command execution through a browser.
    
- 📤 Antak can upload files.
    
- 📥 Antak can download files.
    
- 📜 Antak can execute PowerShell scripts.
    
- 🔐 Antak includes an encode-and-execute feature.
    
- 🛡️ Encoding does not guarantee antivirus/EDR evasion.
    
- 🧠 `help` is useful for discovering the functionality available within a shell.
    
- 🎯 A web shell can serve as an **initial foothold** and may be used to obtain more interactive access.
    
- 🔎 Always identify the **OS + web server + web technology + upload behavior** before selecting a payload.
    

> [!success]+ 🧠 FINAL CPTS MENTAL MODEL
> 
> ```text
> 🔎 ENUMERATE
>      ↓
> 🪟 IDENTIFY WINDOWS
>      ↓
> 🌐 IDENTIFY ASP.NET / ASPX
>      ↓
> 📤 FIND FILE UPLOAD
>      ↓
> 📁 DETERMINE UPLOAD LOCATION
>      ↓
> ⚙️ DETERMINE IF ASPX EXECUTES
>      ↓
> 📄 PREPARE ANTAK
>      ↓
> 🔐 CONFIGURE ACCESS
>      ↓
> 📤 UPLOAD
>      ↓
> 🌐 ACCESS WEB SHELL
>      ↓
> 🔐 AUTHENTICATE
>      ↓
> 🕸️ ANTAK
>      ↓
> 💙 POWERSHELL
>      ↓
> 💻 COMMAND EXECUTION
>      ↓
> 🔎 ENUMERATE TARGET
>      ↓
> 🔄 CONSIDER NEXT STEP
> ```
> 
> **The core idea:**
> 
> > 🪟 **Windows + ASP.NET + Executable ASPX Upload = Potential ASPX Web Shell**
> 
> ```text
> 🐛 Upload/Execution Weakness
>             +
> 📄 ASPX Payload
>             +
> 💙 PowerShell
>             ↓
>        🕸️ ANTAK
>             ↓
>       💻 COMMAND EXECUTION
> ```