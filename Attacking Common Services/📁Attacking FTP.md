# 📁 1. Attacking FTP

> [!info]  
> **FTP (File Transfer Protocol)** is a network protocol used to transfer files between a client and server.
> 
> FTP also provides file/directory management operations such as:
> 
> - Listing files
>     
> - Changing directories
>     
> - Uploading files
>     
> - Downloading files
>     
> - Renaming files/directories
>     
> - Deleting files/directories
>     
> 
> **Default port:** `TCP/21`

---

# 🎯 2. FTP Attack Methodology

The key idea is that **FTP itself is usually not the target**. We attack the **FTP service implementation or its configuration**.

Common attack paths include:

```text
                 🔎 FTP Enumeration
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
   👤 Anonymous     🔑 Credentials   🐛 Vulnerability
      Access           / Brute Force      │
          │             │                 │
          └──────┬──────┴─────────────────┘
                 ▼
          📂 Access Files
                 │
        ┌────────┴────────┐
        ▼                 ▼
   🔍 Sensitive Info   ✏️ Write Access
        │                 │
        ▼                 ▼
   Credentials /      Upload malicious
   Configs / Keys       files/scripts
        │                 │
        └────────┬────────┘
                 ▼
          🚀 Further Attack
```

> [!important]  
> **After gaining FTP access, don't immediately assume the objective is complete.**
> 
> FTP is often valuable because it gives access to **files**, and those files may contain credentials, configuration data, source code, keys, or other information that enables attacks against another service.

---

# 🔎 3. FTP Enumeration

## 🧭 3.1 Nmap

Nmap's default scripts (`-sC`) include `ftp-anon`, which checks whether the FTP server allows **anonymous authentication**.

`-sV` performs version enumeration and can reveal the FTP banner/version.

```bash
sudo nmap -sC -sV -p 21 192.168.2.142
```

### Example Output

```text
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-10 22:04 EDT
Nmap scan report for 192.168.2.142
Host is up (0.00054s latency).

PORT   STATE SERVICE
21/tcp open  ftp
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--   1 1170     924            31 Mar 28  2001 .banner
| d--x--x--x   2 root     root         1024 Jan 14  2002 bin
| d--x--x--x   2 root     root         1024 Aug 10  1999 etc
| drwxr-srwt   2 1170     924          2048 Jul 19 18:48 incoming [NSE: writeable]
| d--x--x--x   2 root     root         1024 Jan 14  2002 lib
| drwxr-sr-x   2 1170     924          1024 Aug  5  2004 pub
|_Only 6 shown. Use --script-args ftp-anon.maxlist=-1 to see all.
```

### 🧠 What to Notice

The output immediately reveals several important things:

|Finding|Why it matters|
|---|---|
|`21/tcp open ftp`|FTP service is available|
|`ftp-anon: Anonymous FTP login allowed`|Authentication may be bypassed|
|`.banner`|Potential information disclosure|
|`incoming [NSE: writeable]`|Anonymous user may be able to upload files|
|`vsFTPd` version/banner|Useful for version-specific research|

> [!tip]  
> A directory marked **`[NSE: writeable]`** is especially interesting. Read access gives you information; **write access may give you a path to place files on the target**.

---

# 👤 4. Anonymous FTP Authentication

FTP servers can be configured to allow an `anonymous` user.

Typical login:

```text
Username: anonymous
Password: <blank>
```

Example:

```bash
ftp 192.168.2.142
```

```text
Connected to 192.168.2.142.
220 (vsFTPd 2.3.4)
Name (192.168.2.142:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0               9 Aug 12 16:51 test.txt
226 Directory send OK.
```

### 🧠 Mental Model

```text
Anonymous Login
      ↓
   📂 Browse
      ↓
   🔍 Identify
      ↓
┌─────┴──────────┐
▼                ▼
📥 Read          📤 Write
▼                ▼
Download         Upload
files            files
▼                ▼
Credentials /    Potential
Secrets          execution chain
```

