# 🔐 1. Linux Authentication Process

> [!info]  
> Linux supports several authentication mechanisms. One of the most common is **PAM — Pluggable Authentication Modules**.
> 
> PAM provides a standardized framework that applications use for:
> 
> - 🔑 Authentication
>     
> - 👤 User/account management
>     
> - 🖥️ Session management
>     
> - 🔄 Password changes
>     
> 
> During credential attacks, understanding how PAM, `/etc/passwd`, and `/etc/shadow` interact is essential.

---

# 🧠 2. PAM — Pluggable Authentication Modules

**PAM** is a framework that allows Linux applications to use standardized authentication modules without implementing authentication logic themselves.

Common PAM modules include:

```text
pam_unix.so
pam_unix2.so
```

On Debian-based systems, PAM security modules are typically located under:

```text
/usr/lib/x86_64-linux-gnu/security/
```

### 🔄 Example: Changing a Password

When a user runs:

```bash
passwd
```

the process involves PAM:

```text
User
 │
 │ passwd
 ↓
PAM
 │
 ├── Authentication
 ├── Account management
 ├── Password management
 └── Session management
       │
       ↓
 /etc/passwd
 /etc/shadow
```

The `pam_unix.so` module uses standardized system-library APIs to manage account information and interacts primarily with:

```text
/etc/passwd
/etc/shadow
```

PAM also supports modules for other authentication mechanisms, including:

- LDAP
    
- Kerberos
    
- Mount operations
    
- Other authentication providers
    

> [!important]  
> **Mental model:** PAM is the **authentication framework**, while files such as `/etc/passwd` and `/etc/shadow` contain important account/password information that PAM can manage.

---

# 👤 3. `/etc/passwd`

The `/etc/passwd` file contains information about **every user account on the Linux system**.

It is normally:

- 📖 **World-readable**
    
- Used by applications and services to obtain user/account information
    
- Structured into **7 colon-separated fields**
    

Example:

```text
htb-student:x:1000:1000:,,,:/home/htb-student:/bin/bash
```

The fields are:

|FieldValue||
|---|---|
|Username|`htb-student`|
|Password|`x`|
|User ID|`1000`|
|Group ID|`1000`|
|[GECOS](https://en.wikipedia.org/wiki/Gecos_field)|`,,,`|
|Home directory|`/home/htb-student`|
|Default shell|`/bin/bash`|

---

# 🧩 4. Understanding the `/etc/passwd` Fields

A useful representation is:

```text
htb-student : x : 1000 : 1000 : ,,, : /home/htb-student : /bin/bash
     │        │      │      │       │          │                 │
     │        │      │      │       │          │                 └─ 🐚 Shell
     │        │      │      │       │          └─ 🏠 Home directory
     │        │      │      │       └─ 📝 GECOS
     │        │      │      └─ 👥 Group ID (GID)
     │        │      └─ 👤 User ID (UID)
     │        └─ 🔑 Password field
     └─ 👤 Username
```

### 🔑 The Password Field

The second field is especially important during credential hunting.

On modern Linux systems, we will usually see:

```text
x
```

For example:

```text
htb-student:x:1000:1000:,,,:/home/htb-student:/bin/bash
```

The `x` generally means:

> The password hash is stored in `/etc/shadow` rather than directly in `/etc/passwd`.

---

# ⚠️ 5. Why `/etc/passwd` Still Matters

Although modern systems normally don't store password hashes in `/etc/passwd`, we should **always inspect it**.

On very old systems, the password field may contain the actual password hash.

Because `/etc/passwd` is normally world-readable:

```text
/etc/passwd
     │
     └── readable by normal users
              │
              ↓
       If hashes are stored here
              │
              ↓
         🔓 Can be copied
              │
              ↓
           Cracked
```

> [!warning]  
> Never assume `/etc/passwd` is harmless just because modern Linux distributions normally use `/etc/shadow`.

---

# 🚨 6. Dangerous `/etc/passwd` Misconfiguration

A particularly serious misconfiguration occurs if `/etc/passwd` is **writeable by an unauthorized user**.

For example:

```text
root::0:0:root:/root:/bin/bash
```

Notice the empty password field:

```text
root::0:0...
     ^^
```

The password field is empty.

This can result in **no password prompt** when attempting to authenticate as `root`.

Example:

```bash
keelwa@htb[/htb]$ head -n 1 /etc/passwd

root::0:0:root:/root:/bin/bash
```

Then:

```bash
keelwa@htb[/htb]$ su

root@htb[/htb]#
```

### 🧠 Attack Chain

```text
/etc/passwd incorrectly writable
            ↓
Modify root's password field
            ↓
Password field becomes empty
            ↓
Attempt authentication
            ↓
Potential root access
```

> [!warning]  
> This is a **rare but critical misconfiguration**. Always check permissions on sensitive authentication files when performing Linux enumeration.

Poorly configured applications may require unusual directory permissions. An inexperienced administrator might accidentally make `/etc` or authentication-related files writable and fail to correct the issue.

---

# 🔒 7. `/etc/shadow`

The `/etc/shadow` file was introduced to prevent ordinary users from accessing password hashes.

Unlike `/etc/passwd`, `/etc/shadow` is intended to be readable only by **administrative/root-level users**.

It contains password-related information for users.

Example:

```text
htb-student:$y$j9T$3QSBB6CbHEu...SNIP...f8Ms:18955:0:99999:7:::
```

It contains **9 colon-separated fields**.

|FieldValue||
|---|---|
|Username|`htb-student`|
|Password|`$y$j9T$3QSBB6CbHEu...SNIP...f8Ms`|
|Last change|`18955`|
|Min age|`0`|
|Max age|`99999`|
|Warning period|`7`|
|Inactivity period|`-`|
|Expiration date|`-`|
|Reserved field|`-`|

---

# 🧩 8. Understanding `/etc/shadow`

The structure can be visualized as:

```text
username : password : last-change : min : max : warning : inactive : expire : reserved
    │           │          │          │     │       │          │         │        │
    │           │          │          │     │       │          │         │        └─ Reserved
    │           │          │          │     │       │          │         └─ Account expiration
    │           │          │          │     │       │          └─ Inactivity period
    │           │          │          │     │       └─ Password warning period
    │           │          │          │     └─ Maximum password age
    │           │          │          └─ Minimum password age
    │           │          └─ Last password change
    │           └─ Password hash
    └─ Username
```

> [!important]  
> For password attacks, the **Password field** is the most interesting field because it contains the password hash and its associated hashing scheme.

---

# 🚫 9. Special Password Field Values

The password field can contain special values.

### `!` or `*`

If the password field contains:

```text
!
```

or:

```text
*
```

the user **cannot authenticate using a Unix password**.

However, other authentication mechanisms may still work, such as:

- 🔑 SSH key-based authentication
    
- 🏢 Kerberos
    
- Other configured authentication mechanisms
    

### Empty Password Field

If the field is empty:

```text
username::...
```

then **no Unix password is required** for password authentication.

This can create security problems and may also cause some applications to deny access to certain functions.

> [!warning]  
> A locked Unix password does **not necessarily mean the account is completely unusable**. Always consider alternative authentication mechanisms.

---

# 🧬 10. Linux Password Hash Format

Linux password hashes commonly follow this structure:

```text
$<id>$<salt>$<hashed>
```

For example:

```text
$y$j9T$3QSBB6CbHEu...SNIP...f8Ms
```

Breakdown:

```text
$y$ j9T $ 3QSBB6CbHEu...f8Ms
 │   │            │
 │   │            └─ 🔐 Salt + password hash data
 │   └─ Algorithm-specific parameters
 └─ 🧬 Hash algorithm identifier
```

The **ID** identifies the cryptographic password hashing algorithm.

---

# 🔢 11. Linux Password Hash Identifiers

|IDCryptographic Hash Algorithm||
|---|---|
|`1`|[MD5](https://en.wikipedia.org/wiki/MD5)|
|`2a`|[Blowfish](https://en.wikipedia.org/wiki/Blowfish_\(cipher\))|
|`5`|[SHA-256](https://en.wikipedia.org/wiki/SHA-2)|
|`6`|[SHA-512](https://en.wikipedia.org/wiki/SHA-2)|
|`sha1`|[SHA1crypt](https://en.wikipedia.org/wiki/SHA-1)|
|`y`|[Yescrypt](https://github.com/openwall/yescrypt)|
|`gy`|[Gost-yescrypt](https://www.openwall.com/lists/yescrypt/2019/06/30/1)|
|`7`|[Scrypt](https://en.wikipedia.org/wiki/Scrypt)|

### 🧠 Quick Recognition

```text
$1$     → MD5
$2a$    → Blowfish
$5$     → SHA-256
$6$     → SHA-512
$y$     → Yescrypt
$gy$    → Gost-yescrypt
$7$     → Scrypt
```

> [!important]  
> Many modern Linux distributions, including Debian, use **yescrypt** as the default password hashing algorithm. Older systems may still contain MD5, SHA-256, SHA-512, or other legacy hashes.

---

# 🔐 12. `/etc/passwd` vs `/etc/shadow`

|File|Purpose|Typical Permissions|Password Hashes?|
|---|---|---|---|
|`/etc/passwd`|User/account information|World-readable|Usually no|
|`/etc/shadow`|Password information|Administrative access|Yes|

### 🧠 Easy Mental Model

```text
/etc/passwd
     │
     ├── Who are the users?
     ├── UID/GID
     ├── Home directory
     ├── Shell
     └── Password pointer → x
                         │
                         ↓
                   /etc/shadow
                         │
                         └── Password hash
```

---

# 🔄 13. `opasswd` — Previous Passwords

PAM can prevent users from reusing old passwords.

Previous password hashes can be stored in:

```text
/etc/security/opasswd
```

Reading this file normally requires **root/administrative privileges**, assuming permissions have not been manually changed.

Example:

```bash
keelwa@htb[/htb]$ sudo cat /etc/security/opasswd
```

Output:

```text
cry0l1t3:1000:2:$1$HjFAfYTG$qNDkF0zJ3v8ylCOrKB0kt0,$1$kcUjWZJX$E9uMSmiQeRh4pAAgzuvkq1
```

---

# 🧠 14. Why `opasswd` Is Interesting

The file can contain **multiple previous password hashes** for a user.

Example:

```text
cry0l1t3
   │
   ├── Previous password #1
   │
   └── Previous password #2
```

This is valuable because users frequently create passwords by **modifying older passwords** rather than inventing completely new ones.

For example:

```text
Old password:
Summer2024!

New password:
Summer2025!
```

or:

```text
Company2024!
Company2025!
```

> [!important]  
> Historical password hashes can reveal **password patterns and mutations**, potentially making the user's current password easier to guess.

---

# ⚡ 15. Hash Algorithm Matters

In the example above, the hashes begin with:

```text
$1$
```

which indicates **MD5-based crypt**.

MD5-based password hashes are significantly easier to crack than stronger schemes such as SHA-512 crypt.

Therefore:

```text
Older / weaker hash
       ↓
Faster cracking
       ↓
Potential old password
       ↓
Identify password pattern
       ↓
Improve current-password guesses
```

> [!tip]  
> Don't only ask **"Did I crack this password?"**
> 
> Also ask:
> 
> **"What does this old password tell me about how the user creates passwords?"**

---

# 🔓 16. Cracking Linux Credentials

Once we obtain **root access**, we can acquire:

```text
/etc/passwd
/etc/shadow
```

and attempt to recover plaintext passwords from the hashes.

A common tool for preparing the hashes for cracking is:

**John the Ripper's `unshadow` utility.**

[John the Ripper unshadow source](https://github.com/pmittaldev/john-the-ripper/blob/master/src/unshadow.c?utm_source=chatgpt.com)

---

# 🧬 17. What `unshadow` Does

`unshadow` combines:

```text
/etc/passwd
      +
/etc/shadow
      ↓
unshadow
      ↓
Combined cracking file
```

Why?

Because `/etc/passwd` contains account information while `/etc/shadow` contains the corresponding password hashes.

---

# 📥 18. Back Up the Authentication Files

The example first creates copies:

```bash
sudo cp /etc/passwd /tmp/passwd.bak
sudo cp /etc/shadow /tmp/shadow.bak
```

Then combines them:

```bash
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
```

### 🧠 Workflow

```text
/etc/passwd ──────┐
                  │
                  ├──→ unshadow ──→ /tmp/unshadowed.hashes
                  │
/etc/shadow ──────┘
```

> [!important]  
> The resulting `unshadowed.hashes` file is now suitable for password-cracking tools such as **John the Ripper** or **Hashcat**.

---

# 💥 19. Crack with Hashcat

Example:

```bash
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked
```

### 🧩 Important Options

```text
-m 1800
   ↓
Hashcat mode for SHA-512 crypt

-a 0
   ↓
Dictionary attack

/tmp/unshadowed.hashes
   ↓
Input hashes

rockyou.txt
   ↓
Wordlist

-o /tmp/unshadowed.cracked
   ↓
Output file for cracked results
```

### 🔄 Complete Process

```text
       🔑 Root Access
            │
            ↓
      Read passwd/shadow
            │
            ↓
       Create backups
            │
            ↓
         unshadow
            │
            ↓
   Combined hash file
            │
       ┌────┴────┐
       ↓         ↓
    Hashcat     JtR
       │         │
       └────┬────┘
            ↓
    🔓 Recovered password
```

---

# 🧨 20. John the Ripper — Single Crack Mode

> [!info]  
> The `unshadow` + password-cracking workflow is particularly relevant to **John the Ripper's `single crack mode`**.

Single crack mode can use information associated with usernames and other account data to generate password guesses.

This makes it particularly useful when cracking Linux password hashes where account information is available.

---

# 📊 21. Linux Authentication Cheat Sheet

|Component|Purpose|Attacker Interest|
|---|---|---|
|PAM|Authentication framework|Understand authentication flow|
|`pam_unix.so`|Unix account/password authentication|Understand local password handling|
|`/etc/passwd`|User/account information|Enumerate users, inspect password field|
|`/etc/shadow`|Password hashes and aging information|High-value credential target|
|`/etc/security/opasswd`|Previous password hashes|Discover old passwords/patterns|
|`unshadow`|Combines passwd + shadow|Prepare hashes for cracking|
|Hashcat|Password cracking|Recover plaintext passwords|
|John the Ripper|Password cracking|Crack Linux hashes / single mode|

---

# 🎯 22. CPTS Credential-Hunting Workflow

```text
🖥️ Linux Access
      │
      ↓
👤 Enumerate Users
      │
      ├── cat /etc/passwd
      │
      └── Identify interesting accounts
      │
      ↓
🔐 Obtain privileged access
      │
      ↓
📂 Read /etc/shadow
      │
      ├── Identify hash algorithm
      │
      ├── Extract password hashes
      │
      └── Check password aging information
      │
      ↓
📜 Check /etc/security/opasswd
      │
      └── Look for historical password hashes
      │
      ↓
🧬 Identify Hash Type
      │
      ↓
🔧 unshadow
      │
      ↓
💥 Hashcat / JtR
      │
      ↓
🔓 Recovered Password
      │
      ↓
♻️ Test for Password Reuse
```

---

# 🧠 23. High-Value CPTS Mental Model

### Windows Credential Hunting

```text
Windows
 ├── SAM
 ├── LSASS
 ├── Credential Manager
 ├── Browsers
 ├── Config files
 └── Scripts
```

### Linux Credential Hunting

```text
Linux
 ├── /etc/passwd
 ├── /etc/shadow
 ├── /etc/security/opasswd
 ├── Config files
 ├── SSH keys
 ├── Shell history
 └── Application credentials
```

For this section, remember the **three most important authentication files**:

```text
/etc/passwd
     ↓
User/account information

/etc/shadow
     ↓
Current password hashes

/etc/security/opasswd
     ↓
Historical password hashes
```

---

# ⚡ 24. Quick Recognition Cheat Sheet

```text
🔑 PAM
→ Linux authentication framework

👤 /etc/passwd
→ User information
→ Usually world-readable
→ `x` normally points password storage to /etc/shadow

🔐 /etc/shadow
→ Password hashes
→ Normally privileged access only

📜 /etc/security/opasswd
→ Previous password hashes

🧬 $<id>$<salt>$<hash>
→ Linux password hash structure

$1$
→ MD5

$5$
→ SHA-256

$6$
→ SHA-512

$y$
→ Yescrypt

🔧 unshadow
→ Combines passwd + shadow

💥 Hashcat
→ Crack resulting hashes

👑 Root
→ Can normally access protected authentication files
```

> [!success]  
> **CPTS Key Takeaway:**
> 
> Linux authentication revolves around **PAM + `/etc/passwd` + `/etc/shadow`**.
> 
> During credential attacks, identify users → obtain protected hashes → identify the hashing algorithm → prepare the hashes with `unshadow` → crack them with Hashcat/JtR → analyze recovered and historical passwords for **reuse and mutation patterns**.