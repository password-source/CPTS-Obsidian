# 🐚 Spawning Interactive Shells

> [!info]+ 🎯 Objective  
> During a penetration test, we may obtain a shell that is **limited**, unstable, or missing normal interactive functionality.
> 
> A common example is:
> 
> ```text
> 💥 Exploit
>      ↓
> 🐚 Limited Shell
>      ↓
> ❌ No proper prompt / limited commands
>      ↓
> 🔧 Spawn Interactive Shell
>      ↓
> 🖥️ More Functional Terminal
> ```
> 
> In the previous section, Python's `pty.spawn()` was used to improve a limited shell.
> 
> But what happens when **Python isn't installed?**
> 
> We need alternative methods.

---

# 🧠 1️⃣ What Is an Interactive Shell?

An **interactive shell** gives us a more usable command-line environment where we can interact with the operating system normally.

A limited shell may look like:

```text
id="v9tq1f"
❌ No useful prompt
❌ Limited terminal functionality
❌ Interactive programs may fail
❌ `su` / `sudo` may not work properly
```

A more functional shell looks like:

```text
id="g8k3zw"
sh-4.2$
```

or:

```text
id="4q9mrx"
bash-5.1$
```

### 🎯 Goal

```text
LIMITED SHELL
     │
     ▼
FIND AVAILABLE INTERPRETER / TOOL
     │
     ▼
SPAWN /bin/sh OR /bin/bash
     │
     ▼
INTERACTIVE SHELL
```

> [!important]+ 🧠 Remember  
> Whenever you see:
> 
> ```text
> /bin/sh
> /bin/bash
> ```
> 
> these paths can potentially be replaced with another shell interpreter available on the target.
> 
> Common Linux shells include:
> 
> - `/bin/sh` → Bourne shell
>     
> - `/bin/bash` → Bourne Again Shell
>     

---

# 🔄 2️⃣ The Shell-Spawning Mindset

Don't memorize every technique as a completely separate trick.

Instead, think:

```text
🎯 I have a limited shell
          │
          ▼
🔎 What interpreters/tools exist?
          │
     ┌────┼────┬────┬────┐
     ▼    ▼    ▼    ▼    ▼
   Python Perl Ruby Lua  AWK
     │    │    │    │    │
     └────┴────┴────┴────┘
               │
               ▼
          🐚 /bin/sh
```

The general idea is:

> **Use something already available on the target to execute a shell interpreter.**

---

# 🐚 3️⃣ `/bin/sh -i`

The simplest method is to execute `/bin/sh` with the **interactive** flag.

```bash
/bin/sh -i
```

### 🔎 What Does `-i` Mean?

```text
-i
│
└── Interactive mode
```

Example:

```text
sh: no job control in this shell
sh-4.2$
```

The important part is that we now have a shell prompt:

```text
sh-4.2$
```

> [!warning]+ ⚠️ "No Job Control"  
> You may see:
> 
> ```text
> sh: no job control in this shell
> ```
> 
> This does **not necessarily mean the shell failed**.
> 
> You may still be able to execute commands.
> 
> It simply means the shell does not have normal terminal job-control functionality.

---

# 🐪 4️⃣ Perl

If **Perl** is installed, it can be used to execute `/bin/sh`.

Check whether Perl exists:

```bash
which perl
```

Then:

```bash
perl -e 'exec "/bin/sh";'
```

### 🧠 What Happens?

```text
perl
 │
 ▼
-e
 │
 ▼
Execute Perl code
 │
 ▼
exec "/bin/sh"
 │
 ▼
🐚 /bin/sh
```

The important concept is:

```text
Perl → execute → /bin/sh
```

> [!tip]+ 💡 Script vs One-Liner  
> The material also shows:
> 
> ```text
> perl: exec "/bin/sh";
> ```
> 
> This form belongs inside a Perl script rather than being used directly as the shell command.

---

# 💎 5️⃣ Ruby

If **Ruby** is available:

```bash
which ruby
```

Ruby can execute the shell interpreter:

```text
ruby: exec "/bin/sh"
```

The command shown above is intended to be placed inside a Ruby script.

### 🧩 Concept

```text
Ruby
 │
 ▼
exec
 │
 ▼
/bin/sh
 │
 ▼
🐚 Shell
```

The important thing isn't memorizing Ruby syntax — it's recognizing that **an installed programming language can sometimes be used to execute another program**.

---

# 🌙 6️⃣ Lua

If **Lua** is installed:

```bash
which lua
```

