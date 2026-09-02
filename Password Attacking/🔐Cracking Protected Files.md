# 🔐 1. Cracking Protected Files

> [!info] 🎯 Objective  
> Password-protected files and archives can contain **valuable sensitive information**. Instead of attacking the file directly, the general methodology is:
> 
> **Find protected file → Extract password hash → Crack hash offline → Recover password → Access file**

```text
                    🔎 FILE DISCOVERY
                          │
                          ▼
              Identify protected file
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
       SSH Private Key          Office / PDF
             │                         │
             ▼                         ▼
         ssh2john.py       office2john.py / pdf2john.py
             │                         │
             └────────────┬────────────┘
                          ▼
                  🔑 Extract hash
                          │
                          ▼
                John the Ripper
                          │
                          ▼
                 💥 Password found
                          │
                          ▼
                  Access protected data
```

> [!important] 🧠 Core Mental Model  
> A password-protected file is often **not cracked directly**.
> 
> Instead, tools such as `ssh2john.py`, `office2john.py`, and `pdf2john.py` convert the protected file into a **crackable hash representation**.
> 
> ```text
> Protected File
>      ↓
> Extraction Tool
>      ↓
> Crackable Hash
>      ↓
> John / Hashcat
>      ↓
> Password
> ```

---

# 🛡️ 2. Why Protected Files Matter

Encryption is frequently neglected in both **private and professional environments**.

Sensitive information may include:

- 📄 Job applications
    
- 💳 Account statements
    
- 📑 Contracts
    
- 🏢 Internal company documents
    
- 🔐 Credentials
    
- 📊 Reports
    
- 🗂️ Configuration information
    

Even when organizations use encryption, employees may still transmit sensitive information through insecure channels such as email.

> [!warning] ⚠️ Security Perspective  
> An encrypted document may be secure against direct access, but a **weak password/passphrase can become the weakest link**.
> 
> Strong encryption + weak password = potentially recoverable data.

---

# 🔑 3. Symmetric vs. Asymmetric Encryption

Understanding the encryption model is important when dealing with protected files and keys.

|Feature|🔐 Symmetric|🔑 Asymmetric|
|---|---|---|
|Keys|One key|Public + private key|
|Encryption|Same key encrypts/decrypts|Public key encrypts|
|Decryption|Same key decrypts|Private key decrypts|
|Common use|File/folder encryption|Secure communication / key exchange|
|Example|AES-256|RSA / ECC|
|Main secret|Encryption key|Private key|

### 🔐 Symmetric Encryption

The **same key** is used for encryption and decryption.

```text
        🔑 Same Key
            │
            ▼
Plaintext ──► 🔒 Encryption ──► Ciphertext
                                  │
                                  ▼
                         🔓 Decryption
                                  │
                                  ▼
                              Plaintext
```

Example:

```text
AES-256
```

### 🔑 Asymmetric Encryption

Two different keys are used:

```text
              Recipient
             ┌──────────┐
             │ Public 🔓 │
             │ Private 🔑│
             └──────────┘
                  ▲
                  │
Sender ──encrypt──┘
       using recipient's
       PUBLIC key

Recipient decrypts using
PRIVATE key
```

> [!tip] 🧠 Remember  
> **Public key → encrypt**
> 
> **Private key → decrypt**
> 
> The private key must remain secret.

---

# 🔎 4. Hunting for Encrypted / Protected Files

There are hundreds of file extensions, so knowing every encrypted format is unrealistic.

A useful approach is to search the filesystem for **common document extensions** and investigate unfamiliar extensions when encountered.

### 🐧 Searching for Common Document Types

```bash
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*");do
    echo -e "\nFile extension: " $ext
    find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done
```

### 🔍 What the command does

```text
for ext in ...
      │
      ├── Iterate through common extensions
      │
      ▼
find /
      │
      ├── Search entire filesystem
      │
      ▼
2>/dev/null
      │
      └── Hide permission/error messages
      │
      ▼
grep -v "lib\|fonts\|share\|core"
      │
      └── Remove noisy/uninteresting paths
```

Example results:

```text
File extension: .od*

/home/cry0l1t3/Docs/document-temp.odt
/home/cry0l1t3/Docs/product-improvements.odp
/home/cry0l1t3/Docs/mgmt-spreadsheet.ods
```

### 🧠 Important Extensions

|Extension|Typical format|
|---|---|
|`.doc`, `.docx`|Microsoft Word|
|`.xls`, `.xlsx`|Microsoft Excel|
|`.pdf`|Portable Document Format|
|`.odt`|OpenDocument Text|
|`.odp`|OpenDocument Presentation|
|`.ods`|OpenDocument Spreadsheet|
|`.ppt`, `.pptx`|Microsoft PowerPoint|

> [!tip] 🔎 Unknown Extension?  
> Don't try to memorize hundreds of extensions.
> 
> **Research the extension** using documentation or search engines to determine what technology created it and whether it supports password protection/encryption.

---

# 🔑 5. Hunting for SSH Private Keys

SSH private keys are different because they **may not have a file extension**.

For example:

```text
~/.ssh/id_rsa
~/.ssh/id_ed25519
SSH.private
```

Therefore, searching by filename extension isn't enough.

Instead, search for **known key headers**.

### 🧬 SSH Private-Key Header

SSH private keys commonly contain a header such as:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
```

or older PEM-style keys:

```text
-----BEGIN RSA PRIVATE KEY-----
```

### 🔍 Recursive Search

```bash
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
```

Example:

```text
/home/jsmith/.ssh/id_ed25519:1:-----BEGIN OPENSSH PRIVATE KEY-----
/home/jsmith/.ssh/SSH.private:1:-----BEGIN RSA PRIVATE KEY-----
/home/jsmith/Documents/id_rsa:1:-----BEGIN OPENSSH PRIVATE KEY-----
```

### 🧠 Mental Model

```text
No extension?
     │
     ▼
Search for known file signatures
     │
     ▼
SSH key header
     │
     ├── BEGIN OPENSSH PRIVATE KEY
     ├── BEGIN RSA PRIVATE KEY
     └── Other PRIVATE KEY formats
```

> [!important] 🎯 Post-Exploitation Value  
> Searching for private keys can reveal credentials that allow:
> 
> - SSH access to other systems
>     
> - Lateral movement
>     
> - Persistence
>     
> - Access to additional user accounts
>     
> - Access to sensitive infrastructure
>     

---

# 🔐 6. Determining Whether an SSH Key Is Encrypted

Older PEM-format SSH keys can sometimes reveal their encryption state directly through their headers.

Example:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2109D25CC91F8DBFCEB0F7589066B2CC
```

The important line is:

```text
Proc-Type: 4,ENCRYPTED
```

This indicates that the private key is encrypted.

> [!warning] ⚠️ Modern SSH Keys  
> Modern SSH private keys may **look the same whether encrypted or not**.
> 
> Therefore, don't rely only on visually inspecting the file header.

---

# 🧪 7. Testing an SSH Key with `ssh-keygen`

A practical way to determine whether a key requires a passphrase is to attempt to read its public key.

```bash
ssh-keygen -yf ~/.ssh/id_ed25519
```

If the key is not protected:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIpNefJd834VkD5iq+22Zh59Gzmmtzo6rAffCx2UtaS6
```

If the key is protected:

```bash
ssh-keygen -yf ~/.ssh/id_rsa
```

You will be prompted:

```text
Enter passphrase for "/home/jsmith/.ssh/id_rsa":
```

### 🧠 Interpretation

|Result|Meaning|
|---|---|
|Public key printed immediately|Key does not require a passphrase|
|`Enter passphrase`|Key is protected by a passphrase|

---

# 💥 8. Cracking Encrypted SSH Keys

John the Ripper (**JtR**) provides utilities for converting many protected files into crackable formats.

These tools commonly follow the naming convention:

```text
<format>2john
```

### 🔎 Find Available `2john` Tools

```bash
locate *2john*
```

Example:

```text
/usr/bin/bitlocker2john
/usr/bin/dmg2john
/usr/bin/gpg2john
/usr/bin/hccap2john
/usr/bin/keepass2john
/usr/bin/putty2john
/usr/bin/racf2john
/usr/bin/rar2john
/usr/bin/uaf2john
/usr/bin/vncpcap2john
/usr/bin/wlanhcx2john
/usr/bin/wpapcap2john
/usr/bin/zip2john
/usr/share/john/1password2john.py
/usr/share/john/7z2john.pl
/usr/share/john/DPAPImk2john.py
/usr/share/john/androidbackup2john.py
```

> [!success] 🧠 Key Pattern
> 
> ```text
> File format
>      ↓
> <format>2john
>      ↓
> Extract crackable representation
>      ↓
> John the Ripper
> ```

---

# 🔓 9. SSH Key → Hash → Password

For an encrypted SSH private key:

```text
SSH.private
     │
     ▼
