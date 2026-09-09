# 🐛 1. Latest FTP Vulnerabilities

> [!info]  
> This section demonstrates how a **specific FTP vulnerability** can be understood using the previously introduced **Source → Process → Privileges → Destination (SPPD)** attack model.
> 
> The example is **CoreFTP before build 727**, affected by **CVE-2022-22836**.
> 
> The vulnerability involves:
> 
> - **Authenticated directory/path traversal**
>     
> - **Arbitrary file write**
>     
> - Improper processing of HTTP `PUT` requests
>     
> 
> The result is the ability to write files **outside the directory that the FTP service is supposed to have access to**.

---

# 🎯 2. CVE-2022-22836 — CoreFTP

**Affected:** CoreFTP before build `727`

**CVE:** `CVE-2022-22836`

### 🔥 Vulnerability Chain

```text
Authenticated FTP User
        │
        ▼
HTTP PUT Request
        │
        ▼
Directory Traversal
../../../../
        │
        ▼
Escape Restricted Directory
        │
        ▼
Arbitrary File Write
        │
        ▼
Write Content Anywhere
the service can reach
```

> [!important]  
> The important concept is not simply **"FTP allows file uploads."**
> 
> The vulnerability occurs because the application **incorrectly processes the requested path**, allowing `../` traversal to escape the intended directory.

---

# 🧠 3. Understanding the Vulnerability

CoreFTP normally uses an HTTP `POST` request for uploading files.

However, the vulnerable service also accepts an HTTP `PUT` request that can be manipulated to write content to a specified file.

The attacker combines:

```text
HTTP PUT
   +
Valid credentials
   +
Directory traversal
   +
Attacker-controlled content
```

to achieve arbitrary file writing.

### 🧩 Core Concept

```text
Normal:

/authorized/directory/file.txt
              │
              ▼
       Allowed location


Vulnerable:

/authorized/directory/../../../../whoops
                         │
                         ▼
                 Escape directory
                         │
                         ▼
                    C:\whoops
```

---

# 💻 4. CoreFTP Exploitation

The exploit can be performed using a single `curl` request:

```bash
curl -k -X PUT -H "Host: <IP>" --basic -u <username>:<password> --data-binary "PoC." --path-as-is https://<IP>/../../../../../../whoops
```

### 🔎 Breaking Down the Command

|Option|Purpose|
|---|---|
|`-k`|Allows HTTPS connections without verifying the certificate|
|`-X PUT`|Explicitly sends an HTTP `PUT` request|
|`-H "Host: <IP>"`|Sets the HTTP `Host` header to the target IP|
|`--basic`|Uses HTTP Basic Authentication|
|`-u <username>:<password>`|Supplies the authenticated credentials|
|`--data-binary "PoC."`|Sends `PoC.` as the file contents|
|`--path-as-is`|Prevents `curl` from normalizing the path|
|`https://<IP>/../../../../../../whoops`|Uses directory traversal to target a file outside the expected directory|

> [!important]  
> **`--path-as-is` is particularly important here.**
> 
> The traversal characters `../` need to reach the vulnerable server/application as supplied. If the client normalizes the path first, the traversal payload may not behave as intended.

---

# 🧬 5. SPPD — Source → Process → Privileges → Destination

This vulnerability is a good example of the **Concept of Attacks** methodology.

```text
SOURCE
  │
  │ User-controlled HTTP request
  │ PUT + path + file content
  ▼
PROCESS
  │
  │ Application processes the request
  ▼
PRIVILEGES
  │
  │ Path restrictions are bypassed
  ▼
DESTINATION
  │
  │ File is written to attacker-selected location
  ▼
TARGET FILE
```

> [!tip]  
> When analyzing a vulnerability, ask four questions:
> 
> **1. What is the attacker controlling?** → Source  
> **2. What processes that input?** → Process  
> **3. What security boundary is being bypassed?** → Privileges  
> **4. What does the input ultimately affect?** → Destination