Lua provides the `os.execute()` function.

Example:

```text
lua: os.execute('/bin/sh')
```

This is intended to be run from a Lua script.

### 🧠 Concept

```text
Lua
 │
 ▼
os.execute()
 │
 ▼
/bin/sh
 │
 ▼
🐚 Shell
```

---

# 📝 7️⃣ AWK

**AWK** is a pattern-processing language commonly found on Unix/Linux systems.

It is frequently used by administrators and developers for processing text and generating reports.

However, AWK can also execute external commands.

Example:

```bash
awk 'BEGIN {system("/bin/sh")}'
```

### 🔎 Breaking It Down

```text
awk
 │
 ▼
BEGIN { ... }
 │
 ▼
system("/bin/sh")
 │
 ▼
Execute /bin/sh
 │
 ▼
🐚 Shell
```

> [!important]+ 🧠 Key Concept  
> AWK isn't being used here for text processing.
> 
> We're taking advantage of its ability to execute an external command.

---

# 🔎 8️⃣ Using `find` to Spawn a Shell

The Linux `find` command is primarily used to search for files and directories.

However, `find` also has an `-exec` option that can execute commands.

This means it can sometimes be used to launch a shell.

Example:

```bash
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
```

### 🧩 What's Happening?

```text
find /
 │
 ▼
Search filesystem
 │
 ▼
-name nameoffile
 │
 ▼
When matching file is found
 │
 ▼
-exec /bin/awk
 │
 ▼
AWK executes /bin/sh
 │
 ▼
🐚 Shell
```

The important piece to recognize is:

```text
-exec
```

> [!important]+ 💡 `find -exec`  
> `find` isn't inherently a shell-spawning tool.
> 
> The important capability is that `find` can **execute another command** using `-exec`.
> 
> Therefore:
> 
> ```text
> find → -exec → another executable → shell
> ```

---

# ⚡ 9️⃣ Using `find` Directly With `-exec`

We can also use `find` to execute `/bin/sh` directly:

```bash
find . -exec /bin/sh \; -quit
```

### 🔎 Breakdown

|Component|Purpose|
|---|---|
|`find .`|Search from the current directory|
|`-exec`|Execute a command|
|`/bin/sh`|Program to execute|
|`\;`|Marks the end of the `-exec` command|
|`-quit`|Stop searching after execution|

The flow is:

```text
find .
  │
  ▼
-exec
  │
  ▼
/bin/sh
  │
  ▼
🐚 Shell
  │
  ▼
-quit
```

> [!warning]+ ⚠️ Important  
> This technique depends on `find` actually processing an entry.
> 
> If `find` doesn't reach an execution condition, you may not obtain a shell.

---

# 📝 🔟 Vim

Vim is primarily a command-line text editor.

However, Vim can execute external commands.

That means it can potentially be used to launch a shell.

### 🚪 Method 1 — Vim Command

```bash
vim -c ':!/bin/sh'
```

The important part is:

```text
:! /bin/sh
```

The `!` command tells Vim to execute an external command.

---

# 📝 1️⃣1️⃣ Vim Shell Escape

Another method is to start Vim:

```bash
vim
```

Then inside Vim:

```vim
:set shell=/bin/sh
:shell
```

### 🧠 What's Happening?

```text
vim
 │
 ▼
:set shell=/bin/sh
 │
 ▼
Tell Vim which shell to use
 │
 ▼
:shell
 │
 ▼
🐚 /bin/sh
```

> [!tip]+ 💡 When Is This Useful?  
> This is a relatively niche technique.
> 
> You probably won't use Vim as your first choice, but if you happen to obtain access to Vim on a restricted system, knowing that it can execute a shell can be valuable.

---

# 🧰 1️⃣2️⃣ Shell-Spawning Cheat Sheet

|Available Tool|Method|Result|
|---|---|---|
|`/bin/sh`|`/bin/sh -i`|Interactive `sh`|
|Perl|`perl -e 'exec "/bin/sh";'`|`/bin/sh`|
|Ruby|`exec "/bin/sh"`|`/bin/sh`|
|Lua|`os.execute('/bin/sh')`|`/bin/sh`|
|AWK|`awk 'BEGIN {system("/bin/sh")}'`|`/bin/sh`|
|Find|`find . -exec /bin/sh \; -quit`|`/bin/sh`|
|Vim|`vim -c ':!/bin/sh'`|`/bin/sh`|

