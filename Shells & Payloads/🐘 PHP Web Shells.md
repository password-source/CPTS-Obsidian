# 🐘 PHP Web Shells

> [!info]+ 🎯 What Is PHP?  
> **PHP (Hypertext Preprocessor)** is an open-source, general-purpose scripting language commonly used on the **server side** of web applications.
> 
> When a web server uses PHP, requests to files such as:
> 
> ```text
> login.php
> upload.php
> index.php
> ```
> 
> may be processed by PHP on the server.
> 
> 🧠 **Pentesting relevance:**  
> If we discover that a target uses PHP, this tells us that a **PHP-based web shell may potentially execute on the target** if we can find a way to upload or create a `.php` file in an executable web-accessible location.

---

# 🌐 1️⃣ Why PHP Matters During Enumeration

Suppose we discover a login page:

```text
https://target/login.php
```

The `.php` extension gives us an important clue:

```text
Browser
   │
   │ HTTP Request
   ▼
🌐 Web Server
   │
   ▼
🐘 PHP
   │
   └── Processes server-side code
```

For a penetration tester, this means we should consider:

```text
❓ Can we upload a PHP file?
❓ Can we create a PHP file?
❓ Is there a file-upload vulnerability?
❓ Is the uploaded file stored in a web-accessible directory?
❓ Will the server execute PHP in that directory?
```

> [!important]+ 🧠 Key Idea  
> **Knowing the server-side language helps determine what type of web shell may work.**
> 
> ```text
> PHP server       → PHP web shell
> ASP.NET server   → ASPX web shell
> JSP server       → JSP web shell
> ```

---

# 🐘 2️⃣ PHP Web Shells

A **PHP web shell** is a PHP file that allows commands to be executed through a web browser.

Conceptually:

```text
🖥️ Attacker
     │
     │ HTTP Request
     ▼
🌐 Web Server
     │
     ▼
🐘 PHP Web Shell
     │
     ▼
🖥️ Operating System
     │
     └── Command execution
```

Instead of receiving a normal interactive terminal, we can interact with the target through the browser.

### 🎯 Two Common Goals

A PHP web shell can potentially provide:

```text
1️⃣ Browser-based command execution

2️⃣ A callback / reverse shell
       ↓
   🖥️ Attack Box
```

The web shell is therefore often a **stepping stone** toward obtaining a more useful shell.

---

# 🧪 3️⃣ rConfig Example

Earlier, we identified an **rConfig 3.9.6** server running on Linux.

The application uses PHP.

We can log into rConfig and navigate to:

```text
Devices
   ↓
Vendors
   ↓
Add Vendor
```

The vendor functionality provides an opportunity to upload a **vendor logo**.

At first glance, this appears to be an image-only upload:

```text
Allowed:
.png
.jpg
.gif
...
```

Our goal is to determine whether we can abuse this functionality to upload a PHP file.

---

# 📤 4️⃣ PHP Web Shell Upload

For the demonstration, we use **WhiteWinterWolf's PHP Web Shell**.

The important part is:

```text
PHP Web Shell
      │
      ▼
   .php file
      │
      ▼
Upload functionality
      │
      ▼
Target web server
```

However, the application checks the uploaded file type.

Therefore:

```text
connect.php
     │
     ▼
❌ Rejected
     │
     └── Application expects an image
```

This is where understanding **HTTP requests** becomes important.

---

# 🕵️ 5️⃣ Using Burp Suite

We can use **Burp Suite** as a web proxy to intercept and modify the upload request.

The browser is configured to send HTTP traffic through:

```text
Proxy IP:   127.0.0.1
Proxy Port: 8080
```

The traffic flow becomes:

```text
🌐 Browser
    │
    │ HTTP Request
    ▼
🕵️ Burp Suite
127.0.0.1:8080
    │
    │ Forward / Modify
    ▼
🌐 Target Web Server
```

> [!tip]+ 🧠 Remember  
> **Burp Suite allows us to intercept HTTP requests before they reach the target.**
> 
> This lets us inspect and, when appropriate during an authorized assessment, modify request parameters and headers.

---

# 🧩 6️⃣ Bypassing the File-Type Restriction

The upload request contains a `Content-Type` header describing the type of data being uploaded.

For example:

```http
Content-Type: application/x-php
```

The application may reject this because it expects an image.

In the lab, we modify it to:

```http
Content-Type: image/gif
```

Conceptually:

