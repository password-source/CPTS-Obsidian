# 🛡️ Firewall & IDS/IPS Evasion

> [!abstract] 🎯 Core Idea  
> **Evasion = understanding how defenders detect and block our activity, then reducing the characteristics that trigger those defenses.**
> 
> Think of the defensive layers as:
> 
> ```text
> Attacker
>    │
>    ▼
> 🌐 Perimeter Defense
>    │
>    ▼
> 🖥️ Endpoint Defense
>    │
>    ▼
> 🎯 Target Process / Service
> ```
> 
> The goal is not simply to "hide the exploit."  
> We must consider **network traffic + files + processes + behavior**.

---

## 1️⃣ 🖥️ Endpoint Protection

**Endpoint protection** protects an individual host.

Examples:

- 🦠 Antivirus (AV)
    
- 🛡️ Antimalware
    
- 🔥 Host-based firewall
    
- 🚫 Anti-ransomware
    
- 🌐 Anti-DDoS
    
- 🕵️ Other endpoint security controls
    

Typical protected endpoints include:

```text
Workstation
Server
Personal computer
DMZ server
```

### 🧠 Mental Model

> **Endpoint protection = security controls directly on the machine we're attacking.**

For example:

```text
Attacker
   │
   ▼
Network
   │
   ▼
Windows Server
   │
   ├── Antivirus
   ├── Host Firewall
   └── EDR
```

---

## 2️⃣ 🌐 Perimeter Protection

**Perimeter protection** exists at the boundary between networks.

Typical examples:

- 🔥 Network firewalls
    
- 🛡️ IDS
    
- 🚨 IPS
    
- 🌐 Edge security appliances
    

A common network design is:

```text
                 🌐 INTERNET
                      │
                      ▼
              🛡️ PERIMETER
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
           🟠 DMZ          🔵 INTERNAL
        Public Servers       Network
```

### 🟠 DMZ

A **DMZ (Demilitarized Zone)** is a network segment between the Internet and the internal network.

Typical DMZ systems:

```text
Web Server
Mail Server
DNS Server
Public API
```

The security relationship is generally:

```text
Internet
   ↓
Lowest trust

DMZ
   ↓
Intermediate trust

Internal Network
   ↓
Higher trust
```

> [!important] 🧠 Remember  
> **DMZ = public-facing systems separated from the more trusted internal network.**

---

# 3️⃣ 📜 Security Policies

Security policies determine what is:

```text
✅ ALLOWED
❌ DENIED
```

This is conceptually similar to **ACLs (Access Control Lists)**.

Policies can apply to different areas:

|Policy|Controls|
|---|---|
|🌐 Network Traffic|Which traffic is allowed|
|🖥️ Application|Which applications/actions are permitted|
|👤 User Access|Which users can access resources|
|📁 File Management|Which files/actions are allowed|
|💥 DDoS Protection|Abnormal traffic volumes|
|🔐 Other Policies|Various security controls|

### 🧠 Core Principle

```text
EVENT
  │
  ▼
Does it match a rule?
  │
 ┌┴──────────┐
 ▼           ▼
ALLOW       DENY
```

The important question for evasion is:

> **What characteristics does the defender use to decide whether an event should be allowed or blocked?**

---

# 4️⃣ 🔎 Detection Methods

There are several major ways security systems identify suspicious activity.

## 4.1 🧬 Signature-Based Detection

The security product looks for **known patterns**.

```text
Incoming data
     │
     ▼
Compare against known signatures
     │
     ├── Match → 🚨 Alert / Block
     │
     └── No match → ✅ Continue
```

Examples of signatures:

- Known malicious byte patterns
    
- Known exploit patterns
    
- Known malware characteristics
    
- Known network attack patterns
    

> [!tip] 🧠 Easy Way to Remember  
> **Signature = "I've seen this exact/similar thing before."**

---

## 4.2 📊 Heuristic / Statistical Anomaly Detection

Instead of looking only for an exact signature, the system establishes a **baseline of normal behavior**.

Then it looks for deviations.

