# 🔐 1. John the Ripper (JtR)

> [!info] 🧠 What Is John the Ripper?  
> **John the Ripper (JtR)** is an open-source penetration-testing tool used to crack password hashes using multiple techniques, including **dictionary attacks, rule-based attacks, and brute-force-style attacks**.
> 
> It was originally developed for UNIX systems and released in **1996**. The **Jumbo** version is recommended because it provides additional formats, features, wordlists, and performance optimizations.

### 🧠 Mental Model

```text
                  🔐 Password Hash
                         │
                         ▼
              ┌─────────────────────┐
              │  John the Ripper    │
              └──────────┬──────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Single Crack    Wordlist       Incremental
       Mode           Mode             Mode
          │              │              │
          ▼              ▼              ▼
    User-derived      Known          Statistical
     candidates     passwords        guessing
```

---

# 🎯 2. JtR Cracking Modes

John provides several different approaches depending on what information is available.

|Mode|Strategy|Best Use|
|---|---|---|
|👤 **Single**|Uses information about the target user|Linux credentials / known user information|
|📖 **Wordlist**|Tests passwords from a wordlist|Common passwords|
|💥 **Incremental**|Generates candidates using a statistical model|More exhaustive cracking|

> [!important] 🧠 Choosing a Mode  
> The goal is to use the **most informed and efficient attack first** rather than immediately attempting the largest possible search space.

---

# 👤 3. Single Crack Mode

**Single crack mode** is a rule-based attack that generates password candidates from information associated with the target account.

It can use information such as:

- 👤 Username
    
- 🏠 Home directory
    
- 📝 GECOS information
    
    - Full name
        
    - Room number
        
    - Phone number
        
    - Other account metadata
        

JtR then applies password-generation rules to those strings.

### Example

Suppose account information contains:

```text
Username:       r0lf
Full name:      Rolf Sebastian
Home directory: /home/r0lf
```

JtR may generate candidates based on those values, such as variations of:

```text
r0lf
Rolf
Sebastian
Smith1
Rolf123
...
```

> [!tip] 🧠 Why This Works  
> Users frequently create passwords from information that is easy to remember.
> 
> **Personal information → transformations → likely password candidates**

---

# 📄 4. Recognizing Linux Account Information

Consider this `/etc/passwd` entry:

```text
r0lf:$6$ues25dIanlctrWxg$nZHVz2z4kCy1760Ee28M1xtHdGoy0C2cYzZ8l2sVa1kIa8K9gAcdBP.GI6ng/qA4oaMrgElZ1Cb9OeXO4Fvy3/:0:0:Rolf Sebastian:/home/r0lf:/bin/bash
```

Important information can be extracted:

```text
Username:       r0lf
Hash:           $6$ues25dIanlctrWxg$...
UID:            0
GID:            0
Full Name:      Rolf Sebastian
Home Directory: /home/r0lf
Shell:          /bin/bash
```

This provides JtR with useful information for **Single crack mode**.

### ⚔️ Attack Flow

```text
/etc/passwd
     │
     ├── Username ─────────┐
     ├── Full Name ───────┤
     └── Home Directory ──┤
                           ▼
                  Candidate Generation
                           │
                           ▼
                    Apply JtR Rules
                           │
                           ▼
                      Hash Candidate
                           │
                           ▼
                     Compare Hash
                           │
                           ▼
                       🎯 Match
```

---

# 💻 5. Running Single Crack Mode

Basic command:
$passwd would be the file that has the hash$

```bash
john --single passwd
```

Example:

```bash
keelwa@htb[/htb]$ john --single passwd
```

JtR may report information such as:

```text
Loaded 1 password hash (sha512crypt, crypt(3) $6$ ...)
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
```

If the password is cracked, JtR indicates the result.

> [!success] 🎯 Important  
> In the example, JtR successfully cracked the password using information extracted from the account entry and its rules.

To reliably display cracked passwords afterward:

```bash
john --show passwd
```

---

# 📖 6. Wordlist Mode

**Wordlist mode** performs a dictionary attack against the target hash.

Instead of generating every possible combination, JtR reads candidate passwords from a supplied wordlist.

### Basic Syntax

```bash
john --wordlist=<wordlist_file> <hash_file>
```

Example:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

> [!important] 📌 Wordlist Requirements  
> Wordlists must be **plain-text files**, with **one candidate per line**.
> 
> Multiple wordlists can be specified by separating them with a comma.

---

# 🛠️ 7. Rules with Wordlists

Wordlists contain likely passwords, but users frequently modify those passwords.

For example:

```text
password
```

could become:

```text
Password
Password1
password1
password!
Password!
```