> [!warning]  
> **Anonymous access is not automatically a critical vulnerability.**
> 
> The real question is:
> 
> **"What can the anonymous account actually read, write, modify, or delete?"**
> 
> A read-only public directory is very different from an anonymous account with write access to a directory used by a web server.

---

# 📂 5. Navigating an FTP Server

Once authenticated, enumerate the available directory structure.

### Directory Operations

```text
ls
cd <directory>
```

- `ls` → list files/directories
    
- `cd` → change the remote working directory
    

### Download Files

```text
get <file>
```

Download a single file.

```text
mget <files>
```

Download multiple files.

### Upload Files

```text
put <file>
```

Upload a single file.

```text
mput <files>
```

Upload multiple files.

### Help

```text
help
```

Displays available FTP client commands.

---

# 📥📤 6. FTP File Operations Cheat Sheet

|Command|Purpose|
|---|---|
|`ls`|List remote files/directories|
|`cd <dir>`|Change remote directory|
|`get <file>`|Download one file|
|`mget <files>`|Download multiple files|
|`put <file>`|Upload one file|
|`mput <files>`|Upload multiple files|
|`help`|Show FTP client help|

> [!important]  
> Once you gain access, **enumerate the entire accessible directory structure** before moving on.
> 
> Look for:
> 
> - Credentials
>     
> - Configuration files
>     
> - Source code
>     
> - Backups
>     
> - SSH keys
>     
> - Database files
>     
> - Website files
>     
> - Scripts
>     
> - Notes/documentation
>     
> - Files containing usernames/passwords
>     

---

# ⚠️ 7. Why Anonymous FTP Can Be Dangerous

The primary problem isn't simply:

```text
Anonymous FTP = bad
```

Instead:

```text
Anonymous FTP
      +
Incorrect permissions
      ↓
Unauthorized file access
      ↓
Sensitive information disclosure
```

Or:

```text
Anonymous FTP
      +
Write permission
      ↓
Upload attacker-controlled file
      ↓
Find a way for another service to process it
      ↓
Potential code execution
```

### Example Attack Chain

The source gives an important example involving a web application:

```text
Anonymous FTP
      ↓
Upload malicious file
      ↓
File is stored somewhere accessible by web server
      ↓
Path traversal / other web vulnerability
      ↓
Locate uploaded file
      ↓
Web server interprets it as PHP
      ↓
Potential RCE
```

> [!warning]  
> **FTP upload alone does not mean RCE.**
> 
> You need another condition that causes the uploaded file to be **processed or executed**. The source specifically gives a web application/path traversal scenario as an example.

---

# 🔐 8. Brute-Forcing FTP Credentials

If anonymous authentication is unavailable, valid credentials may still be obtained through password attacks.

One tool covered in the source is **Medusa**.

### Medusa Syntax

```bash
medusa -u <username> -P <password-list> -h <target> -M ftp
```

Important options:

|Option|Meaning|
|---|---|
|`-u`|Single username|
|`-U`|Username wordlist|
|`-P`|Password wordlist|
|`-h`|Target hostname/IP|
|`-M`|Module/protocol to attack|

### Example

```bash
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp
```

Example successful result:

```text
ACCOUNT FOUND: [ftp] Host: 10.129.203.7 User: fiona Password: family [SUCCESS]
```

### 🧠 What Happened?

```text
Known username: fiona
        +
Password wordlist: rockyou.txt
        ↓
Medusa tries credentials
        ↓
FTP authentication
        ↓
Valid combination found
        ↓
fiona : family
```

> [!important]  
> The source notes that **brute force is often less effective against modern applications**, because many services implement protections against repeated login attempts.
> 
> **Password spraying** can be a more effective approach when appropriate because it attempts a small number of common passwords across many accounts rather than rapidly attacking one account.

---

# 🔄 9. Brute Force vs Password Spraying