```text
Normal behavior
      │
      ▼
Establish baseline
      │
      ▼
Observe new activity
      │
      ▼
Deviation?
   ┌──┴──┐
  NO     YES
  │       │
  ▼       ▼
Allow    Alert
```

Example:

```text
Normal:
User → DNS → normal domains

Sudden:
User → thousands of unusual DNS requests
```

That deviation may trigger an alert.

> [!important]  
> **Anomaly detection = behavior differs significantly from the established norm.**

---

## 4.3 🔄 Stateful Protocol Analysis

This approach understands how protocols are **supposed to behave**.

It compares observed traffic against expected protocol behavior.

```text
Expected HTTP behavior
        │
        ▼
Compare actual traffic
        │
        ▼
Does it follow protocol expectations?
        │
     ┌──┴──┐
    YES    NO
     │      │
     ▼      ▼
   Allow   Alert
```

> [!tip] 🧠 Think:  
> **Signature = known bad pattern**
> 
> **Anomaly = unusual behavior**
> 
> **Stateful analysis = protocol behaving incorrectly**

---

## 4.4 👨‍💻 SOC-Based Live Monitoring

A **SOC (Security Operations Center)** can monitor security events in real time.

```text
Network / Endpoints
        │
        ▼
   Security Tools
        │
        ▼
     🚨 Alerts
        │
        ▼
      👨‍💻 SOC
        │
        ▼
 Investigation / Response
```

The SOC may:

- Investigate the alert
    
- Determine whether it is malicious
    
- Escalate it
    
- Trigger automated response
    
- Block the activity
    

---

# 5️⃣ 🥷 Evasion Techniques

## 🧠 The Detection Chain

A useful mental model is:

```text
                 ATTACK
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   🌐 Network     📁 File      🧠 Behavior
       │            │            │
      IDS/IPS       AV          EDR/AV
       │            │            │
       └────────────┼────────────┘
                    ▼
                 🚨 ALERT
```

Therefore, evasion can target different stages.

---

# 6️⃣ 🦠 AV Signature Detection

Traditional AV heavily relies on signatures to identify known malicious files.

Conceptually:

```text
Malicious File
      │
      ▼
   AV Scanner
      │
      ▼
Signature Match?
   ┌──┴──┐
  YES    NO
   │      │
   ▼      ▼
 BLOCK   Allow
```

If a malicious payload has a recognizable signature, the AV may:

- Quarantine it
    
- Kill the process
    
- Prevent execution
    

> [!warning]  
> Modern endpoint security is not limited to static signatures. Behavioral detection, reputation, heuristics, memory inspection, and EDR capabilities can also detect malicious activity.

---

# 7️⃣ 🔐 Encrypted Meterpreter Communication

Metasploit can use encrypted communication between Meterpreter and the attacker.

Conceptually:

```text
Attacker
   │
   │ 🔐 Encrypted channel
   ▼
Target
   │
   ▼
Meterpreter
```

This can make **network inspection** more difficult because the payload's communication is not simply visible as plaintext.

> [!important] ⚠️ Important Distinction  
> Encryption can help protect the **contents of traffic**, but it does **not automatically make traffic invisible**.
> 
> Defenders can still inspect:
> 
> - Source/destination IP
>     
> - Destination port
>     
> - Connection frequency
>     
> - TLS/encryption characteristics
>     
> - DNS activity
>     
> - Traffic volume
>     
> - Endpoint behavior
>     

---

# 8️⃣ 🌐 Network Evasion

Sometimes a connection is blocked based on network-level rules.

For example:

```text
Attacker IP
    │
    ▼
Firewall
    │
    ├── ❌ Source blocked
    │
    └── ✅ Allowed
```

In restrictive environments, attackers may look for **legitimate services or communication paths that are permitted by the network policy**.

### 🧠 Key Concept

> **Evasion often means finding a permitted communication path rather than trying to force traffic through a blocked one.**

---

# 9️⃣ 📡 DNS as a Communication Channel

DNS is normally required for ordinary network functionality, making it an interesting channel from a defender's perspective.

Historically, attackers have abused DNS for:

```text
Command & Control
        +
Data Exfiltration
```

Conceptually:

```text
Internal Network
      │
      │ DNS queries
      ▼
Attacker-controlled DNS
      │
      ▼
External Infrastructure
```

> [!warning]  
> DNS exfiltration is detectable. Defenders can monitor unusual:
> 
> - Query frequency
>     
> - Query lengths
>     
> - Random-looking subdomains
>     
> - High-entropy domains
>     
> - Large volumes of DNS requests
>     
> - Requests to suspicious domains
>     

---

# 🔟 💾 Payload Detection Before Execution

Even if the network communication is protected, the payload still has to reach the target.

That creates another detection opportunity:

```text
Payload arrives
      │
      ▼
📁 File on disk
      │
      ▼
AV scans it
      │
      ├── 🚨 Signature detected
      │
      └── ✅ Not detected
             │
             ▼
          Execution
             │
             ▼
      🧠 Memory / Behavior
```

> [!important] 🧠 Mental Model  
> **Getting the payload onto the machine is not the same as successfully executing it.**
> 
> Security controls can inspect it at multiple stages.

---

# 1️⃣1️⃣ 🧩 Executable Templates

`msfvenom` supports using an existing executable as a **template** and embedding a payload into it.

Conceptually:

```text
Legitimate Executable
        +
    Payload
        │
        ▼
Backdoored Executable
```

The idea is to make the resulting file resemble a legitimate application.

The `-x` option specifies an executable template.

The `-k` option is intended to preserve the original executable's execution behavior while running the payload separately.

> [!warning] ⚠️ CPTS Context  
> Template-based payloads are **not a guarantee of AV bypass**.
> 
> Modern security products can detect:
> 
> - Known payloads
>     
> - Suspicious modifications
>     
> - Behavioral activity
>     
> - Memory execution
>     
> - Process injection
>     
> - Abnormal child processes
>     
> - Network connections
>     

---

# 1️⃣2️⃣ 📦 Archives

Another technique discussed is placing a payload inside an archive.

For example:

```text
payload
   │
   ▼
Password-protected archive
   │
   ▼
File may be harder for AV to inspect
```

The important concept is:

> **Encryption/password protection can prevent some scanners from inspecting the contents immediately.**

However, this does **not mean the archive is invisible**.

A security system may detect:

```text
🔒 Encrypted archive
       │
       ▼
Cannot inspect contents
       │
       ▼
🚨 Suspicious / unable to scan
```

The administrator may then manually inspect it.

---

# 1️⃣3️⃣ 📦 Packers

A **packer** compresses/encodes an executable together with decompression code.

Conceptually:

```text
Original Program
      +
Payload
      +
Decompression Stub
      │
      ▼
Packed Executable
      │
      ▼
Execution
      │
      ▼
Unpacking
      │
      ▼
Original Program + Payload
```

The goal historically was to make static inspection more difficult.

Examples:

- UPX
    
- The Enigma Protector
    
- MPRESS
    
- Themida
    
- Morphine
    
- MEW
    

> [!important]  
> **Packer ≠ encryption.**
> 
> Packing primarily transforms/compresses the executable structure. Modern security products can recognize common packers and analyze the unpacked code at runtime.

---

# 1️⃣4️⃣ 💥 Exploit Traffic Evasion

Network IDS/IPS can sometimes recognize exploit traffic by looking for characteristic patterns.

For example, a buffer overflow may contain recognizable:

```text
NOP sled
Shellcode
Buffer patterns
Return address
```

A simplistic detection model could be:

```text
Incoming exploit traffic
        │
        ▼
IDS / IPS
        │
        ▼
Known exploit pattern?
     ┌──┴──┐
    YES    NO
     │      │
     ▼      ▼
   BLOCK   PASS
```

---

# 1️⃣5️⃣ 🎲 Randomization

One way exploit developers historically attempted to avoid simple signatures was to introduce variation into exploit data.

For example:

```text
Exploit A:
[pattern][pattern][pattern]

Exploit B:
[random][pattern][random]

Exploit C:
[pattern][random][pattern]
```