JtR can apply **rules** to transform words into additional password candidates.

Common transformations include:

- 🔠 Capitalizing letters
    
- 🔢 Appending numbers
    
- 🔣 Adding special characters
    
- 🔄 Other character transformations
    

Syntax:

```bash
john --wordlist=<wordlist_file> --rules <hash_file>
```

Conceptually:

```text
              Wordlist
                  │
                  ▼
             "password"
                  │
             ┌────┴────┐
             ▼         ▼
        Apply Rules   Original
             │
      ┌──────┼──────────┐
      ▼      ▼          ▼
 Password  password1  Password!
      │      │          │
      └──────┴──────────┘
             │
             ▼
          Test Hashes
```

> [!tip] 🧠 Mental Model  
> **Wordlist = likely base words**
> 
> **Rules = likely human modifications**
> 
> Together they can cover a much larger portion of realistic passwords than a raw wordlist alone.

---

# 💥 8. Incremental Mode

**Incremental mode** is JtR's powerful **brute-force-style** cracking mode.

Unlike Wordlist mode, it does **not** depend on a predefined list of passwords.

Instead, JtR dynamically generates candidates using a **statistical model based on Markov chains**.

```text
Character Set
      │
      ▼
Statistical Model
      │
      ▼
Prioritize likely combinations
      │
      ▼
Generate candidates
      │
      ▼
Hash + Compare
```

### Basic Syntax

```bash
john --incremental <hash_file>
```

Example:

```bash
john --incremental hashes.txt
```

---

# 📊 9. Incremental Mode vs Naïve Brute Force

Incremental mode is more intelligent than simply generating combinations randomly or in an arbitrary order.

It uses statistical information to prioritize combinations that are more likely to represent real passwords.

|Approach|Candidate Generation|
|---|---|
|💥 Naïve brute force|Systematically explores combinations|
|🧠 JtR Incremental|Uses statistical probability to prioritize likely combinations|
|📖 Wordlist|Uses predefined known candidates|

> [!important] ⚠️ Trade-off  
> Incremental mode is **more exhaustive** but can also be **extremely time-consuming**, especially for long or complex passwords.

---

# ⚙️ 10. Incremental Configuration

JtR's predefined incremental modes are specified in its configuration file:

```text
/etc/john/john.conf
```

You can inspect the incremental configurations with:

```bash
grep '# Incremental modes' -A 100 /etc/john/john.conf
```

Example configurations include:

```text
[Incremental:Custom]
File = $JOHN/custom.chr
MinLen = 0

[Incremental:UTF8]
File = $JOHN/utf8.chr
MinLen = 0
CharCount = 196

[Incremental:Latin1]
File = $JOHN/latin1.chr
MinLen = 0
CharCount = 203

[Incremental:ASCII]
File = $JOHN/ascii.chr
MinLen = 0
MaxLen = 13
CharCount = 95
```

### 🔍 What These Settings Control

|Setting|Purpose|
|---|---|
|`File`|Character-set/statistical data used by the mode|
|`MinLen`|Minimum candidate length|
|`MaxLen`|Maximum candidate length|
|`CharCount`|Number of characters available|

You can also specify a character set directly:

```bash
john --incremental=whatever.chr
```

> [!tip] 🎯 Optimization  
> Restricting the **character set** and **password length** can dramatically reduce unnecessary work and focus the attack on a more realistic target space.

---

# 🕵️ 11. Identifying Unknown Hash Formats

Before cracking a hash, you need to determine **what type of hash you're dealing with**.

Consider:

```text
193069ceb0461e1d40d216e32c79c704
```

From the string alone, it may be difficult to determine the exact algorithm because several hash types can have similar-looking representations.

For example, `hashID` may report:

```text
MD2
MD5
MD4
Double MD5
LM
RIPEMD-128
Haval-128
Tiger-128
NTLM
Domain Cached Credentials
Domain Cached Credentials 2
...
```

Therefore, hash identification is often **probabilistic**, not absolute.

---

# 🔎 12. Using hashID

**hashID** can compare a supplied hash against a database of known hash formats.

Use:

```bash
hashid -j <hash>
```

Example:

```bash
hashid -j 193069ceb0461e1d40d216e32c79c704
```

The `-j` option additionally provides the corresponding **JtR format** when available.

Example output:

```text
[+] MD5 [JtR Format: raw-md5]
[+] MD4 [JtR Format: raw-md4]
[+] LM [JtR Format: lm]
[+] RIPEMD-128 [JtR Format: ripemd-128]
[+] NTLM [JtR Format: nt]
...
```