> [!success]+ 🧠 The Pattern  
> You don't need to memorize every individual command immediately.
> 
> Remember the **concept**:
> 
> ```text
> AVAILABLE TOOL
>       │
>       ▼
> CAN IT EXECUTE COMMANDS?
>       │
>       ▼
> EXECUTE /bin/sh
>       │
>       ▼
> 🐚 INTERACTIVE SHELL
> ```

---

# 🔐 1️⃣3️⃣ Don't Forget Permissions

Getting an interactive shell does **not** mean we have high privileges.

We must always determine:

```text
👤 Who am I?
🔐 What can I execute?
📁 What files can I access?
👑 Can I use sudo?
```

For a particular file or binary, inspect its permissions with:

```bash
ls -la <path/to/fileorbinary>
```

Example:

```text
-rwxr-xr-x 1 root root ...
```

### 🧠 What Are We Looking For?

```text
-rwxr-xr-x
 │ │ │ │
 │ │ │ └── Others
 │ │ └──── Group
 │ └────── Owner
 └──────── File type
```

Understanding permissions helps determine what actions our current account can perform.

---

# 👑 1️⃣4️⃣ Check `sudo` Permissions

One of the most important commands after obtaining a Linux shell is:

```bash
sudo -l
```

This displays the commands the current user is allowed to execute through `sudo`.

Example:

```text
Matching Defaults entries for apache on ILF-WebSrv:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User apache may run the following commands on ILF-WebSrv:
    (ALL : ALL) NOPASSWD: ALL
```

The important line is:

```text
User apache may run the following commands:
    (ALL : ALL) NOPASSWD: ALL
```

This indicates that the `apache` account has extremely powerful `sudo` permissions in the lab environment.

---

# 🚨 1️⃣5️⃣ Why `sudo -l` Matters

Think of `sudo -l` as asking the system:

> **"What commands am I allowed to run with elevated privileges?"**

The result can reveal potential privilege-escalation paths.

```text
🐚 Current Shell
      │
      ▼
👤 Current User
      │
      ▼
sudo -l
      │
      ▼
🔎 Allowed sudo commands
      │
      ▼
❓ Can any allowed command lead to higher privileges?
```

> [!important]+ 👑 Privilege Escalation Connection  
> Shell spawning and privilege escalation are closely connected.
> 
> Obtaining a shell gives us a foothold.
> 
> Checking permissions tells us what we can potentially do with that foothold.
> 
> ```text
> 🐚 Shell
>    ↓
> 👤 User
>    ↓
> 🔎 Permissions
>    ↓
> `sudo -l`
>    ↓
> 🔓 Potential Privilege Escalation
> ```

---

# ⚠️ 1️⃣6️⃣ Stable Shell Required for `sudo -l`

The material makes an important point:

> [!warning]+ 🐚 Shell Stability Matters  
> `sudo -l` may require a **stable interactive shell**.
> 
> If you're sitting in a very limited or unstable shell, the command may:
> 
> - ❌ Produce no useful output
>     
> - ❌ Behave unexpectedly
>     
> - ❌ Fail to interact properly
>     
> 
> This is another reason why upgrading a limited shell can be important.

The practical workflow becomes:

```text
💥 Gain Initial Access
       ↓
🐚 Limited Shell
       ↓
🔧 Upgrade / Spawn Interactive Shell
       ↓
👤 Identify User
       ↓
🔐 Check Permissions
       ↓
sudo -l
       ↓
🔎 Look for Privilege Escalation
```

---

# 🧠 1️⃣7️⃣ CPTS Shell-Spawning Methodology

When you land on a Linux target with a limited shell, don't immediately panic or start trying random commands.

Use this methodology:

```text
                    🐚 LIMITED SHELL
                           │
                           ▼
                 🔎 WHAT IS AVAILABLE?
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
       Python            Perl             Ruby
          │                │                │
          ├──────────── Lua ────────────────┤
          │                │                │
          └──────────── AWK ────────────────┘
                           │
                           ▼
                    Can execute `/bin/sh`?
                           │
                           ▼
                    🐚 INTERACTIVE SHELL
                           │
                           ▼
                       `whoami`
                           │
                           ▼
                    🔐 `sudo -l`
                           │
                           ▼
                    📁 Check permissions
                           │
                           ▼
                 🔎 Privilege Escalation
```

### 🔍 First Things to Check

```bash
which python
which perl
which ruby
which lua
which awk
which vim
```

Then choose an available method.

