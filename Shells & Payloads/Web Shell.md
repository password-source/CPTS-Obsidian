# 🕸️ Laudanum — One Webshell to Rule Them All

> [!info]+ 🎯 What Is Laudanum?  
> **Laudanum** is a collection of ready-made web shells and injectable files that can be uploaded to a target web server to:
> 
> - 🐚 Obtain a web shell
>     
> - 💻 Execute commands through a browser
>     
> - 🔄 Establish reverse-shell callbacks
>     
> - ⚙️ Interact with the underlying operating system
>     
> 
> It contains files for multiple web application languages, including:
> 
> ```text
> 🌐 ASP
> 🌐 ASPX
> 🌐 JSP
> 🌐 PHP
> ```
> 
> The basic idea is:
> 
> ```text
> 🎯 Vulnerable Web Application
>             │
>             ▼
>        📤 File Upload
>             │
>             ▼
>       🕸️ Web Shell
>             │
>             ▼
>     💻 Execute Commands
>             │
>             ▼
>       🎯 Target System
> ```

---

# 🧠 1️⃣ Why Laudanum Is Useful

During a penetration test, we may discover a web application that allows us to upload files.

If we can upload a file that the web server interprets as executable code, we may be able to turn that upload functionality into **remote command execution**.

Laudanum provides pre-built files that can help us do exactly that.

```text
🔎 Discover upload functionality
             ↓
📤 Upload executable web file
             ↓
🌐 Web server processes file
             ↓
🕸️ Access web shell
             ↓
💻 Execute commands
```

> [!important]+ 💡 Core Concept  
> **A web shell is a file hosted on the target web server that allows commands or other actions to be executed through a web request.**
> 
> Instead of having:
> 
> ```text
> 🖥️ Terminal → Target
> ```
> 
> we can have:
> 
> ```text
> 🌐 Browser
>      │
>      ▼
> 🕸️ Web Shell
>      │
>      ▼
> 🎯 Target OS
> ```

---

# 🧰 2️⃣ Laudanum Location

On Kali and Parrot OS, Laudanum is included by default.

The files can be found in:

```text
/usr/share/laudanum
```

For example:

```bash
ls -la /usr/share/laudanum
```

The repository contains different shells based on the target's web technology.

```text
/usr/share/laudanum
        │
        ├── asp/
        │
        ├── aspx/
        │
        ├── jsp/
        │
        └── php/
```

> [!tip]+ 🧠 Remember  
> **Choose the web-shell type based on the technology the target web server can execute.**
> 
> ```text
> ASPX application → ASPX shell
> PHP application  → PHP shell
> JSP application  → JSP shell
> ```

---

# 🔍 3️⃣ Read the Payload Before Using It

Laudanum provides ready-made files, but we should **always inspect them before uploading them**.

> [!warning]+ ⚠️ Don't Blindly Upload Payloads  
> Before using a Laudanum file:
> 
> - 🔎 Read the source code.
>     
> - 💬 Read the comments.
>     
> - 🧩 Understand what it does.
>     
> - 🌐 Identify any IP addresses that need changing.
>     
> - ⚙️ Check what configuration is required.
>     
> 
> ```text
> 📄 Laudanum File
>       ↓
> 🔎 Read
>       ↓
> 🧠 Understand
>       ↓
> ✏️ Modify
>       ↓
> 📤 Upload
>       ↓
> 🕸️ Use
> ```

---

# ✏️ 4️⃣ Make a Copy Before Modifying

The recommended approach is to **copy the original Laudanum file** rather than modifying the original directly.

Example:

```bash
cp /usr/share/laudanum/aspx/shell.aspx /home/tester/demo.aspx
```

Now we have:

```text
Original:
 /usr/share/laudanum/aspx/shell.aspx

Working copy:
 /home/tester/demo.aspx
```

### 🧠 Why Make a Copy?

```text
📦 Original Laudanum File
          │
          ├── Keep untouched
          │
          ▼
     📄 Working Copy
          │
          ▼
      ✏️ Modify
          │
          ▼
      📤 Upload
```

This keeps the original template available if we need it again.

---

# 🎯 5️⃣ Configure the Allowed IP

Some Laudanum shell files have an `allowedIps` variable.