```text
Original request

connect.php
Content-Type: application/x-php
        │
        ▼
     ❌ Reject


Modified request

connect.php
Content-Type: image/gif
        │
        ▼
     ✅ Accepted
```

> [!warning]+ ⚠️ Important  
> Changing `Content-Type` only changes what the request **claims** the file is.
> 
> It does **not** magically turn PHP into an image.
> 
> The bypass works only if the application's validation relies insufficiently on the client-supplied MIME type and the server later stores the file somewhere PHP execution is possible.

---

# 🔬 7️⃣ Understanding the Upload Bypass

The important methodology is:

```text
1️⃣ Identify an upload function
        ↓
2️⃣ Determine what file types are allowed
        ↓
3️⃣ Attempt a legitimate upload
        ↓
4️⃣ Intercept the request with Burp
        ↓
5️⃣ Inspect the upload request
        ↓
6️⃣ Examine filename and Content-Type validation
        ↓
7️⃣ Test whether validation can be bypassed
        ↓
8️⃣ Determine where the file is stored
        ↓
9️⃣ Determine whether the server executes it
        ↓
🔟 Access the resulting web shell
```

### 🧠 This Is the Important Part

Don't memorize:

```text
"Change application/x-php to image/gif"
```

Instead remember:

> **Upload restriction → intercept request → understand validation → test whether validation can be bypassed → determine storage location → test execution.**

This methodology applies much more broadly than this specific rConfig example.

---

# 📂 8️⃣ Finding the Uploaded Web Shell

After the upload succeeds, the application creates the vendor entry.

In this lab, the uploaded file is stored under:

```text
/images/vendor/
```

The PHP file is:

```text
/images/vendor/connect.php
```

We can therefore navigate to:

```text
https://TARGET/images/vendor/connect.php
```

The server processes the PHP file.

```text
🌐 Browser
     │
     │ GET /images/vendor/connect.php
     ▼
🌐 Apache
     │
     ▼
🐘 PHP
     │
     ▼
💻 OS Command
     │
     ▼
📤 Output returned to browser
```

🎉 We now have a **browser-based web shell**.

---

# 🐚 9️⃣ What Kind of Shell Do We Have?

The resulting shell is:

> [!warning]+ ⚠️ Non-Interactive Web Shell  
> We are **not** sitting inside a normal terminal.
> 
> Instead:
> 
> ```text
> Browser
>    │
>    ├── Send command
>    │
>    ▼
> PHP Web Shell
>    │
>    ▼
> Operating System
>    │
>    └── Execute command
>             │
>             ▼
>         Return output
> ```
> 
> This provides command execution, but usually lacks the functionality of a proper interactive shell.

---

# ⚠️ 🔟 Limitations of Web Shells

Web shells can be extremely useful, but they have several disadvantages.

### 1️⃣ Limited Interactivity

You may have difficulty with:

```text
❌ Interactive programs
❌ Full terminal interaction
❌ File-system navigation
❌ Uploading/downloading files
❌ Chaining commands
```

For example:

```bash
whoami && hostname
```

may not behave as expected depending on how the web shell executes commands.

---

### 2️⃣ Instability

A web shell may provide an unreliable method of interacting with the operating system.

```text
Web Shell
    │
    └── Limited command execution
             │
             ▼
        Less stable
        than a real shell
```

---

### 3️⃣ Evidence Left Behind

Uploading a web shell creates artifacts.

Potential evidence can include:

```text
📄 Uploaded file
📂 File-system entry
🌐 Web-server logs
📝 Application logs
🔍 EDR / AV telemetry
```

Therefore, web shells can increase the chance that defenders discover the activity.

---

# 🔄 1️⃣1️⃣ Web Shell → Reverse Shell

A common objective is to use the web shell as an initial foothold and then establish a more useful reverse shell.

```text
                    INITIAL ACCESS
                          │
                          ▼
                   🐘 PHP Web Shell
                          │
                          ▼
                  Command Execution
                          │
                          ▼
                   🔄 Reverse Shell
                          │
                          ▼
                    🖥️ Attack Box
                          │
                          ▼
                    🐚 Better Shell
```

This can provide a more practical environment for:

```text
🔎 Enumeration
📂 File-system navigation
⬆️ File transfers
⬇️ File downloads
🔐 Privilege escalation
↔️ Lateral movement
```

---

# 🧹 1️⃣2️⃣ Operational Considerations