ssh2john.py
     │
     ▼
ssh.hash
     │
     ▼
John the Ripper
     │
     ▼
Passphrase
```

### Step 1 — Extract the Hash

```bash
ssh2john.py SSH.private > ssh.hash
```

This does **not** crack the key.

It converts the protected SSH key into a format JtR can attack.

### Step 2 — Crack with a Wordlist

```bash
john --wordlist=rockyou.txt ssh.hash
```

Example result:

```text
1234         (SSH.private)
```

> [!important] 🔑 Result  
> The recovered password/passphrase is:
> 
> ```text
> 1234
> ```

### Step 3 — Display Cracked Passwords

```bash
john ssh.hash --show
```

Output:

```text
SSH.private:1234

1 password hash cracked, 0 left
```

---

# 📄 10. Cracking Password-Protected Office Documents

Microsoft Office documents are commonly encountered during engagements.

Examples:

```text
.doc
.docx
.xls
.xlsx
.ppt
.pptx
```

JtR provides:

```text
office2john.py
```

to extract password hashes from common Office formats.

### 🔓 Standard Workflow

```text
Protected.docx
      │
      ▼
office2john.py
      │
      ▼
protected-docx.hash
      │
      ▼
John + Wordlist
      │
      ▼
Recovered password
```

### Step 1 — Extract

```bash
office2john.py Protected.docx > protected-docx.hash
```

### Step 2 — Crack

```bash
john --wordlist=rockyou.txt protected-docx.hash
```

### Step 3 — Show Result

```bash
john protected-docx.hash --show
```

Example:

```text
Protected.docx:1234

1 password hash cracked, 0 left
```

---

# 📑 11. Cracking Password-Protected PDFs

The process is almost identical to Office documents.

The main difference is the extraction tool:

```text
pdf2john.py
```

### 🔓 Workflow

```text
PDF.pdf
   │
   ▼
pdf2john.py
   │
   ▼
pdf.hash
   │
   ▼
John the Ripper
   │
   ▼
Password
```

### Step 1 — Extract

```bash
pdf2john.py PDF.pdf > pdf.hash
```

### Step 2 — Crack

```bash
john --wordlist=rockyou.txt pdf.hash
```

### Step 3 — Show Password

```bash
john pdf.hash --show
```

Example:

```text
PDF.pdf:1234

1 password hash cracked, 0 left
```

---

# 🧰 12. File-Type → Extraction Tool Cheat Sheet

|🔒 Protected Object|🛠️ Extraction Tool|🎯 Crack With|
|---|---|---|
|SSH private key|`ssh2john.py`|John / Hashcat|
|Microsoft Office|`office2john.py`|John / Hashcat|
|PDF|`pdf2john.py`|John / Hashcat|
|ZIP|`zip2john`|John / Hashcat|
|RAR|`rar2john`|John / Hashcat|
|KeePass|`keepass2john`|John / Hashcat|
|BitLocker|`bitlocker2john`|John / Hashcat|
|GPG|`gpg2john`|John / Hashcat|
|7-Zip|`7z2john.pl`|John / Hashcat|
|PuTTY key|`putty2john`|John / Hashcat|

> [!tip] 🧠 Exam Memory Trick  
> Think:
> 
> **`THING → thing2john → hash → john`**
> 
> Example:
> 
> ```text
> SSH       → ssh2john.py
> Office    → office2john.py
> PDF       → pdf2john.py
> ZIP       → zip2john
> RAR       → rar2john
> KeePass   → keepass2john
> ```

---

# 🧠 13. Why Wordlists Matter

The extraction step is often straightforward.

The harder problem is usually **password candidate generation**.

```text
Protected File
      │
      ▼
