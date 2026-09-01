# 🗄️ Metasploit Databases

---

> [!info] 🎯 What Are Metasploit Databases?  
> Metasploit can use a **PostgreSQL database** to store and organize information discovered during an assessment.
> 
> This becomes extremely useful when an engagement contains many:
> 
> - 🌐 Hosts
>     
> - 🔌 Services
>     
> - 🐛 Vulnerabilities
>     
> - 🔑 Credentials
>     
> - 💰 Loot
>     
> - 📝 Notes
>     
> - 📡 Scan results
>     
> 
> Instead of losing track of findings, Metasploit stores them in a structured database that can be searched, filtered, imported, and exported.

---

## 🧠 The Big Picture

Think of the Metasploit database as your **pentesting notebook + searchable inventory**.

```text
                    PostgreSQL
                        │
                        ▼
                ┌───────────────┐
                │  Metasploit   │
                │    Database   │
                └───────┬───────┘
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
    Hosts            Services          Vulns
       │                │                │
       ▼                ▼                ▼
 Credentials          Loot             Notes
       │
       ▼
   Workspaces
```

> [!important] 💡 Why It Matters  
> During a large assessment, manually tracking everything becomes difficult.
> 
> The database lets Metasploit **remember what you discovered** and use that information later.

---

# 1️⃣ PostgreSQL Setup

Metasploit uses **PostgreSQL** as its database backend.

### 🔍 Check PostgreSQL Status

```bash
sudo service postgresql status
```

If it is not running:

```bash
sudo systemctl start postgresql
```

---

## 🏗️ Initialize the Metasploit Database

```bash
sudo msfdb init
```

This creates/configures the Metasploit database.

Typically, this creates:

```text
User:
msf

Databases:
msf
msf_test

Configuration:
 /usr/share/metasploit-framework/config/database.yml
```

### 🧠 Initialization Flow

```text
PostgreSQL
    │
    ▼
msfdb init
    │
    ├── Create MSF database user
    ├── Create MSF databases
    ├── Create database configuration
    └── Create database schema
```

> [!warning] ⚠️ Initialization Errors  
> Older or outdated Metasploit installations may produce errors during initialization.
> 
> Updating Metasploit/packages and attempting initialization again can resolve some version-related issues.

---

# 2️⃣ Check Database Status

Use:

```bash
sudo msfdb status
```

You can also check from inside `msfconsole`:

```text
db_status
```

Expected result:

```text
[*] Connected to msf. Connection type: PostgreSQL.
```

> [!success] ✅ This Is the Important Check  
> Before relying on Metasploit's database functionality, run:
> 
> ```text
> db_status
> ```
> 
> You want to see:
> 
> ```text
> Connected to msf
> ```

---

# 3️⃣ Start Metasploit With the Database

```bash
sudo msfdb run
```

This starts Metasploit and connects it to the configured database.

Alternatively:

```bash
msfconsole
```

Then:

```text
db_status
```

---

## 🔄 Reinitialize the Database

If the database is configured incorrectly or the MSF database credentials need to be recreated:

```bash
msfdb reinit
```

Then:

```bash
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
```

Verify:

```text
db_status
```

---

# 4️⃣ Database Commands

Inside `msfconsole`:

```text
help database
```

Important commands:

|Command|Purpose|
|---|---|
|`db_connect`|Connect to a database|
|`db_disconnect`|Disconnect from database|
|`db_import`|Import scan/results data|
|`db_export`|Export database data|
|`db_nmap`|Run Nmap and automatically store results|
|`db_rebuild_cache`|Rebuild module cache|
|`db_status`|Check database connection|
|`hosts`|Show stored hosts|
|`services`|Show discovered services|
|`vulns`|Show vulnerabilities|
|`creds`|Show stored credentials|
|`loot`|Show stored loot|
|`notes`|Show stored notes|
|`workspace`|Manage/switch workspaces|

---

