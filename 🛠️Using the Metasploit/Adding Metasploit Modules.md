# 🧩 Writing & Importing Metasploit Modules

> [!info]+ 🎯 What Is This Section About?  
> Metasploit is not limited to the modules that come installed by default.
> 
> During a penetration test, we may encounter:
> 
> ```text
> 🆕 New exploit
> 🧩 Custom exploit
> 📜 Public PoC
> 🐍 Python exploit
> 🐘 PHP exploit
> 💎 Ruby exploit
> ```
> 
> We may need to:
> 
> ```text
> 1️⃣ Update Metasploit
> 2️⃣ Find a Metasploit-compatible module
> 3️⃣ Import a module manually
> 4️⃣ Reload Metasploit
> 5️⃣ Verify the module
> 6️⃣ Port another exploit into a Metasploit module
> 7️⃣ Write our own module when necessary
> ```
> 
> 🧠 **Core idea:**
> 
> > Metasploit is extensible. If the module you need isn't installed, you can often obtain or develop it yourself.

---

# 🔄 1️⃣ Keeping Metasploit Up to Date

Metasploit's module collection is continually updated.

If a module has already been merged into the main Metasploit Framework repository, updating Metasploit can make it available locally.

Conceptually:

```text
🌐 Metasploit Framework Updates
            │
            ▼
     📦 New Modules
            │
            ▼
       msfconsole
```

This is normally the easiest option when you want the **latest framework version**.

> [!tip]+ 🧠 Remember  
> If the module already exists in the current Metasploit release:
> 
> ```text
> 🔄 Update Metasploit
>       ↓
> 🧩 Module becomes available
> ```
> 
> But sometimes we only need **one specific module**.

---

# 📥 2️⃣ Manually Importing a Specific Module

If you don't want to perform a full update, you can obtain a specific Metasploit-compatible module and place it in the appropriate local module directory.

A common source for public exploits is:

[Exploit Database](https://www.exploit-db.com/?utm_source=chatgpt.com)

ExploitDB contains many public proof-of-concept exploits, including exploits already written in **Metasploit module format**.

```text
ExploitDB
    │
    ▼
🔎 Find exploit
    │
    ▼
🏷️ Metasploit tag
    │
    ▼
📜 Ruby module
    │
    ▼
📥 Download
    │
    ▼
📂 Place in Metasploit modules directory
```

---

# 🏷️ 3️⃣ Finding Metasploit-Compatible Exploits

ExploitDB provides a **Metasploit Framework** tag.

When searching ExploitDB, this allows us to narrow the results to exploits that have a Metasploit-compatible version.

```text
ExploitDB
   │
   ▼
Search target/software
   │
   ▼
Filter:
Metasploit Framework
   │
   ▼
📜 .rb module
```

> [!important]+ 🧠 Important  
> Finding an exploit on ExploitDB does **not automatically mean it is a Metasploit module**.
> 
> You need to determine whether the exploit is:
> 
> ```text
> ✅ Metasploit-compatible module
> ```
> 
> or:
> 
> ```text
> ❌ Standalone exploit written in another format
> ```

---

# 💻 4️⃣ `searchsploit`

Instead of using the ExploitDB website, we can search the local ExploitDB database from the terminal using:

```bash
searchsploit <keyword>
```

Example:

```bash
searchsploit nagios3
```

This may return:

```text
Nagios3 - 'history.cgi' Host Command Execution (Metasploit)
Nagios3 - 'history.cgi' Remote Command Execution
Nagios3 - 'statuswml.cgi' Command Execution (Metasploit)
Nagios3 - 'statuswml.cgi' Command Injection (Metasploit)
```

This is extremely useful because we can quickly determine whether a public exploit already exists.

---

# 📜 5️⃣ Understanding `.rb` Files

Metasploit modules are primarily written in:

```text
💎 Ruby
```

Therefore, many Metasploit modules use:

```text
.rb
```

Example:

```text
9861.rb
```

However:

> [!warning]+ ⚠️ `.rb` Does NOT Automatically Mean "Metasploit Module"
> 
> ```text
> .rb
>  │
>  ├── Could be a Metasploit module
>  │
>  └── Could simply be a standalone Ruby exploit
> ```
> 
> Always inspect the code or exploit description.

A particularly useful clue is an ExploitDB entry explicitly marked:

```text
(Metasploit)
```

---

# 🎯 6️⃣ Filtering `searchsploit`

We can filter results to make searches more useful.

For example:

```bash
searchsploit -t Nagios3 --exclude=".py"
```

This can remove Python results and leave the Ruby-based results.

Conceptually:

```text
searchsploit nagios3
       │
       ▼
Many results
       │
       ▼
Exclude .py
       │
       ▼
More relevant Ruby results
```

> [!tip]+ 🧠 Search Strategy  
> When looking for a Metasploit module:
> 
> ```text
> 🔎 Search software
>       ↓
> 🏷️ Look for "(Metasploit)"
>       ↓
> 📜 Check the file extension/code
>       ↓
> 📥 Obtain module
> ```

---

# 📂 7️⃣ Metasploit's Module Directory

The main Metasploit installation is commonly located at:

```text
/usr/share/metasploit-framework/
```

Inside it are directories such as:

```text
modules/
plugins/
scripts/
data/
tools/
msfconsole
msfvenom
...
```

The most important directory for this topic is:

```text
/usr/share/metasploit-framework/modules/
```

Inside `modules/`, Metasploit organizes modules by type and target.

For example:

```text
modules/
│
├── exploits/
├── auxiliary/
├── post/
├── payloads/
├── encoders/
├── nops/
└── ...
```

---

# 🏠 8️⃣ The Local `.msf4` Directory

Metasploit also uses a user-specific directory:

```text
~/.msf4/
```

It may contain:

```text
history
local
logos
logs
loot
modules
plugins
store
```

A particularly important directory is:

```text
~/.msf4/modules/
```

This provides a location for **custom/local Metasploit modules**.

Conceptually:

```text
System modules
     │
     ▼
/usr/share/metasploit-framework/modules/

User/custom modules
     │
     ▼
~/.msf4/modules/
```

> [!important]+ 🧠 Practical Lesson  
> The exact locations can vary depending on the installation and environment.
> 
> What matters is understanding that Metasploit has:
> 
> ```text
> 📦 Framework modules
> +
> 🧩 Local/custom modules
> ```

---

# 🗂️ 9️⃣ Module Directory Structure

Metasploit expects the module's filesystem location to correspond to its module path.

For example:

```text
modules/exploits/unix/webapp/
```

could contain:

```text
nagios3_command_injection.rb
```

which becomes:

```text
exploit/unix/webapp/nagios3_command_injection
```

The relationship is:

```text
Filesystem:

modules/
└── exploits/
    └── unix/
        └── webapp/
            └── nagios3_command_injection.rb


Metasploit path:

exploit/unix/webapp/nagios3_command_injection
```

> [!important]+ 🧠 Remember  
> **The directory structure determines the module path and category.**

---

# ✍️ 🔟 Module Naming Conventions

Metasploit modules should follow appropriate naming conventions.

Use:

```text
snake_case
```

Example:

```text
nagios3_command_injection.rb
```

Good:

```text
our_module_here.rb
nagios3_command_injection.rb
bludit_auth_bypass.rb
```

Avoid names containing:

```text
❌ Spaces
❌ Random special characters
❌ Dashes where underscores are expected
❌ Inconsistent naming
```

> [!tip]+ 🧠 Easy Rule
> 
> ```text
> lowercase
> +
> underscores
> +
> alphanumeric characters
> ```
> 
> Example:
> 
> ```text
> good_module_name.rb
> ```

---

# 📥 1️⃣1️⃣ Copying a Module Into Metasploit

Once we obtain a compatible module, we place it into the correct module directory.

Conceptually:

```bash
cp <module.rb> <metasploit-module-directory>/
```

For example:

```text
Downloaded module
       │
       ▼
9861.rb
       │
       ▼
Rename appropriately
       │
       ▼
nagios3_command_injection.rb
       │
       ▼
modules/exploits/unix/webapp/
```

Now Metasploit has the module in the expected location.

---

# 🔄 1️⃣2️⃣ Loading Custom Modules

There are several ways to make Metasploit recognize a newly added module.

### Method 1 — `loadpath`

Inside `msfconsole`:

```bash
loadpath /usr/share/metasploit-framework/modules/
```

This tells Metasploit where to look for modules.

---

### Method 2 — `reload_all`

If Metasploit is already running:

```bash
reload_all
```

This reloads the module tree.

Conceptually:

```text
📂 New module added
       │
       ▼
reload_all
       │
       ▼
🔄 Metasploit rescans modules
       │
       ▼
🧩 New module available
```

---

### Method 3 — Start Metasploit With a Module Path

Metasploit can also be launched with a module path:

```bash
msfconsole -m /usr/share/metasploit-framework/modules/
```

---

# 🔎 1️⃣3️⃣ Verifying the Imported Module

After loading the module, search for it:

```bash
search <module-name>
```

or directly use the module path:

```bash
use exploit/unix/webapp/nagios3_command_injection
```

Then:

```bash
show options
```

If the module loads successfully:

```text
msf6 exploit(unix/webapp/nagios3_command_injection) >
```

you know Metasploit has recognized it.

> [!success]+ 🎯 Verification Workflow
> 
> ```text
> 📥 Download
>      ↓
> 📂 Place correctly
>      ↓
> 🔄 reload_all
>      ↓
> 🔎 search
>      ↓
> 🎯 use
>      ↓
> 📋 show options
> ```
> 
> If it appears correctly, the module has been imported successfully.

---

# 🧩 1️⃣4️⃣ Importing ≠ Porting

These two concepts are easy to confuse.

### 📥 Importing

The exploit **already is a Metasploit module**.

```text
Existing MSF module
       │
       ▼
Download
       │
       ▼
Place in modules/
       │
       ▼
reload_all
       │
       ▼
Use
```

---

### 🔨 Porting

The exploit exists, but it is **not yet a proper Metasploit module**.

For example:

```text
Python exploit
       │
       ▼
Understand exploit
       │
       ▼
Rewrite/adapt
       │
       ▼
Ruby Metasploit module
       │
       ▼
Load into Metasploit
```

> [!important]+ 🧠 Golden Distinction
> 
> ```text
> IMPORT
> = Already a Metasploit module
> 
> PORT
> = Convert/adapt an existing exploit into a Metasploit module
> ```

---

# 💎 1️⃣5️⃣ Porting Exploits Into Metasploit

Sometimes you find a useful exploit written in:

```text
🐍 Python
🐘 PHP
💎 Ruby
📝 Other languages
```

but there is no Metasploit module for it.

We can potentially **port** the exploit into Metasploit.

This requires understanding:

```text
💎 Ruby
+
🧩 Metasploit Framework API
+
🎯 Original exploit logic
```

The basic process is:

```text
Existing PoC
     │
     ▼
Understand how it works
     │
     ▼
Find suitable MSF boilerplate
     │
     ▼
Adapt the exploit logic
     │
     ▼
Define module metadata
     │
     ▼
Define options
     │
     ▼
Implement exploit logic
     │
     ▼
Load into Metasploit
     │
     ▼
Test
```

---

# 🧱 1️⃣6️⃣ Use an Existing Module as Boilerplate

You don't normally need to start from an empty Ruby file.

Instead:

```text
Find a similar existing module
          │
          ▼
Copy it
          │
          ▼
Modify it
          │
          ▼
Adapt it to your exploit
```

For example, if you're porting an HTTP-based exploit, an existing HTTP exploit module can provide useful structure.

```text
Existing HTTP Module
        │
        ├── HTTP client
        ├── Options
        ├── Request handling
        └── Exploit structure
                │
                ▼
        🛠️ Modify for new exploit
```

> [!tip]+ 🧠 Why This Is Better  
> Existing modules demonstrate:
> 
> ```text
> 📐 Correct structure
> 🧩 Appropriate mixins
> ⚙️ Option registration
> 🌐 HTTP handling
> 📦 Payload handling
> 📝 Metadata
> ```
> 
> You learn the framework by studying modules that already work.

---

# 🧩 1️⃣7️⃣ Metasploit Mixins

Metasploit modules commonly use `include` statements to add functionality.

Example:

```ruby
include Msf::Exploit::Remote::HttpClient
include Msf::Exploit::PhpEXE
include Msf::Auxiliary::Report
```

These are **mixins**.

Think of a mixin as providing reusable functionality to your module.

```text
Your Module
     │
     ├── HttpClient
     │      └── HTTP functionality
     │
     ├── PhpEXE
     │      └── PHP payload functionality
     │
     └── Report
            └── Reporting functionality
```

---

# 🌐 1️⃣8️⃣ Common Mixins From the Example

### `Msf::Exploit::Remote::HttpClient`

Provides functionality for interacting with HTTP servers.

Useful when:

```text
🎯 Target
   │
   ▼
🌐 HTTP/HTTPS
```

---

### `Msf::Exploit::PhpEXE`

Provides functionality related to generating/handling a PHP first-stage payload.

Useful when the exploit involves PHP execution.

---

### `Msf::Exploit::FileDropper`

Provides functionality related to transferring files and cleaning them up after exploitation.

If your exploit doesn't need this functionality:

```text
❌ Don't include it
```

---

### `Msf::Auxiliary::Report`

Provides methods for reporting information to the Metasploit database.

---

> [!important]+ 🧠 Mixin Mental Model
> 
> ```text
> Need HTTP?
>    → HttpClient
> 
> Need PHP payload functionality?
>    → PhpEXE
> 
> Need file cleanup?
>    → FileDropper
> 
> Need reporting?
>    → Report
> ```
> 
> Don't blindly copy every mixin from the boilerplate. **Use only what the module requires.**

---

# 📚 1️⃣9️⃣ Metasploit Documentation

When writing or porting a module, you may need to determine:

```text
❓ Which class?
❓ Which mixin?
❓ Which method?
❓ Which option type?
❓ Which API function?
```

The Metasploit API documentation is therefore extremely useful.

[Metasploit API Documentation](https://docs.metasploit.com/api/?utm_source=chatgpt.com)

The important skill isn't memorizing the entire API.

Instead:

```text
Need functionality
       │
       ▼
Search documentation
       │
       ▼
Find class/mixin/method
       │
       ▼
Implement it
```

---

# 📝 2️⃣0️⃣ Module Metadata

A Metasploit module contains metadata describing the exploit.

A simplified example:

```ruby
def initialize(info={})
  super(update_info(info,
    'Name'           => 'Example Module',
    'Description'    => %q{
      Example vulnerability description.
    },
    'Author'         =>
      [
        'researcher'
      ],
    'References'     =>
      [
        ['CVE', 'CVE-XXXX-XXXXX']
      ],
    'Platform'       => 'php',
    'Arch'           => ARCH_PHP,
    'Targets'        =>
      [
        ['Target', {}]
      ],
    'Privileged'     => false,
    'DisclosureDate' => '2020-01-01',
    'DefaultTarget'  => 0
  ))
end
```

This information becomes visible through:

```bash
info
```

---

# 📋 2️⃣1️⃣ Important Module Metadata

|Field|Purpose|
|---|---|
|`Name`|Module name|
|`Description`|Explains the vulnerability/module|
|`Author`|Credits researchers/developers|
|`References`|CVEs, URLs and other references|
|`Platform`|Target platform|
|`Arch`|Target architecture|
|`Targets`|Supported target configurations|
|`Privileged`|Indicates whether privileged access is expected|
|`DisclosureDate`|Vulnerability disclosure date|
|`DefaultTarget`|Default target configuration|
|`Notes`|Side effects, reliability and stability information|

---

# 🎯 2️⃣2️⃣ Module Options

Modules need configurable values.

These are registered using option definitions.

Example:

```ruby
register_options(
  [
    OptString.new(
      'TARGETURI',
      [true, 'The base path for the application', '/']
    )
  ]
)
```

A module can have options such as:

```text
TARGETURI
USERNAME
PASSWORD
PASSWORDS
RHOSTS
RPORT
SSL
VHOST
```

These eventually appear under:

```bash
show options
```

---

# 📂 2️⃣3️⃣ `OptString` vs `OptPath`

Different option types represent different kinds of input.

### `OptString`

Used for string values.

Example:

```ruby
OptString.new('BLUDITUSER', [true, 'Username'])
```

Conceptually:

```text
BLUDITUSER
    ↓
"user"
```

---

### `OptPath`

Used when the option expects a filesystem path.

Example:

```ruby
OptPath.new(
  'PASSWORDS',
  [true, 'Password wordlist', '/path/to/wordlist']
)
```

Conceptually:

```text
PASSWORDS
    ↓
/path/to/wordlist.txt
```

> [!tip]+ 🧠 Remember
> 
> ```text
> OptString
>     → Text value
> 
> OptPath
>     → Filesystem path
> ```

---

# 🧠 2️⃣4️⃣ Understanding the Porting Process

Suppose we discover:

```text
Bludit 3.9.2
     │
     ▼
Public Python/Ruby PoC
     │
     ▼
No suitable MSF module
```

We can port the exploit.

The process becomes:

```text
1️⃣ Obtain original PoC
        ↓
2️⃣ Understand its vulnerability
        ↓
3️⃣ Identify how the PoC communicates
        ↓
4️⃣ Find similar Metasploit module
        ↓
5️⃣ Copy boilerplate
        ↓
6️⃣ Select required mixins
        ↓
7️⃣ Rewrite module metadata
        ↓
8️⃣ Register required options
        ↓
9️⃣ Adapt exploit logic
        ↓
🔟 Load into Metasploit
        ↓
1️⃣1️⃣ Test
```

---

# 🧪 2️⃣5️⃣ Testing the New Module

After writing or importing the module:

```bash
msfconsole
```

Then:

```bash
reload_all
```

Search:

```bash
search bludit
```

Or directly:

```bash
use exploit/linux/http/<module_name>
```

Then:

```bash
show options
```

You should verify:

```text
✅ Module loads
✅ Correct module type
✅ Correct target platform
✅ Options appear
✅ Required options are identified
✅ `info` displays correctly
```

Only then should you proceed with authorized testing.

---

# 🧠 2️⃣6️⃣ Why Custom Module Development Matters

Most penetration tests won't require you to write an entirely new Metasploit module.

However, understanding module development gives you an important capability:

```text
Public PoC
    │
    ▼
Not directly useful in your workflow
    │
    ▼
🛠️ Adapt it
    │
    ▼
Metasploit module
    │
    ▼
Reusable framework workflow
```

This can be especially useful when dealing with:

```text
🏢 Proprietary applications
🧩 Custom-built software
🆕 Recently disclosed vulnerabilities
🔬 Research PoCs
🌐 Unusual HTTP services
```

---

# ⚠️ 2️⃣7️⃣ Common Problems When Importing Modules

If a new module doesn't appear, check:

```text
❓ Is the file actually a Metasploit module?
❓ Is it in the correct directory?
❓ Is the filename valid?
❓ Is the module path correct?
❓ Did you run reload_all?
❓ Are there Ruby syntax errors?
❓ Are required dependencies available?
```

A useful troubleshooting workflow:

```text
Module missing
     │
     ▼
Check file
     │
     ▼
Check directory
     │
     ▼
Check filename
     │
     ▼
reload_all
     │
     ▼
Search
     │
     ▼
Read errors
     │
     ▼
Fix module
```

---

# 🔬 2️⃣8️⃣ Import vs Port vs Write

This distinction is extremely important.

|Situation|What to do|
|---|---|
|Module already exists in your Metasploit version|🔄 Update/search|
|Module exists publicly but isn't installed|📥 Import|
|Exploit exists but isn't an MSF module|🔨 Port|
|No suitable exploit exists|✍️ Potentially write your own|
|Module doesn't work|🧠 Investigate/customize rather than assuming vulnerability is absent|

Think:

```text
                    PUBLIC EXPLOIT
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
      Already an MSF             Not an MSF
          module                    module
             │                       │
             ▼                       ▼
          📥 IMPORT                🔨 PORT
                                     │
                                     ▼
                              Metasploit Module
```

---

# 📋 2️⃣9️⃣ Quick Reference

### 🔎 Search ExploitDB Locally

```bash
searchsploit <keyword>
```

### 🔎 Search by title

```bash
searchsploit -t <keyword>
```

### 📂 Main Metasploit Directory

```text
/usr/share/metasploit-framework/
```

### 🧩 Module Directory

```text
/usr/share/metasploit-framework/modules/
```

### 🏠 User Module Directory

```text
~/.msf4/modules/
```

### 🔄 Reload Modules

```bash
reload_all
```

### 📂 Load Module Path

```bash
loadpath /path/to/modules/
```

### 🔎 Search Loaded Modules

```bash
search <name>
```

### 🎯 Use Module

```bash
use <module/path>
```

### 📋 Show Options

```bash
show options
```

### 📖 Show Information

```bash
info
```

---

# 🧠 3️⃣0️⃣ CPTS Mental Model

> [!success]+ 🎯 Finding & Using a Missing Metasploit Module
> 
> ```text
> 🔎 Find vulnerability
>          │
>          ▼
> 🧩 Search Metasploit
>          │
>          ├── Module exists?
>          │       │
>          │       └── YES
>          │             ↓
>          │          Use it
>          │
>          └── NO
>                  │
>                  ▼
>             🔎 Search ExploitDB
>                  │
>                  ▼
>             `searchsploit`
>                  │
>                  ▼
>          ┌───────┴────────┐
>          ▼                ▼
>     MSF module        Standalone PoC
>          │                │
>          ▼                ▼
>       📥 Import         🔨 Port
>          │                │
>          └───────┬────────┘
>                  ▼
>             📂 Place module
>                  │
>                  ▼
>              🔄 reload_all
>                  │
>                  ▼
>                🔎 search
>                  │
>                  ▼
>              🎯 use module
>                  │
>                  ▼
>             📋 show options
>                  │
>                  ▼
>                🧪 Test
> ```
> 
> ---

# ⭐ 3️⃣1️⃣ The Golden Rules

> [!important]+ 🧠 Remember These
> 
> **1. `.rb` does not automatically mean Metasploit module.**
> 
> ```text
> Ruby file ≠ automatically MSF module
> ```
> 
> **2. Importing and porting are different.**
> 
> ```text
> Import = already an MSF module
> Port   = adapt an existing exploit into an MSF module
> ```
> 
> **3. Use existing modules as boilerplate.**
> 
> ```text
> Similar working module
>          ↓
>      Copy structure
>          ↓
>       Adapt it
> ```
> 
> **4. Directory structure matters.**
> 
> ```text
> modules/exploits/linux/http/
>              │
>              └── exploit becomes:
>                  exploit/linux/http/<name>
> ```
> 
> **5. Use `info` and `show options` to understand the module.**
> 
> **6. Use the Metasploit API documentation when you don't know which mixin, class, or method to use.**
> 
> **7. A failed exploit is not automatically proof that the vulnerability is absent.**

---

# 🏆 3️⃣2️⃣ Final CPTS Takeaway

> [!success]+ 🧠 The Big Picture
> 
> ```text
>              🔎 VULNERABILITY
>                     │
>                     ▼
>             Does MSF have it?
>                     │
>          ┌──────────┴──────────┐
>          │                     │
>         YES                    NO
>          │                     │
>          ▼                     ▼
>       Use it             Search public PoCs
>                                │
>                       ┌────────┴────────┐
>                       │                 │
>                    MSF-ready        Standalone
>                       │                 │
>                       ▼                 ▼
>                    📥 Import          🔨 Port
>                       │                 │
>                       └────────┬────────┘
>                                ▼
>                         📂 Load Module
>                                │
>                                ▼
>                         🔄 reload_all
>                                │
>                                ▼
>                         🔎 Verify/Search
>                                │
>                                ▼
>                         ⚙️ Configure
>                                │
>                                ▼
>                           🧪 Test
> ```
> 
> **The important skill isn't memorizing Ruby syntax.**
> 
> The important CPTS skill is knowing what to do when the exploit you need **isn't available in your current Metasploit installation**:
> 
> **Search → Identify → Import or Port → Load → Verify → Test.**