Extract Hash
      │
      ▼
Password Candidates
      │
      ▼
   Crack Attempt
      │
      ├── ❌ Wrong
      │     ↓
      │  Next candidate
      │
      └── ✅ Correct
             ↓
         Password found
```

A standard wordlist such as:

```text
rockyou.txt
```

may work for weak or common passwords.

However, real-world passwords may be:

- Long
    
- Randomly generated
    
- Unique
    
- Complex
    
- Based on personal information
    
- Generated from passphrases
    
- Modified versions of known words
    

> [!warning] ⚠️ Why `rockyou.txt` Isn't Always Enough  
> Public wordlists contain known/common passwords and are often insufficient against strong passwords.
> 
> If the target password isn't present in the wordlist, simply running the same attack faster will not help.

---

# 🎯 14. Password Mutation & Targeted Wordlists

When standard lists fail, password **generation and mutation** become increasingly important.

For example:

```text
company
   │
   ├── Company
   ├── company123
   ├── Company123!
   ├── C0mpany123!
   └── Company2026!
```

A targeted wordlist can be much more effective than blindly using a massive generic list.

### 🧠 Attack Methodology

```text
             OSINT / Context
                    │
                    ▼
             Candidate Words
                    │
                    ▼
           Wordlist Generation
                    │
                    ▼
             Password Mutations
                    │
                    ▼
             Extracted Hash
                    │
                    ▼
             John / Hashcat
                    │
                    ▼
              🔓 Password
```

> [!important] 🧠 CPTS Perspective  
> The ability to **generate good password candidates** is often more important than simply knowing how to execute John.
> 
> The cracking tool performs the testing; **your candidate-generation strategy determines what gets tested**.

---

# 🧠 15. Key Takeaways / Mental Model

> [!success] 🎯 CPTS Exam Mental Model

```text
🔎 FIND
Find protected documents, archives, keys, etc.
        ↓
🧬 IDENTIFY
Determine the file type / protection mechanism.
        ↓
🛠️ EXTRACT
Use the appropriate *2john utility.
        ↓
🔑 HASH
Obtain a crackable representation.
        ↓
📚 WORDLIST
Use rockyou.txt or a targeted/custom list.
        ↓
💥 CRACK
John the Ripper or Hashcat.
        ↓
🔓 RECOVER
Obtain the password/passphrase.
        ↓
📂 ACCESS
Use the recovered credential to access the protected data.
```

### ⚡ Essential Commands

```bash
# Find common document types
find / -name "*.pdf" 2>/dev/null

# Search filesystem for SSH private keys
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null

# Test SSH key
ssh-keygen -yf ~/.ssh/id_rsa

# Find available *2john utilities
locate *2john*

# SSH private key
ssh2john.py SSH.private > ssh.hash
john --wordlist=rockyou.txt ssh.hash
john ssh.hash --show

# Microsoft Office
office2john.py Protected.docx > protected-docx.hash
john --wordlist=rockyou.txt protected-docx.hash
john protected-docx.hash --show

# PDF
pdf2john.py PDF.pdf > pdf.hash
john --wordlist=rockyou.txt pdf.hash
john pdf.hash --show
```

> [!important] 🧠 Remember the Separation of Tasks
> 
> **Extraction ≠ Cracking**
> 
> ```text
> ssh2john.py
> office2john.py
> pdf2john.py
>       │
>       ▼
> Extract crackable hash
>       │
>       ▼
> john / hashcat
>       │
>       ▼
> Crack password
> ```
> 
> The `*2john` tools **prepare the target**.  
> John/Hashcat **perform the password attack**.

> [!success] 🏆 Final Takeaway  
> Password-protected files should be treated as potential sources of sensitive information during post-exploitation.
> 
> **Find → Identify → Extract → Crack → Access**
> 
> The most important practical skill is recognizing the file type and knowing which extraction utility converts it into a format that JtR or Hashcat can attack.