# 5️⃣ 🗂️ Workspaces

> [!info] What Is a Workspace?  
> A **workspace** is essentially a folder for an assessment's database information.
> 
> It allows you to separate different targets, networks, or engagements.

### 🧠 Mental Model

```text
Metasploit Database
│
├── default
│
├── Target_1
│   ├── Hosts
│   ├── Services
│   ├── Credentials
│   ├── Vulnerabilities
│   └── Loot
│
└── Target_2
    ├── Hosts
    ├── Services
    ├── Credentials
    ├── Vulnerabilities
    └── Loot
```

This prevents findings from different assessments from becoming mixed together.

---

## 📋 List Workspaces

```text
workspace
```

Example:

```text
* default
```

The `*` indicates the **currently active workspace**.

---

## ➕ Create a Workspace

```text
workspace -a Target_1
```

This creates and switches to the workspace.

---

## 🔀 Switch Workspace

```text
workspace Target_1
```

---

## 🗑️ Delete Workspace

```text
workspace -d Target_1
```

---

## ✏️ Rename Workspace

```text
workspace -r
```

For the exact syntax:

```text
workspace -h
```

---

## 📚 Workspace Help

```text
workspace -h
```

Important options:

```text
workspace                 List workspaces
workspace -v              List workspaces verbosely
workspace [name]          Switch workspace
workspace -a [name]       Add workspace
workspace -d [name]       Delete workspace
workspace -D              Delete all workspaces
workspace -r              Rename workspace
workspace -h              Help
```

> [!tip] 🎯 CPTS Mental Model  
> **One assessment → one workspace**
> 
> If you are working on separate targets/networks, separate them into workspaces.

---

# 6️⃣ 📥 Importing Nmap Results

Metasploit can import scan results into its database.

The primary command is:

```text
db_import <file>
```

For Nmap, **XML output is preferred**.

### Example

First generate an Nmap XML scan:

```bash
nmap -sV -oX Target.xml 10.10.10.40
```

Then inside Metasploit:

```text
db_import Target.xml
```

Metasploit parses the XML and stores the findings.

---

## 🔎 Verify Imported Hosts

```text
hosts
```

Example:

```text
Hosts
=====

address
-------
10.10.10.40
```

---

## 🔌 Verify Services

```text
services
```

Example:

```text
host         port   proto   name
----         ----   -----   ----
10.10.10.40  135    tcp     msrpc
10.10.10.40  139    tcp     netbios-ssn
10.10.10.40  445    tcp     microsoft-ds
```

### 🧠 Import Flow

```text
Nmap
 │
 │ XML
 ▼
db_import
 │
 ▼
Metasploit Database
 │
 ├── hosts
 ├── services
 └── other findings
```

> [!important] ⚠️ Remember  
> Prefer:
> 
> ```text
> Nmap XML → db_import
> ```
> 
> because XML contains structured information that Metasploit can parse reliably.

---

# 7️⃣ 🔎 Run Nmap Directly From MSFconsole

Instead of running Nmap separately and importing the results, Metasploit can execute Nmap directly.

Use:

```text
db_nmap
```

Example:

```text
db_nmap -sV -sS 10.10.10.8
```

The results are automatically stored in the current workspace.

Then:

```text
hosts
```

and:

```text
services
```

can be used to view the results.

---

## 🆚 Two Ways to Get Nmap Data Into Metasploit

|Method|Workflow|
|---|---|
|Import|`nmap -oX Target.xml` → `db_import Target.xml`|
|Direct|`db_nmap -sV <IP>`|

### 🧠 Easy Mental Model

```text
                 NMAP
                  │
          ┌───────┴────────┐
          │                │
       XML file         db_nmap
          │                │
          ▼                ▼
     db_import       Automatic storage
          │                │
          └───────┬────────┘
                  ▼
          Metasploit Database
```

---

# 8️⃣ 💾 Database Backup

You can export the current workspace/database data.