> [!warning] 🧠 Do Not Blindly Trust hashID  
> A hash can match multiple possible formats.
> 
> **Context matters.**
> 
> Where the hash was obtained can provide the strongest clue about its actual format.

For example:

```text
Hash found in:
     │
     ├── Linux passwd/shadow → likely Unix-related format
     ├── Windows credential data → likely NT/NTLM-related
     ├── Database → depends on application
     └── Encrypted file → depends on file format
```

In the example provided by the module, the ambiguous hash is ultimately identified as **RIPEMD-128**.

---

# 🎯 13. Specifying the JtR Hash Format

JtR supports **hundreds of hash formats**.

If JtR cannot correctly determine the format, explicitly specify it using:

```bash
john --format=<format> <hash_file>
```

Example:

```bash
john --format=raw-md5 hashes.txt
```

Another example:

```bash
john --format=nt hashes.txt
```

### 🧠 Why This Matters

```text
Unknown Hash
     │
     ▼
Identify / infer format
     │
     ▼
Select JtR format
     │
     ▼
john --format=<format>
     │
     ▼
Correct cracking engine
     │
     ▼
🎯 Attempt cracking
```

> [!important] 🔑 CPTS Skill  
> **Correct hash identification is a prerequisite for effective cracking.**
> 
> If you use the wrong format, JtR may fail to crack the hash even when the password is present in your wordlist.

---

# 🗂️ 14. Important JtR Formats

JtR supports many formats. Some particularly important ones include:

| **Hash format**      | **Example command**                               | **Description**                                                      |
| -------------------- | ------------------------------------------------- | -------------------------------------------------------------------- |
| afs                  | `john --format=afs [...] <hash_file>`             | AFS (Andrew File System) password hashes                             |
| bfegg                | `john --format=bfegg [...] <hash_file>`           | bfegg hashes used in Eggdrop IRC bots                                |
| bf                   | `john --format=bf [...] <hash_file>`              | Blowfish-based crypt(3) hashes                                       |
| bsdi                 | `john --format=bsdi [...] <hash_file>`            | BSDi crypt(3) hashes                                                 |
| crypt(3)             | `john --format=crypt [...] <hash_file>`           | Traditional Unix crypt(3) hashes                                     |
| des                  | `john --format=des [...] <hash_file>`             | Traditional DES-based crypt(3) hashes                                |
| dmd5                 | `john --format=dmd5 [...] <hash_file>`            | DMD5 (Dragonfly BSD MD5) password hashes                             |
| dominosec            | `john --format=dominosec [...] <hash_file>`       | IBM Lotus Domino 6/7 password hashes                                 |
| EPiServer SID hashes | `john --format=episerver [...] <hash_file>`       | EPiServer SID (Security Identifier) password hashes                  |
| hdaa                 | `john --format=hdaa [...] <hash_file>`            | hdaa password hashes used in Openwall GNU/Linux                      |
| hmac-md5             | `john --format=hmac-md5 [...] <hash_file>`        | hmac-md5 password hashes                                             |
| hmailserver          | `john --format=hmailserver [...] <hash_file>`     | hmailserver password hashes                                          |
| ipb2                 | `john --format=ipb2 [...] <hash_file>`            | Invision Power Board 2 password hashes                               |
| krb4                 | `john --format=krb4 [...] <hash_file>`            | Kerberos 4 password hashes                                           |
| krb5                 | `john --format=krb5 [...] <hash_file>`            | Kerberos 5 password hashes                                           |
| LM                   | `john --format=LM [...] <hash_file>`              | LM (Lan Manager) password hashes                                     |
| lotus5               | `john --format=lotus5 [...] <hash_file>`          | Lotus Notes/Domino 5 password hashes                                 |
| mscash               | `john --format=mscash [...] <hash_file>`          | MS Cache password hashes                                             |
| mscash2              | `john --format=mscash2 [...] <hash_file>`         | MS Cache v2 password hashes                                          |
| mschapv2             | `john --format=mschapv2 [...] <hash_file>`        | MS CHAP v2 password hashes                                           |
| mskrb5               | `john --format=mskrb5 [...] <hash_file>`          | MS Kerberos 5 password hashes                                        |
| mssql05              | `john --format=mssql05 [...] <hash_file>`         | MS SQL 2005 password hashes                                          |
| mssql                | `john --format=mssql [...] <hash_file>`           | MS SQL password hashes                                               |
| mysql-fast           | `john --format=mysql-fast [...] <hash_file>`      | MySQL fast password hashes                                           |
| mysql                | `john --format=mysql [...] <hash_file>`           | MySQL password hashes                                                |
| mysql-sha1           | `john --format=mysql-sha1 [...] <hash_file>`      | MySQL SHA1 password hashes                                           |
| NETLM                | `john --format=netlm [...] <hash_file>`           | NETLM (NT LAN Manager) password hashes                               |
| NETLMv2              | `john --format=netlmv2 [...] <hash_file>`         | NETLMv2 (NT LAN Manager version 2) password hashes                   |
| NETNTLM              | `john --format=netntlm [...] <hash_file>`         | NETNTLM (NT LAN Manager) password hashes                             |
| NETNTLMv2            | `john --format=netntlmv2 [...] <hash_file>`       | NETNTLMv2 (NT LAN Manager version 2) password hashes                 |
| NEThalfLM            | `john --format=nethalflm [...] <hash_file>`       | NEThalfLM (NT LAN Manager) password hashes                           |
| md5ns                | `john --format=md5ns [...] <hash_file>`           | md5ns (MD5 namespace) password hashes                                |
| nsldap               | `john --format=nsldap [...] <hash_file>`          | nsldap (OpenLDAP SHA) password hashes                                |
| ssha                 | `john --format=ssha [...] <hash_file>`            | ssha (Salted SHA) password hashes                                    |
| NT                   | `john --format=nt [...] <hash_file>`              | NT (Windows NT) password hashes                                      |
| openssha             | `john --format=openssha [...] <hash_file>`        | OPENSSH private key password hashes                                  |
| oracle11             | `john --format=oracle11 [...] <hash_file>`        | Oracle 11 password hashes                                            |
| oracle               | `john --format=oracle [...] <hash_file>`          | Oracle password hashes                                               |
| pdf                  | `john --format=pdf [...] <hash_file>`             | PDF (Portable Document Format) password hashes                       |
| phpass-md5           | `john --format=phpass-md5 [...] <hash_file>`      | PHPass-MD5 (Portable PHP password hashing framework) password hashes |
| phps                 | `john --format=phps [...] <hash_file>`            | PHPS password hashes                                                 |
| pix-md5              | `john --format=pix-md5 [...] <hash_file>`         | Cisco PIX MD5 password hashes                                        |
| po                   | `john --format=po [...] <hash_file>`              | Po (Sybase SQL Anywhere) password hashes                             |
| rar                  | `john --format=rar [...] <hash_file>`             | RAR (WinRAR) password hashes                                         |
| raw-md4              | `john --format=raw-md4 [...] <hash_file>`         | Raw MD4 password hashes                                              |
| raw-md5              | `john --format=raw-md5 [...] <hash_file>`         | Raw MD5 password hashes                                              |
| raw-md5-unicode      | `john --format=raw-md5-unicode [...] <hash_file>` | Raw MD5 Unicode password hashes                                      |
| raw-sha1             | `john --format=raw-sha1 [...] <hash_file>`        | Raw SHA1 password hashes                                             |
| raw-sha224           | `john --format=raw-sha224 [...] <hash_file>`      | Raw SHA224 password hashes                                           |
| raw-sha256           | `john --format=raw-sha256 [...] <hash_file>`      | Raw SHA256 password hashes                                           |
| raw-sha384           | `john --format=raw-sha384 [...] <hash_file>`      | Raw SHA384 password hashes                                           |
| raw-sha512           | `john --format=raw-sha512 [...] <hash_file>`      | Raw SHA512 password hashes                                           |
| salted-sha           | `john --format=salted-sha [...] <hash_file>`      | Salted SHA password hashes                                           |
| sapb                 | `john --format=sapb [...] <hash_file>`            | SAP CODVN B (BCODE) password hashes                                  |
| sapg                 | `john --format=sapg [...] <hash_file>`            | SAP CODVN G (PASSCODE) password hashes                               |
| sha1-gen             | `john --format=sha1-gen [...] <hash_file>`        | Generic SHA1 password hashes                                         |
| skey                 | `john --format=skey [...] <hash_file>`            | S/Key (One-time password) hashes                                     |
| ssh                  | `john --format=ssh [...] <hash_file>`             | SSH (Secure Shell) password hashes                                   |
| sybasease            | `john --format=sybasease [...] <hash_file>`       | Sybase ASE password hashes                                           |
| xsha                 | `john --format=xsha [...] <hash_file>`            | xsha (Extended SHA) password hashes                                  |
| zip                  | `john --format=zip [...] <hash_file>`             | ZIP (WinZip) password hashes                                         |
The module provides a much larger list of supported formats.

---

# 📦 15. Cracking Password-Protected Files

JtR isn't limited to standalone password hashes.

It can also attack **password-protected or encrypted files**.

The general process is:

```text
Password-Protected File
          │
          ▼
       *2john Tool
          │
          ▼
    JtR-Compatible Hash
          │
          ▼
     John the Ripper
          │
          ▼
     Password Cracking
```

### General Syntax

```bash
<tool> <file_to_crack> > file.hash
```

For example:

```bash
zip2john protected.zip > file.hash
```

Then:

```bash
john file.hash
```

> [!important] 🧠 The `2john` Concept  
> The `2john` utilities **do not crack the file themselves**.
> 
> Their job is to extract/convert the relevant authentication data into a format that **John the Ripper can attack**.

---

# 🛠️ 16. Common `*2john` Utilities

|Tool|Purpose|
|---|---|
|`pdf2john`|Convert PDF password data|
|`ssh2john`|Convert SSH private keys|
|`mscash2john`|Convert MS Cache hashes|
|`keychain2john`|Convert OS X Keychain files|
|`rar2john`|Convert RAR archives|
|`pfx2john`|Convert PKCS#12 files|
|`truecrypt_volume2john`|Convert TrueCrypt volumes|
|`keepass2john`|Convert KeePass databases|
|`vncpcap2john`|Convert VNC PCAP files|
|`putty2john`|Convert PuTTY private keys|
|`zip2john`|Convert ZIP archives|
|`hccap2john`|Convert WPA/WPA2 handshake captures|
|`office2john`|Convert MS Office documents|
|`wpa2john`|Convert WPA/WPA2 handshake data|

---

# 🔍 17. Finding Available `2john` Tools

On Pwnbox, you can locate installed `*2john` utilities using:

```bash
locate *2john*
```

Example results include:

```text
/usr/bin/bitlocker2john
/usr/bin/dmg2john
/usr/bin/gpg2john
/usr/bin/hccap2john
/usr/bin/keepass2john
/usr/bin/putty2john
/usr/bin/rar2john
/usr/bin/zip2john
/usr/share/john/1password2john.py
/usr/share/john/7z2john.pl
...
```

> [!tip] 💡 Useful Habit  
> If you encounter an unfamiliar password-protected file, first check whether JtR has a corresponding `*2john` conversion utility.

---

# ⚔️ 18. Complete JtR Workflow

A useful CPTS mental model is:

```text
             🔎 Obtain Credential / File
                       │
                       ▼
              Identify Hash / Format
                       │
                       ▼
              ┌────────────────────┐
              │ Is it a file?      │
              └─────────┬──────────┘
                   Yes  │  No
                        │
          ┌─────────────┘
          ▼
       *2john
          │
          ▼
    JtR-compatible hash
          │
          └──────────────┐
                         ▼
                 Choose Attack
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Single         Wordlist      Incremental
          │              │              │
          │           + Rules           │
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                    🎯 Crack Hash
                         │
                         ▼
                  john --show
```

---

# 🧠 19. CPTS Exam Mental Model

> [!important] 🎯 Remember This Sequence

```text
1️⃣ WHAT DO I HAVE?
   └── Hash? File? Credential?

2️⃣ WHAT FORMAT IS IT?
   └── Identify using structure + context + hashID

3️⃣ CAN I CONVERT IT?
   └── For protected files → appropriate *2john utility

4️⃣ WHAT INFORMATION DO I KNOW?
   └── Username / names / paths / password patterns

5️⃣ WHAT IS THE MOST EFFICIENT ATTACK?
   ├── Single crack
   ├── Wordlist
   ├── Wordlist + rules
   └── Incremental

6️⃣ DOES JtR KNOW THE FORMAT?
   └── If not → --format=<format>

7️⃣ DID IT CRACK?
   └── john --show <hash_file>
```

> [!success] 🔑 Key Takeaways
> 
> - 🧠 **John the Ripper** is a password-cracking framework with multiple attack modes.
>     
> - 👤 `--single` uses information from the target account to generate candidates.
>     
> - 📖 `--wordlist=<file>` performs dictionary/wordlist cracking.
>     
> - 🛠️ `--rules` transforms wordlist entries into additional candidates.
>     
> - 💥 `--incremental` performs statistical brute-force-style candidate generation.
>     
> - 🔎 `hashid -j <hash>` helps identify possible hash formats and their JtR equivalents.
>     
> - 🎯 `john --format=<format>` explicitly tells JtR which cracking format to use.
>     
> - 📦 `*2john` utilities convert password-protected files into JtR-compatible hashes.
>     
> - 👁️ `john --show <hash_file>` displays cracked passwords.
>     
> - 🧩 **Context is critical** when identifying ambiguous hashes.
>     
> 
> **Core principle:**
> 
> ```text
> Identify → Convert → Choose the smartest attack → Crack → Show
> ```