|Technique|Basic Approach|Typical Use|
|---|---|---|
|Brute force|Many passwords against one/few accounts|When rate limiting/lockout is weak|
|Password spraying|Few common passwords against many accounts|When account lockout makes brute force risky|

### 🧠 Mental Model

```text
Brute Force
User A → password1
       → password2
       → password3
       → password4
       → ...

Password Spray
password1 → User A
          → User B
          → User C
          → User D
```

---

# 🌐 10. FTP Bounce Attack

## 🧠 Concept

An **FTP Bounce Attack** abuses an FTP server as an intermediary to send traffic toward another system.

Instead of:

```text
Attacker ───────────→ Internal Target
```

the attacker uses:

```text
Attacker
    │
    ▼
FTP Server
    │
    ▼
Internal Target
```

This can allow an attacker to probe systems that are **not directly reachable** from the attacker's position.

---

# 🎯 11. FTP Bounce Scenario

Imagine:

```text
                🌐 External Network
                       │
                    Attacker
                       │
                       ▼
               ┌──────────────┐
               │   FTP_DMZ    │
               │ Internet     │
               │ accessible   │
               └──────┬───────┘
                      │
             FTP Bounce Proxy
                      │
                      ▼
               ┌──────────────┐
               │ Internal_DMZ │
               │ Not directly │
               │ exposed      │
               └──────────────┘
```

The attacker can potentially use `FTP_DMZ` to scan `Internal_DMZ`.

### Why This Matters

The attacker may discover:

```text
Internal_DMZ
    ↓
Open port 80
    ↓
HTTP service exists
    ↓
Use information for further enumeration/attacks
```

> [!important]  
> The attack is not necessarily about obtaining files from FTP.
> 
> The FTP server is being abused as a **network proxy/traffic relay**.

---

# 🛠️ 12. Nmap FTP Bounce Scan

Nmap provides the `-b` option for FTP bounce attacks.

```bash
nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2
```

### Important Arguments

|Option|Meaning|
|---|---|
|`-Pn`|Treat target as online; skip host discovery|
|`-v`|Verbose output|
|`-n`|Disable DNS resolution|
|`-p80`|Scan TCP port 80|
|`-b`|Use FTP bounce scanning|
|`anonymous:password@10.10.110.213`|FTP server used as the bounce proxy|
|`172.17.0.2`|Internal target being probed|

Example output:

```text
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-27 04:55 EDT
Resolved FTP bounce attack proxy to 10.10.110.213 (10.10.110.213).
Attempting connection to ftp://anonymous:password@10.10.110.213:21
Connected:220 (vsFTPd 3.0.3)
Login credentials accepted by FTP server!
Initiating Bounce Scan at 04:55
FTP command misalignment detected ... correcting.
Completed Bounce Scan at 04:55, 0.54s elapsed (1 total ports)
Nmap scan report for 172.17.0.2
Host is up.

PORT   STATE  SERVICE
80/tcp open http
```

### 🧠 Read the Command Like This

```text
-b anonymous:password@10.10.110.213
        │       │
        │       └── FTP bounce server
        └────────── FTP credentials

172.17.0.2
    │
    └── Actual internal target
```

---

# 🛡️ 13. Modern FTP Bounce Protections

Modern FTP servers commonly include protections that prevent FTP bounce attacks by default.

However:

```text
Modern FTP server
       +
Misconfiguration
       ↓
Bounce protection weakened/disabled
       ↓
Potential FTP Bounce vulnerability
```

> [!warning]  
> Don't assume a modern FTP version is automatically safe from FTP Bounce attacks. **Configuration matters.**

---

# 🧭 14. CPTS FTP Enumeration Workflow