> [!tip]+ 🎯 Practical Priority  
> You don't need to blindly try every technique.
> 
> A reasonable thought process is:
> 
> ```text
> Python available?
>      ↓
> Use Python
> 
> Python unavailable?
>      ↓
> Check Perl / Ruby / Lua / AWK
> 
> None available?
>      ↓
> Look for other command-execution capabilities
> ```
> 
> The exact method depends on what is installed on the target.

---

# 🔄 1️⃣8️⃣ Complete Mental Model

```text
                 🎯 COMPROMISED LINUX HOST
                            │
                            ▼
                       🐚 LIMITED SHELL
                            │
                            ▼
                 🔎 ENUMERATE AVAILABLE TOOLS
                            │
          ┌─────────┬────────┼────────┬─────────┐
          ▼         ▼        ▼        ▼         ▼
        Python     Perl     Ruby     Lua       AWK
          │         │        │        │         │
          └─────────┴────────┴────────┴─────────┘
                            │
                            ▼
                     Execute `/bin/sh`
                            │
                            ▼
                     🖥️ INTERACTIVE SHELL
                            │
                            ▼
                         `whoami`
                            │
                            ▼
                    👤 Identify User
                            │
                            ▼
                         `sudo -l`
                            │
                            ▼
                    🔐 Check Privileges
                            │
                            ▼
                    📁 Check Permissions
                            │
                            ▼
                    🔓 Find Escalation
                            │
                            ▼
                         👑 ROOT
```

---

# 📌 1️⃣9️⃣ Quick Reference

### 🐚 `/bin/sh`

```bash
/bin/sh -i
```

### 🐪 Perl

```bash
perl -e 'exec "/bin/sh";'
```

### 💎 Ruby

```text
exec "/bin/sh"
```

### 🌙 Lua

```text
os.execute('/bin/sh')
```

### 📝 AWK

```bash
awk 'BEGIN {system("/bin/sh")}'
```

### 🔎 Find

```bash
find . -exec /bin/sh \; -quit
```

### 📝 Vim

```bash
vim -c ':!/bin/sh'
```

Or inside Vim:

```vim
:set shell=/bin/sh
:shell
```

### 🔐 Check File Permissions

```bash
ls -la <path/to/fileorbinary>
```

### 👑 Check Sudo Permissions

```bash
sudo -l
```

### 🔎 Check Available Interpreters

```bash
which python
which perl
which ruby
which lua
which awk
which vim
```

---

# 🎯 2️⃣0️⃣ Key Takeaways

- 🐚 A compromised Linux system may initially provide only a **limited/non-interactive shell**.
    
- 🔧 Python is one way to spawn a more functional shell, but it may not always be installed.
    
- 🧠 Other installed interpreters and programs can sometimes be used to execute `/bin/sh`.
    
- 🐪 **Perl**, 💎 **Ruby**, 🌙 **Lua**, and 📝 **AWK** can potentially execute a shell.
    
- 🔎 `find` has an `-exec` capability that can execute commands.
    
- 📝 **Vim** can execute external commands and can therefore be used to launch a shell.
    
- 🐚 `/bin/sh -i` explicitly requests interactive shell mode.
    
- ⚠️ `no job control` does not necessarily mean the shell failed.
    
- 🔐 After obtaining a shell, always investigate the privileges of the current user.
    
- 📁 `ls -la` helps inspect file ownership and permissions.
    
- 👑 `sudo -l` shows what commands the current account can execute through `sudo`.
    
- 🖥️ A **stable interactive shell** can make enumeration and privilege escalation much easier.
    
- 🎯 Shell spawning is not the end goal — it is about turning a limited foothold into a **more useful working environment**.
    

> [!success]+ 🧠 FINAL CPTS MENTAL MODEL
> 
> ```text
> 🐚 LIMITED SHELL
>        │
>        ▼
> 🔎 CHECK WHAT EXISTS
>        │
>        ▼
> 🧰 USE AVAILABLE TOOL
>        │
>        ▼
> 🖥️ SPAWN INTERACTIVE SHELL
>        │
>        ▼
> 👤 `whoami`
>        │
>        ▼
> 🔐 `sudo -l`
>        │
>        ▼
> 📁 CHECK PERMISSIONS
>        │
>        ▼
> 🔎 ENUMERATE
>        │
>        ▼
> 🔓 PRIVILEGE ESCALATION
>        │
>        ▼
> 👑 ROOT
> ```
> 
> **The key mindset:**
> 
> **Don't ask only "How do I get a shell?"**
> 
> Ask:
> 
> **"What shell do I have, what can it do, what tools are available, and how can I turn this foothold into useful access?"**