---

# 📂 6. Directory Traversal

The first stage is escaping the restricted directory using traversal sequences such as:

```text
../../../../
```

Each:

```text
../
```

means:

> Move to the parent directory.

Repeated traversal can therefore move the requested path outside the directory intended by the application.

---

## 🧠 6.1 Directory Traversal — SPPD Breakdown

|**StepDirectory TraversalConcept of Attacks - Category**|||
|---|---|---|
|`1.`|The user specifies the type of HTTP request with the file's content, including escaping characters to break out of the restricted area.|`Source`|
|`2.`|The changed type of HTTP request, file contents, and path entered by the user are taken over and processed by the process.|`Process`|
|`3.`|The application checks whether the user is authorized to be in the specified path. Since the restrictions only apply to a specific folder, all permissions granted to it are bypassed as it breaks out of that folder using the directory traversal.|`Privileges`|
|`4.`|The destination is another process that has the task of writing the specified contents of the user on the local system.|`Destination`|

### 🧠 What Actually Went Wrong?

The application was supposed to enforce something like:

```text
        Allowed Directory
              │
              ▼
       /authorized/
              │
              ├── file1
              ├── file2
              └── file3
```

But the attacker supplies:

```text
/authorized/../../../../whoops
```

The application fails to properly enforce the boundary:

```text
/authorized/
     │
     │ ../../../../
     ▼
Outside authorized directory
```

> [!warning]  
> Directory traversal is fundamentally a **boundary-validation problem**.
> 
> The application believes the user is restricted to one directory, but the attacker manipulates the path so the final filesystem location is somewhere else.

---

# ✍️ 7. Arbitrary File Write

After escaping the restricted directory, the second part of the vulnerability is **arbitrary file write**.

The attacker controls:

```text
Filename:
whoops

Content:
PoC.
```

The vulnerable service then writes:

```text
C:\whoops
```

containing:

```text
PoC.
```

---

# 🧬 8. Arbitrary File Write — SPPD Breakdown

|**StepArbitrary File WriteConcept of Attacks - Category**|||
|---|---|---|
|`5.`|The same information that the user entered is used as the source. In this case, the filename (`whoops`) and the contents (`--data-binary "PoC."`).|`Source`|
|`6.`|The process takes the specified information and proceeds to write the desired content to the specified file.|`Process`|
|`7.`|Since all restrictions were bypassed during the directory traversal vulnerability, the service approves writing the contents to the specified file.|`Privileges`|
|`8.`|The filename specified by the user (`whoops`) with the desired content (`"PoC."`) now serves as the destination on the local system.|`Destination`|

---

# 🔗 9. Putting Both Vulnerabilities Together

The two stages should be understood as **one attack chain**:

```text
        🧑 Attacker
             │
             │ Authenticated credentials
             ▼
      ┌───────────────┐
      │    CoreFTP    │
      └───────┬───────┘
              │
              │ HTTP PUT
              ▼
      ┌────────────────┐
      │ Path Traversal │
      │    ../../..    │
      └───────┬────────┘
              │
              ▼
     Escape restricted path
              │
              ▼
      ┌────────────────┐
      │ Arbitrary File │
      │     Write      │
      └───────┬────────┘
              │
              ▼
        C:\whoops
              │
              ▼
           "PoC."
```

### 🧠 The Critical Relationship

```text
Directory Traversal
        ↓
Bypass intended filesystem boundary
        ↓
Arbitrary File Write
        ↓
Choose filename/location
        ↓
Choose file contents
```

> [!important]  
> **Directory traversal is what defeats the location restriction.**
> 
> **Arbitrary file write is what allows attacker-controlled content to be placed there.**
> 
> Together, they turn an apparently restricted upload function into a much more powerful primitive.

---

# 🖥️ 10. Verifying the File on the Target

After the exploit succeeds, the file can be found on the target system.

```cmd
C:\> type C:\whoops
```