```text
                    🔎 FTP FOUND
                        │
                        ▼
              TCP/21 + Version Scan
                        │
                        ▼
              sudo nmap -sC -sV -p 21
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
     👤 Anonymous?              ❌ No Anonymous
            │                       │
           YES                      ▼
            │                 🔑 Credential Attack
            ▼                       │
       ftp anonymous                 │
            │                       │
            ▼                       ▼
       📂 Enumerate              Valid Login?
       directories                   │
            │                       YES
      ┌─────┴─────┐                  │
      ▼           ▼                  ▼
   📥 Read      📤 Write          📂 Enumerate
      │           │                  │
      ▼           ▼                  ▼
 Sensitive     Upload          Sensitive Info
   Files       Files                │
      │           │                  │
      └─────┬─────┴──────────────────┘
            ▼
       🔗 Correlate
       with other services
            │
            ▼
       🚀 Further Attack
```

---

# 🧠 15. What to Look For After FTP Access

Once authenticated, think beyond **"What files can I download?"**

Ask:

### 🔍 Information

- Are there credentials?
    
- Are there usernames?
    
- Are there configuration files?
    
- Is source code exposed?
    
- Are database credentials present?
    
- Are SSH/private keys present?
    
- Are there backups?
    
- Are there internal hostnames/IP addresses?
    

### 📤 Write Access

- Can I upload files?
    
- Where are uploaded files stored?
    
- Is the directory associated with a web server?
    
- Is another application processing these files?
    
- Can another vulnerability expose or execute the uploaded file?
    

### 🌐 Network Position

- Can FTP reach internal systems?
    
- Is FTP potentially usable for bounce scanning?
    
- Are there internal services that aren't directly reachable?
    

---

# ⚡ 16. FTP CPTS Cheat Sheet

|Situation|Action|
|---|---|
|TCP/21 open|Enumerate FTP with `-sC -sV`|
|Anonymous login allowed|Login as `anonymous`|
|Anonymous access|Enumerate all accessible directories/files|
|Readable sensitive files|Download and analyze locally|
|Writable directory|Investigate upload possibilities|
|No anonymous access|Consider credential attacks|
|Known username|Medusa with `-u`|
|Username list|Medusa with `-U`|
|Password list|Medusa with `-P`|
|FTP version/banner found|Research version-specific vulnerabilities|
|FTP server can reach internal network|Consider FTP Bounce|
|FTP bounce suspected|Nmap `-b`|
|FTP Bounce blocked|Check whether configuration/protection prevents it|
|Credentials found in FTP files|Try correlating them with other services|
|Interesting uploaded file|Determine whether another service can process/execute it|

---

# 🏆 17. One-Minute FTP Methodology

```text
1️⃣ FIND
   └─ TCP/21

2️⃣ ENUMERATE
   └─ nmap -sC -sV -p 21 <TARGET>

3️⃣ CHECK ANONYMOUS
   └─ ftp <TARGET>
   └─ username: anonymous

4️⃣ ENUMERATE FILES
   └─ ls
   └─ cd
   └─ get / mget

5️⃣ CHECK WRITE ACCESS
   └─ put / mput

6️⃣ NO ANONYMOUS?
   └─ Credential attack
   └─ Medusa / password spraying

7️⃣ ANALYZE FILES
   └─ Credentials
   └─ Configs
   └─ Source code
   └─ Keys
   └─ Backups

8️⃣ CORRELATE
   └─ Try discovered credentials/information
      against other services

9️⃣ CHECK NETWORK POSITION
   └─ Could FTP reach internal hosts?
   └─ Consider FTP Bounce

🔟 CONTINUE THE ATTACK
   └─ FTP is often the source of information
      that enables the next stage
```

> [!success]
> 
> ### 🧠 Core CPTS Mental Model
> 
> **FTP → Access → Enumerate → Read/Write → Extract Information → Correlate → Pivot**
> 
> The most important lesson is to treat FTP as a **source of attack primitives**, not merely a file-transfer service. Anonymous access, exposed files, writable directories, credentials, and FTP Bounce can each become stepping stones toward the next stage of an assessment.