During an authorized penetration test, it is important to understand what evidence your actions create.

The material recommends documenting:

```text
📌 Payload used
📌 Filename
📌 Upload location
📌 Methods attempted
📌 Successful techniques
📌 Failed techniques
📌 File hashes
```

For example:

```text
Payload:
connect.php

Location:
/images/vendor/connect.php

Hash:
SHA1 / MD5
```

This provides useful attribution and evidence for the final report.

> [!important]+ 🧠 Engagement Consideration  
> Depending on the assessment type, the goal may include testing whether defenders can detect an attacker.
> 
> Therefore, **stealth and cleanup requirements depend on the rules of engagement**.
> 
> Do not assume that deleting evidence is always appropriate; follow the engagement's agreed scope and procedures.

---

# 🧠 1️⃣3️⃣ Web Shell Methodology

When you encounter a web application with a file-upload function, think:

```text
📤 FILE UPLOAD
      │
      ▼
❓ What file types are allowed?
      │
      ▼
❓ How is the file type validated?
      │
      ├── Extension?
      ├── Content-Type?
      ├── File signature / magic bytes?
      ├── Server-side inspection?
      └── Multiple validation layers?
      │
      ▼
❓ Where is the file stored?
      │
      ▼
❓ Is the location web-accessible?
      │
      ▼
❓ Will the server execute the uploaded file?
      │
      ▼
🐚 Web Shell
      │
      ▼
🔄 Possible Reverse Shell
```

This is much more valuable to remember than one specific exploit.

---

# 🔐 1️⃣4️⃣ Why File Upload Validation Matters

From a defensive perspective, this example demonstrates why checking only the client-provided MIME type is insufficient.

A secure upload implementation should consider multiple properties, such as:

```text
📄 Filename extension
🔍 File signature / magic bytes
📦 MIME type
📂 Storage location
⚙️ Execution permissions
🧱 Server-side validation
```

Ideally, uploaded files should be stored somewhere that **cannot execute server-side code**.

---

# 📌 1️⃣5️⃣ Quick Reference

### 🕵️ Burp Proxy

```text
IP:   127.0.0.1
Port: 8080
```

### 📤 Upload Concept

```text
PHP Web Shell
      │
      ▼
Upload function
      │
      ▼
🕵️ Burp Suite
      │
      ▼
Inspect / modify request
      │
      ▼
Target server
```

### 🧩 Lab Bypass

```text
Content-Type:
application/x-php

        ↓

image/gif
```

### 📂 Lab Web Shell Location

```text
/images/vendor/connect.php
```

### 🐚 Result

```text
Browser
   │
   ▼
PHP Web Shell
   │
   ▼
Linux OS
```

---

# 🧠 1️⃣6️⃣ Key Takeaways

- 🐘 **PHP** is a common server-side language and can potentially execute PHP web shells.
    
- 📤 File-upload functionality can become an attack vector if validation is weak.
    
- 🕵️ **Burp Suite** can intercept and modify HTTP requests during authorized testing.
    
- 🧩 The `Content-Type` header may be involved in weak file-type validation.
    
- 📂 Always determine **where an uploaded file is stored**.
    
- ⚙️ Determine whether the uploaded file is actually **executed by the server**.
    
- 🐚 A web shell provides command execution through a web interface but is usually less capable than an interactive shell.
    
- 🔄 A web shell can potentially be used as a stepping stone toward a reverse shell.
    
- ⚠️ Web shells can be unstable and leave evidence behind.
    
- 📝 Document payloads, filenames, upload locations, hashes, and successful/failed techniques during an engagement.
    
- 🧠 **Don't memorize the exact bypass — understand the methodology.**
    

> [!success]+ 🎯 CPTS Mental Model
> 
> ```text
> 🌐 Discover Web Application
>          │
>          ▼
> 🐘 Identify Server-Side Language
>          │
>          ▼
> 📤 Find File Upload Function
>          │
>          ▼
> 🔍 Understand Validation
>          │
>          ▼
> 🕵️ Intercept With Burp
>          │
>          ▼
> 🧪 Test Validation
>          │
>          ▼
> 📂 Find Uploaded File
>          │
>          ▼
> ⚙️ Determine If It Executes
>          │
>          ▼
> 🐚 Web Shell
>          │
>          ▼
> 🔄 Upgrade To Better Shell
> ```
> 
> **Upload → Validate → Store → Execute → Shell**