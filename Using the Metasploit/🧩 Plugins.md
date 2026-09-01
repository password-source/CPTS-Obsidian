# 🧩 Plugins

---

## 🎯 What Are Metasploit Plugins?

**Metasploit Plugins** are third-party extensions that integrate additional functionality directly into **Metasploit Framework / `msfconsole`**.

They can:

- 🔗 Integrate external security tools
    
- 🤖 Automate repetitive tasks
    
- ➕ Add new commands to `msfconsole`
    
- 🗃️ Automatically interact with the Metasploit database
    
- 🔍 Import and manage results from other tools
    
- ⚙️ Extend the functionality of the Metasploit Framework
    

🧠 **Mental Model:**

> **Modules = perform specific security tasks**  
> **Plugins = extend what Metasploit itself can do**

For example:

```text
                    Metasploit
                       │
             ┌─────────┴─────────┐
             │                   │
          Modules             Plugins
             │                   │
     Exploits / Payloads    Extend Framework
     Auxiliary / Post       Add commands
     Encoders / NOPs        Integrate tools
```

---

# 📂 Where Are Plugins Stored?

The default plugin directory is:

```bash
/usr/share/metasploit-framework/plugins
```

List installed plugins:

```bash
ls /usr/share/metasploit-framework/plugins
```

Example:

```text
aggregator.rb
db_credcollect.rb
nessus.rb
nexpose.rb
openvas.rb
sqlmap.rb
wmap.rb
...
```

💡 Metasploit plugins are generally **Ruby (`.rb`) files**.

---

# 🚀 Loading a Plugin

Plugins are loaded from inside `msfconsole` using:

```text
load <plugin>
```

### Example: Nessus

```text
msf6 > load nessus

[*] Nessus Bridge for Metasploit
[*] Type nessus_help for a command listing
[*] Successfully loaded Plugin: Nessus
```

Once loaded, the plugin can add its own commands.

For example:

```text
msf6 > nessus_help
```

This displays commands provided by the Nessus plugin.

🧠 **Important:**

```text
load nessus
     │
     ▼
Metasploit loads nessus.rb
     │
     ▼
New Nessus-specific commands become available
```

---

# ❌ Plugin Loading Errors

If the plugin doesn't exist in the plugin directory:

```text
msf6 > load Plugin_That_Does_Not_Exist

[-] Failed to load plugin from
/usr/share/metasploit-framework/plugins/Plugin_That_Does_Not_Exist.rb
```

This usually means:

- The plugin isn't installed.
    
- The filename is incorrect.
    
- The plugin isn't located in the expected directory.
    

---

# 🛠️ Installing Custom Plugins

Custom plugins can be obtained from their developers and placed inside:

```bash
/usr/share/metasploit-framework/plugins
```

A common workflow is:

```text
Download plugin
      │
      ▼
Obtain .rb file
      │
      ▼
Copy to Metasploit plugins directory
      │
      ▼
Start msfconsole
      │
      ▼
load <plugin>
      │
      ▼
Use plugin commands
```

### Example

Clone a plugin repository:

```bash
git clone https://github.com/darkoperator/Metasploit-Plugins
```

Copy a plugin:

```bash
sudo cp ./Metasploit-Plugins/pentest.rb \
/usr/share/metasploit-framework/plugins/pentest.rb
```

Start Metasploit:

```bash
msfconsole -q
```

Load the plugin:

```text
msf6 > load pentest
```

If successful:

```text
[*] Successfully loaded plugin: pentest
```

---

# ➕ Plugins Can Add New Commands

After loading a plugin, the normal:

```text
help
```

command can contain additional command categories.

For example, the `pentest` plugin can add functionality such as:

```text
Tradecraft Commands
    check_footprint

auto_exploit Commands
    show_client_side
    vuln_exploit

Discovery Commands
    discover_db
    network_discover
    pivot_network_discover
    show_session_networks

Postauto Commands
    app_creds
    get_lhost
    multi_cmd
    multi_meter_cmd
    multi_post
    sys_creds
```

🧠 **Key idea:**

> Loading a plugin can effectively give `msfconsole` an entirely new set of commands.

---

# 🔌 Common Metasploit Plugin Integrations