First check syntax:

```text
db_export -h
```

Export as XML:

```text
db_export -f xml backup.xml
```

Example:

```text
[*] Starting export of workspace default to backup.xml
[*] Finished export of workspace default to backup.xml
```

> [!tip] 💡 Why Export?  
> Your database contains valuable assessment information.
> 
> Exporting provides a backup that can later be imported into Metasploit.

---

# 9️⃣ 🖥️ Hosts

The `hosts` command displays hosts stored in the database.

```text
hosts
```

Typical information includes:

```text
IP address
Hostname
OS
Architecture
Comments
Services
Vulnerabilities
Credentials
Tags
```

### 🔍 Useful Options

```text
hosts -h
```

Common options:

|Option|Purpose|
|---|---|
|`-a`|Add hosts|
|`-d`|Delete hosts|
|`-u`|Show only hosts that are up|
|`-S`|Search/filter|
|`-c`|Select columns|
|`-o`|Export to CSV|
|`-R`|Set `RHOSTS` from results|
|`-i`|Modify host information|
|`-n`|Change hostname|
|`-m`|Add/change comments|
|`-t`|Add tags|

---

## ⭐ Important: `hosts -R`

```text
hosts -R
```

This can set the Metasploit `RHOSTS` option from the hosts stored in the database.

### 🧠 Why This Is Useful

Instead of manually typing:

```text
set RHOSTS 10.10.10.8
```

you can use database results to populate the target list.

```text
Database
   │
   ▼
hosts -R
   │
   ▼
RHOSTS
   │
   ▼
Exploit / Scanner
```

---

# 🔟 🔌 Services

The `services` command displays services discovered on stored hosts.

```text
services
```

Example:

```text
host         port   proto   name
----         ----   -----   ----
10.10.10.8   80     tcp     http
10.10.10.8   445    tcp     microsoft-ds
```

### 🔍 Useful Options

```text
services -h
```

|Option|Purpose|
|---|---|
|`-a`|Add service|
|`-d`|Delete service|
|`-p`|Filter by port|
|`-s`|Filter by service name|
|`-r`|Protocol|
|`-u`|Show only services that are up|
|`-S`|Search/filter|
|`-o`|Export to CSV|
|`-R`|Set `RHOSTS` from results|
|`-U`|Update service information|

### Example

Show SMB-related services:

```text
services -s smb
```

Show a specific port:

```text
services -p 445
```

---

# 1️⃣1️⃣ 🔑 Credentials

The `creds` command manages credentials discovered during an assessment.

```text
creds
```

Credentials can include:

```text
Username + Password
Username + NTLM hash
SSH keys
Database credentials
Other hashes
```

---

## 🔎 Credential Filtering

```text
creds -h
```

Useful filters:

|Command|Purpose|
|---|---|
|`creds`|List all credentials|
|`creds -u admin`|Find credentials for a user|
|`creds -p 22-25,445`|Credentials associated with specified ports|
|`creds -s ssh,smb`|Credentials associated with SSH/SMB|
|`creds -t NTLM`|Show NTLM credentials|
|`creds -O <IP>`|Filter by credential origin|
|`creds -R`|Set `RHOSTS` from results|

---

## ➕ Add Credentials

Example:

```text
creds add user:admin password:password123
```

Add an NTLM hash:

```text
creds add user:admin ntlm:<HASH>
```

Add an SSH key:

```text
creds add user:sshadmin ssh-key:/path/to/id_rsa
```

> [!warning] 🔐 Security  
> Credentials are sensitive assessment data. Treat database files, exports, hashes, and loot as confidential.

---

# 1️⃣2️⃣ 💰 Loot

The `loot` command stores valuable information obtained from compromised systems.

```text
loot
```

Examples include:

```text
Password hashes
/etc/passwd
/etc/shadow
Configuration files
Tokens
Keys
Other extracted data
```

### 🔍 Help

```text
loot -h
```