For the ASPX shell, the material instructs us to modify the value on the appropriate line and add our attack-machine IP.

Example:

```text
allowedIps
    │
    ▼
10.10.14.12
```

### 🧠 Why?

The shell can restrict who is allowed to interact with it.

Conceptually:

```text
🌐 Request
   │
   ▼
🕸️ Web Shell
   │
   ▼
Is source IP allowed?
   │
   ├── ❌ No → Reject
   │
   └── ✅ Yes → Execute
```

> [!important]+ 🔐 Security Configuration  
> The `allowedIps` setting is important because it controls which IP addresses can interact with the shell.
> 
> Make sure it contains the **appropriate attack-machine IP** for your lab environment.

---

# 🧹 6️⃣ Remove Unnecessary Payload Content

The material also mentions removing:

```text
ASCII art
Comments
```

from the payload.

Why?

Security products may use **signatures** to identify known malicious files.

For example:

```text
📄 Payload
   │
   ├── Known code
   ├── ASCII art
   └── Known comments
          │
          ▼
     🛡️ AV / Defender
          │
          ▼
       🚨 Alert
```

> [!warning]+ 🛡️ Defensive Perspective  
> Payloads can be detected based on recognizable content.
> 
> Removing unnecessary content can reduce obvious signatures, but **this does not guarantee that a payload will evade antivirus or EDR**.
> 
> In a real engagement, always operate within the agreed testing scope.

---

# 🌐 7️⃣ Identify a File Upload Function

In the lab, the target web application contains an upload function.

The application provides:

```text
📄 Configuration File
        │
        ▼
     📤 Upload
```

This is the functionality we use to deliver the Laudanum ASPX shell.

> [!important]+ 💡 The Vulnerability Chain  
> The interesting part isn't simply that Laudanum exists.
> 
> The important chain is:
> 
> ```text
> 📤 File Upload
>       +
> ⚙️ Server Executes Uploaded File
>       ↓
> 🕸️ Web Shell
>       ↓
> 💻 Command Execution
> ```
> 
> **File upload by itself does not automatically mean RCE.**
> 
> We need to determine whether the uploaded file can be **executed/interpreted by the web server**.

---

# 📤 8️⃣ Upload the Laudanum Shell

After modifying the shell:

```text
demo.aspx
```

we upload it through the application's upload functionality.

```text
🖥️ Attack Box
      │
      │ demo.aspx
      ▼
🌐 Web Application
      │
      ▼
📤 Upload Function
      │
      ▼
📁 Server Filesystem
```

If successful, the application displays the location where the file was saved.

For this lab, the shell is placed under:

```text
/files/demo.aspx
```

---

# 📁 9️⃣ Locate the Uploaded Shell

The target application stores the uploaded file in a publicly accessible `files` directory.

Conceptually:

```text
🌐 status.inlanefreight.local
             │
             ▼
          /files/
             │
             ▼
        demo.aspx
```

We can then navigate to:

```text
status.inlanefreight.local/files/demo.aspx
```

> [!warning]+ ⚠️ Upload Behavior Varies  
> Don't assume every application behaves like this lab.
> 
> Real applications may:
> 
> - 🔀 Randomize uploaded filenames.
>     
> - 📁 Store files outside the web root.
>     
> - 🚫 Prevent direct access.
>     
> - 🔒 Disable script execution.
>     
> - 🛡️ Scan uploaded files.
>     
> - 🔐 Require authentication.
>     
> 
> Therefore, after an upload, determine **where the file went and whether it is executable**.

---

# 🕸️ 🔟 Access the Laudanum Web Shell

Once we navigate to the uploaded file, the Laudanum interface appears.

The interface provides a way to submit commands to the target.

Conceptually:

```text
🌐 Browser
     │
     │ HTTP Request
     ▼
🕸️ Laudanum Web Shell
     │
     │ Execute command
     ▼
🎯 Windows Host
     │
     ▼
📤 Command Output
     │
     ▼
🌐 Browser
```

This means we don't necessarily need a traditional terminal to interact with the compromised system.

---

# 💻 1️⃣1️⃣ Executing Commands

In the lab, the shell can execute Windows commands.

For example:

```text
systeminfo
```

The output is returned through the browser.