Some examples include:

|Plugin|Purpose|
|---|---|
|🗺️ Nmap|Network scanning integration|
|🔍 Nessus|Vulnerability scanner integration|
|🛡️ Nexpose|Vulnerability management/scanning integration|
|🔎 OpenVAS|Vulnerability scanner integration|
|🐍 SQLMap|SQL injection tooling integration|
|🌐 WMAP|Web application assessment|
|🔑 Mimikatz|Credential/post-exploitation functionality|
|⚙️ Railgun|Windows API interaction|
|🔄 DarkOperator plugins|Additional automation/utilities|

> ⚠️ Availability and included plugins can vary between Metasploit installations and versions.

---

# 🧠 Plugins vs Modules

This distinction is **very important**.

|Feature|Modules|Plugins|
|---|---|---|
|Purpose|Perform security tasks|Extend Metasploit|
|Examples|Exploit, Payload, Auxiliary|Nessus, SQLMap, custom plugins|
|Loaded with|`use`|`load`|
|Changes `msfconsole` commands?|Usually no|✅ Yes|
|Extends framework functionality?|❌|✅|
|Written in|Primarily Ruby|Ruby|

### Easy Memory Trick

```text
MODULE
   ↓
"Do something to the target"

PLUGIN
   ↓
"Add something to Metasploit"
```

---

# 🧬 Mixins

Metasploit is written primarily in **Ruby**, an object-oriented programming language.

**Mixins** are a Ruby programming concept that allows classes to include functionality from other modules without using traditional inheritance.

In Ruby, this is commonly done using:

```ruby
include SomeModule
```

### Why Mixins Exist

They are useful when:

1. A class needs many **optional features**.
    
2. The same functionality needs to be shared across **many different classes**.
    

🧠 Think of a mixin as a **feature package** that a class can include.

```text
             Feature A
                │
             Feature B
                │
                ▼
          ┌─────────────┐
          │    Class    │
          └─────────────┘
                ▲
                │
             Feature C
```

Instead of forcing everything into one inheritance hierarchy, Ruby allows classes to **include reusable functionality**.

---

# ⚠️ CPTS Perspective on Mixins

If you're currently learning Metasploit for the **CPTS**, you don't need to deeply understand Ruby Mixins to operate Metasploit effectively.

The important takeaway is simply:

> **Mixins are a Ruby mechanism used by Metasploit developers to share reusable functionality between classes.**

You will encounter them more when examining the **internal structure and customization of Metasploit**.

---

# 🧠 Metasploit Plugin Mental Model

Think of Metasploit as a platform that can be extended:

```text
                         MSFCONSOLE
                             │
             ┌───────────────┼────────────────┐
             │               │                │
          Modules          Plugins          Database
             │               │                │
       ┌─────┼─────┐     ┌───┼────┐       PostgreSQL
       │     │     │     │   │    │
    Exploit Payload Post  Nessus SQLMap WMAP
       │
       ▼
   Attack / Assess
```

### 🔑 Remember

**`use` → select a module**

```text
use exploit/...
```

**`load` → load a plugin**

```text
load nessus
```

**`help` → discover available commands**

```text
help
```

**Plugin-specific help → learn its commands**

```text
nessus_help
```

---

> [!important] 🎯 CPTS Key Takeaways
> 
> - 🧩 **Plugins extend Metasploit's functionality.**
>     
> - 📁 Default plugin location:
>     
> 
> ```text
> /usr/share/metasploit-framework/plugins
> ```
> 
> - 🚀 Load a plugin:
>     
> 
> ```text
> load <plugin>
> ```
> 
> - ❌ If a plugin cannot be found, Metasploit will report a loading error.
>     
> - ➕ Plugins can add completely new commands to `msfconsole`.
>     
> - 🔗 Plugins can integrate external security tools such as Nessus and Nmap.
>     
> - 🐍 Metasploit plugins are commonly written in **Ruby**.
>     
> - 🧬 **Mixins** are Ruby modules used to share reusable functionality between classes.
>     
> 
> ### ⭐ Most Important Distinction
> 
> ```text
> Modules → perform tasks
> Plugins → extend Metasploit
> ```