### Common Operations

```text
loot -f <file>
```

Add loot from a file.

```text
loot -i <info>
```

Add information about the loot.

```text
loot -t <type>
```

Filter by loot type.

```text
loot -S <search>
```

Search/filter loot.

```text
loot -d <host>
```

Delete matching loot.

---

# 1️⃣3️⃣ 🐛 Vulnerabilities

Metasploit also tracks vulnerabilities in the database.

```text
vulns
```

This allows you to review vulnerabilities associated with discovered hosts/services.

---

# 1️⃣4️⃣ 📝 Notes

Additional information can be stored using:

```text
notes
```

Notes are useful for recording assessment-specific observations that don't fit neatly into hosts, services, credentials, or loot.

---

# 🧠 Database Cheat Sheet

|Goal|Command|
|---|---|
|Check DB connection|`db_status`|
|Start PostgreSQL|`sudo systemctl start postgresql`|
|Initialize DB|`sudo msfdb init`|
|Run MSF with DB|`sudo msfdb run`|
|Reinitialize DB|`msfdb reinit`|
|Database help|`help database`|
|List workspaces|`workspace`|
|Create workspace|`workspace -a NAME`|
|Switch workspace|`workspace NAME`|
|Delete workspace|`workspace -d NAME`|
|Import Nmap XML|`db_import scan.xml`|
|Run Nmap from MSF|`db_nmap <options> <target>`|
|List hosts|`hosts`|
|List services|`services`|
|List vulnerabilities|`vulns`|
|List credentials|`creds`|
|List loot|`loot`|
|List notes|`notes`|
|Export database|`db_export -f xml backup.xml`|

---

# 🧭 Practical Metasploit Database Methodology

When starting an assessment:

```text
1. Start PostgreSQL
       │
       ▼
2. Verify database
   db_status
       │
       ▼
3. Create/select workspace
   workspace -a Target_1
       │
       ▼
4. Perform reconnaissance
   db_nmap -sV <TARGET>
       │
       ▼
5. Review hosts
   hosts
       │
       ▼
6. Review services
   services
       │
       ▼
7. Investigate vulnerabilities
   vulns
       │
       ▼
8. Store discovered credentials
   creds
       │
       ▼
9. Store useful extracted information
   loot
       │
       ▼
10. Backup results
    db_export -f xml backup.xml
```

> [!important] 🧠 The CPTS Mental Model  
> **The database is not the attack itself — it is the memory of your assessment.**
> 
> Think:
> 
> ```text
> Scan → Store → Organize → Search → Attack → Store Findings
> ```
> 
> This becomes increasingly valuable as the number of targets and findings grows.

---

# 🎯 Key Takeaways

> [!success] What You Should Remember
> 
> **1. PostgreSQL** is Metasploit's database backend.
> 
> **2. `db_status`** tells you whether Metasploit is connected to the database.
> 
> **3. Workspaces** separate different assessments/targets.
> 
> **4. `db_import`** imports external scan results, especially Nmap XML.
> 
> **5. `db_nmap`** runs Nmap from inside `msfconsole` and stores the results automatically.
> 
> **6. `hosts`** → discovered machines.
> 
> **7. `services`** → ports/services running on those machines.
> 
> **8. `vulns`** → discovered vulnerabilities.
> 
> **9. `creds`** → discovered credentials/hashes.
> 
> **🔟 `loot`** → valuable information extracted from targets.
> 
> **11. `notes`** → additional assessment information.
> 
> **12. `db_export`** → backup/export your assessment data.

> [!tip] 🔥 One-Line Memory Trick  
> **Hosts → Services → Vulns → Creds → Loot**
> 
> Think of it as:
> 
> ```text
> WHO is there?
>      ↓
> WHAT are they running?
>      ↓
> WHAT is vulnerable?
>      ↓
> WHAT credentials did I obtain?
>      ↓
> WHAT valuable data did I extract?
> ```