```text
🌐 Browser
    │
    │ systeminfo
    ▼
🕸️ demo.aspx
    │
    ▼
🎯 Windows OS
    │
    ▼
📊 systeminfo
    │
    ▼
🌐 Browser
```

The response can reveal useful information such as:

```text
🖥️ Hostname
🪟 OS version
🏭 Manufacturer
⚙️ System type
🧠 Processor
💾 Memory
🌐 Network information
```

> [!success]+ 🎯 Result  
> We have achieved **command execution through the web shell**.
> 
> ```text
> File Upload
>      ↓
> ASPX Web Shell
>      ↓
> Browser
>      ↓
> Command Execution
>      ↓
> Target Information
> ```

---

# 🧩 1️⃣2️⃣ Web Shell vs Reverse Shell

It is important to distinguish between these two.

||🕸️ Web Shell|🔄 Reverse Shell|
|---|---|---|
|Interface|Browser/web requests|Terminal/network connection|
|Commands|Sent through HTTP|Sent through shell connection|
|Listener required|❌ Not necessarily|✅ Usually|
|Initial access|Uploaded/executed web file|Target connects back|
|Interaction|Web-based|Terminal-based|
|Example|Laudanum|Netcat / Meterpreter|

### 🕸️ Web Shell

```text
🌐 Browser
   │
   ▼
🕸️ Web Shell
   │
   ▼
🎯 Target
```

### 🔄 Reverse Shell

```text
🎯 Target
   │
   │ Connects back
   ▼
🖥️ Attack Box
   │
   ▼
🐚 Shell
```

> [!important]+ 🧠 Key Difference  
> A **web shell** gives you command execution through a web-accessible file.
> 
> A **reverse shell** gives you a shell session through a network connection back to your attack machine.
> 
> A web shell can potentially be used as a stepping stone toward a more interactive shell.

---

# 🪟 1️⃣3️⃣ Why ASPX in This Lab?

The target application is running on a Windows web server capable of processing **ASPX** files.

Therefore:

```text
🎯 Windows Web Server
       │
       ▼
⚙️ ASP.NET
       │
       ▼
📄 .aspx
       │
       ▼
🕸️ Laudanum ASPX Shell
```

This illustrates a very important penetration-testing concept:

> [!tip]+ 🧠 Match the Payload to the Technology
> 
> ```text
> Identify web technology
>          ↓
> Choose compatible payload
>          ↓
> Upload
>          ↓
> Execute
> ```
> 
> Don't choose a PHP web shell simply because PHP is familiar.
> 
> The target must be able to **interpret the file type** you upload.

---

# 🔎 1️⃣4️⃣ A General Web-Shell Methodology

When you discover a file-upload function, use this thought process:

```text
📤 FILE UPLOAD FOUND
        │
        ▼
🔎 What files are accepted?
        │
        ▼
🔎 Where are uploads stored?
        │
        ▼
🔎 Can uploaded files be accessed?
        │
        ▼
🔎 Can uploaded files be executed?
        │
        ▼
⚙️ What web technology is running?
        │
        ▼
📄 Choose compatible payload
        │
        ▼
📤 Upload
        │
        ▼
🌐 Access uploaded file
        │
        ▼
🕸️ Web Shell
        │
        ▼
💻 Command Execution
```

> [!warning]+ 🚨 Don't Assume File Upload = RCE  
> A secure upload implementation may allow:
> 
> ```text
> 📄 image.jpg
> 📄 document.pdf
> ```
> 
> but prevent:
> 
> ```text
> ❌ .aspx
> ❌ .php
> ❌ .jsp
> ```
> 
> or may store them somewhere where the web server cannot execute them.
> 
> Always test and understand the application's upload behavior.

---

# 🧠 1️⃣5️⃣ The Bigger Picture

Laudanum is not the vulnerability.

This distinction is extremely important.

```text
❌ Laudanum = vulnerability
```

is incorrect.

Instead:

```text
🐛 Vulnerable Upload Function
             │
             ▼
📤 Ability to Upload Executable File
             │
             ▼
🕸️ Laudanum Web Shell
             │
             ▼
💻 Command Execution
```

> [!success]+ 💡 CPTS Concept  
> **The vulnerability is what allows you to deliver/execute the payload.**
> 
> **The payload is what performs the action after delivery.**
> 
> ```text
> 🐛 Vulnerability
>       +
> 📦 Payload
>       ↓
> 💥 Exploitation
>       ↓
> 🐚 Access
> ```