Output:

```text
PoC.
```

This confirms:

```text
HTTP PUT
   ↓
Traversal accepted
   ↓
Restricted directory escaped
   ↓
File created
   ↓
Attacker-controlled content written
```

---

# 🎯 11. Why Arbitrary File Write Matters

An arbitrary file write vulnerability does **not automatically equal RCE**.

The important question is:

> **"What can I do with the ability to write a file to this location?"**

Potential attack chains depend on what consumes the file.

For example:

```text
Arbitrary File Write
        │
        ├──► Configuration file
        │         ↓
        │      Modify behavior
        │
        ├──► Web-accessible script
        │         ↓
        │      Potential execution
        │
        ├──► Startup/service-related file
        │         ↓
        │      Potential execution
        │
        └──► Sensitive location
                  ↓
              Impact depends
              on permissions
```

> [!warning]  
> The source example only demonstrates **arbitrary file creation** with the contents `PoC.`. It does **not** demonstrate a complete RCE chain.
> 
> For CPTS methodology, remember to separate:
> 
> **File Write ≠ Code Execution**
> 
> You need an additional condition that causes the written file to be executed, loaded, or otherwise interpreted.

---

# 🔍 12. Vulnerability Analysis Methodology

When you encounter a service vulnerability, don't memorize only the exploit command.

Break it into its underlying mechanism:

```text
🔎 INPUT
   │
   ▼
What can the attacker control?
   │
   ├─ HTTP method
   ├─ File path
   └─ File contents
   │
   ▼
🧠 PROCESSING
   │
   ▼
How does the application process that input?
   │
   ▼
🔐 SECURITY CHECK
   │
   ▼
What restriction should have stopped us?
   │
   ▼
💥 BYPASS
   │
   ▼
What happens after the restriction is bypassed?
   │
   ▼
📂 IMPACT
   │
   ▼
Where can attacker-controlled data go?
```

This is the same **SPPD** methodology:

```text
Source
  ↓
Process
  ↓
Privileges
  ↓
Destination
```

---

# ⚡ 13. CPTS Cheat Sheet — CoreFTP CVE-2022-22836

|Finding|Meaning|Next Step|
|---|---|---|
|CoreFTP before build `727`|Potentially vulnerable version|Investigate `CVE-2022-22836`|
|Valid FTP credentials|Authentication requirement satisfied|Test authenticated functionality|
|HTTP `PUT` accepted|File-writing functionality may be available|Investigate PUT behavior|
|`../` accepted in path|Potential path traversal|Test directory boundary|
|File written outside authorized directory|Arbitrary file write confirmed|Determine security impact|
|Attacker controls file contents|Can place arbitrary data|Identify useful target locations|
|Written file is interpreted by another service|Potential execution primitive|Investigate further|
|Written file is only stored|File-write primitive|Look for another way to leverage it|

---

# 🧠 14. Exam Mental Model

### Remember the vulnerability as:

```text
          🔑 AUTHENTICATED USER
                   │
                   ▼
             HTTP PUT
                   │
                   ▼
          ../../../../
          DIRECTORY TRAVERSAL
                   │
                   ▼
        🚪 ESCAPE RESTRICTION
                   │
                   ▼
       ✍️ ARBITRARY FILE WRITE
                   │
                   ▼
       📄 ATTACKER-CONTROLLED FILE
                   │
                   ▼
          🔎 LOOK FOR IMPACT
```

> [!success]
> 
> ### 🏆 Key Takeaway
> 
> **CVE-2022-22836 = authenticated path traversal + arbitrary file write in vulnerable CoreFTP.**
> 
> The most important CPTS lesson is the methodology:
> 
> **Attacker-controlled input → vulnerable processing → security boundary bypass → attacker-controlled destination.**
> 
> Don't stop at identifying the CVE. Understand **what input is controlled, what validation fails, what privilege boundary is bypassed, and what the resulting primitive allows you to do.**