The goal is to make the exploit less predictable to simplistic pattern matching.

Metasploit exploit modules may expose things such as an `Offset` value for exploit-specific memory layout.

> [!warning]  
> Randomization does **not** automatically defeat modern IDS/IPS. Modern detection can use multiple indicators rather than one exact byte sequence.

---

# 1️⃣6️⃣ 🧱 NOP Sleds

A **NOP sled** is a sequence of instructions designed to allow execution to "slide" toward the shellcode.

Simplified memory layout:

```text
┌───────────────────────┐
│       Buffer          │
├───────────────────────┤
│      NOP sled         │
│  NOP NOP NOP NOP ...  │
├───────────────────────┤
│       Shellcode       │
└───────────────────────┘
```

The idea is:

```text
Execution lands somewhere
        │
        ▼
   NOP → NOP → NOP
                 │
                 ▼
             Shellcode
```

> [!important]  
> Historically, predictable NOP sleds could themselves become recognizable to IDS/IPS systems.

---

# 1️⃣7️⃣ 🧠 The Four Main Detection Surfaces

A very useful CPTS mental model:

|Detection Surface|What Defender Can Inspect|
|---|---|
|🌐 **Network**|Packets, ports, IPs, protocols, patterns|
|📁 **File**|Payload signatures, metadata, structure|
|🧠 **Memory/Process**|Runtime behavior and injected code|
|👤 **Behavior**|Commands, connections, unusual activity|

Think:

```text
             ATTACK
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
  NETWORK      FILE      EXECUTION
     │          │          │
   IDS/IPS      AV       EDR/AV
     │          │          │
     └──────────┼──────────┘
                ▼
             SOC 🚨
```

---

# 1️⃣8️⃣ 🧠 Evasion vs Detection — The Mental Model

When attacking a defended target, ask:

```text
1. 🌐 Can my network traffic be detected?
             │
             ▼
2. 📁 Can my payload/file be detected?
             │
             ▼
3. 🧠 Can my process/memory activity be detected?
             │
             ▼
4. 👤 Can my behavior be detected?
```

This gives us a much better methodology than simply thinking:

> ❌ "How do I bypass antivirus?"

Instead think:

> ✅ **"At which stage can the defender observe me, and what detection mechanism are they using?"**

---

# 1️⃣9️⃣ ⚔️ CPTS Takeaways

> [!success] 🎯 What You Should Remember

### 🖥️ Endpoint vs Perimeter

```text
Endpoint Protection
= protects the individual host

Perimeter Protection
= protects the network boundary
```

### 🔍 Detection Types

```text
Signature
    ↓
Known pattern

Heuristic / Anomaly
    ↓
Unusual behavior

Stateful Protocol Analysis
    ↓
Protocol doesn't behave as expected

SOC Monitoring
    ↓
Human analysts investigate events
```

### 🥷 Evasion

```text
Network
  ↓
Encrypted / permitted communication paths

File
  ↓
Reduce static detection characteristics

Execution
  ↓
Understand runtime detection

Exploit
  ↓
Avoid predictable patterns
```

### 📦 Archives

Password-protected archives can prevent some scanners from inspecting their contents immediately, but may themselves generate alerts.

### 🧩 Packers

Packers transform/compress executables and can make static analysis harder, but modern security products can detect common packers and analyze runtime behavior.

### 💥 Exploit Randomization

Variation in exploit patterns can defeat **simple signatures**, but modern IDS/IPS can use multiple indicators.

---

> [!tip] 🧠 Final Mental Model
> 
> **Evasion is a cat-and-mouse game between attacker characteristics and defender detection.**
> 
> ```text
>             ATTACK
>                │
>                ▼
>       What can expose me?
>                │
>       ┌────────┼────────┐
>       ▼        ▼        ▼
>     Network   File    Behavior
>       │        │        │
>      IDS      AV      EDR/SOC
>       │        │        │
>       └────────┼────────┘
>                ▼
>              ALERT
> ```
> 
> The key CPTS skill is recognizing **where detection occurs and why**, rather than memorizing individual "AV bypass" tricks.