---

# 🔄 1️⃣6️⃣ Complete Laudanum Attack Chain

```text
                       🎯 TARGET WEB SERVER
                              │
                              ▼
                     🔎 ENUMERATE APPLICATION
                              │
                              ▼
                     📤 FIND UPLOAD FUNCTION
                              │
                              ▼
                    ⚙️ IDENTIFY WEB TECHNOLOGY
                              │
                              ▼
                         🪟 ASP.NET
                              │
                              ▼
                    🕸️ SELECT ASPX SHELL
                              │
                              ▼
                 📄 Copy Laudanum Template
                              │
                              ▼
                    ✏️ Modify Configuration
                              │
                              ▼
                     📤 Upload `demo.aspx`
                              │
                              ▼
                    📁 `/files/demo.aspx`
                              │
                              ▼
                    🌐 Browse to Web Shell
                              │
                              ▼
                       💻 Execute Command
                              │
                              ▼
                         `systeminfo`
                              │
                              ▼
                    📊 Receive Output
```

---

# 🧰 1️⃣7️⃣ Quick Reference

### 📁 Laudanum Location

```text
/usr/share/laudanum
```

### 📄 Copy ASPX Shell

```bash
cp /usr/share/laudanum/aspx/shell.aspx /home/tester/demo.aspx
```

### ✏️ Modify

```text
allowedIps → <ATTACKER_IP>
```

### 📤 Upload

```text
demo.aspx
      ↓
Target Upload Function
```

### 🌐 Access

```text
status.inlanefreight.local/files/demo.aspx
```

### 💻 Test Command

```text
systeminfo
```

---

# 🎯 1️⃣8️⃣ Key Takeaways

- 🕸️ **Laudanum** is a collection of ready-made web shells and injectable files.
    
- 🌐 It supports multiple web technologies, including **ASP, ASPX, JSP, and PHP**.
    
- 📁 On Kali/Parrot, Laudanum is commonly located in `/usr/share/laudanum`.
    
- 📄 Make a **copy** of the original payload before modifying it.
    
- ✏️ Some shells require configuration such as the `allowedIps` value.
    
- 🔎 Always **read the payload source and comments** before using it.
    
- 🛡️ Payloads can contain recognizable signatures that security tools may detect.
    
- 📤 A file-upload function can potentially become a path to code execution if the server allows uploaded executable files to run.
    
- ⚙️ The web shell must match the **technology supported by the target web server**.
    
- 🕸️ A web shell provides command execution through a web interface.
    
- 🔄 A reverse shell provides a more traditional shell connection back to the attack machine.
    
- 🧠 **Laudanum itself is not the vulnerability** — the vulnerability is what allows us to deliver and execute the payload.
    
- 🔎 After uploading a file, determine **where it is stored, whether it is accessible, and whether it is executable**.
    
- 🎯 A web shell can provide an initial foothold that may later be used to obtain a more interactive shell and continue enumeration.
    

> [!success]+ 🧠 FINAL CPTS MENTAL MODEL
> 
> ```text
> 🔎 ENUMERATE
>       ↓
> 🌐 IDENTIFY WEB TECHNOLOGY
>       ↓
> 📤 FIND FILE UPLOAD
>       ↓
> 🔎 UNDERSTAND UPLOAD BEHAVIOR
>       ↓
> 📄 CHOOSE COMPATIBLE WEB SHELL
>       ↓
> ✏️ CONFIGURE PAYLOAD
>       ↓
> 📤 UPLOAD
>       ↓
> 🌐 ACCESS UPLOADED FILE
>       ↓
> 🕸️ WEB SHELL
>       ↓
> 💻 COMMAND EXECUTION
>       ↓
> 👤 IDENTIFY ACCESS
>       ↓
> 🔎 ENUMERATE FURTHER
> ```
> 
> **The core idea:**
> 
> > 📤 **Upload + Execute + Access = Potential Web Shell**
> 
> ```text
> 🐛 Vulnerability
>       +
> 📦 Compatible Payload
>       ↓
> 🕸️ Web Shell
>       ↓
> 💻 Command Execution
> ```