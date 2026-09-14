# 🔐 Protected File Transfers

> [!info]+ 📌 Why Protect Files Before Transfer?  
> During a penetration test, we may encounter highly sensitive information such as:
> 
> - 👤 User lists
>     
> - 🔑 Credentials
>     
> - 🗄️ `NTDS.dit`
>     
> - 🔎 Enumeration results
>     
> - 🏢 Active Directory information
>     
> - 🌐 Network infrastructure information
>     
> 
> If this information is transferred over an insecure connection, an attacker or unauthorized party who intercepts the traffic may be able to read it.
> 
> Therefore:
> 
> ```text
> Sensitive Data
>      │
>      ▼
> 🔐 Encrypt
>      │
>      ▼
> 📦 Transfer
>      │
>      ▼
> 🖥️ Destination
>      │
>      ▼
> 🔓 Decrypt
> ```

> [!warning]+ ⚠️ Professional Responsibility  
> Data leakage during a penetration test can have serious consequences for:
> 
> - 👨‍💻 The penetration tester
>     
> - 🏢 The security company
>     
> - 🤝 The client
>     
> 
> Unless specifically authorized by the client, sensitive information such as **PII, financial information, credit-card data, or trade secrets should not be exfiltrated**.
> 
> When testing DLP or egress controls, use **dummy data that imitates the protected information** instead.

---

# 🔒 1️⃣ Encryption vs Encrypted Transport

There are two related approaches:

### 🌐 Encrypted Connection

```text
Sensitive File
      │
      ▼
🔐 HTTPS / SFTP / SSH
      │
      ▼
Remote Host
```

The **connection itself** provides encryption.

Examples:

- 🔐 SSH
    
- 📁 SFTP
    
- 🌐 HTTPS
    

### 📦 Encrypt the File First

```text
Sensitive File
      │
      ▼
🔐 File Encryption
      │
      ▼
Encrypted File
      │
      ▼
📡 Transfer
      │
      ▼
🔓 Decrypt
```

> [!tip]+ 💡 Why Encrypt the File?  
> Sometimes secure transport options such as SSH, SFTP, or HTTPS are unavailable.
> 
> In that situation, encrypting the file **before transferring it** provides protection against the contents being read if the transfer is intercepted.

---

# 🪟 2️⃣ File Encryption on Windows

One method described in the material is:

```text
Invoke-AESEncryption.ps1
```

> [!info]+ 🧠 What It Provides  
> The PowerShell script can encrypt:
> 
> - 📝 Strings
>     
> - 📄 Files
>     
> 
> It uses **AES encryption** and can output encrypted text as **Base64** or encrypted files with a `.aes` extension.

---

# 📜 3️⃣ Invoke-AESEncryption

The script exposes the following parameters:

```text
Invoke-AESEncryption
        │
        ├── -Mode Encrypt / Decrypt
        │
        ├── -Key <password>
        │
        ├── -Text <string>
        │
        └── -Path <file>
```

### 🧩 Main Parameters

|Parameter|Purpose|
|---|---|
|`-Mode`|Select `Encrypt` or `Decrypt`|
|`-Key`|Password/key used for encryption|
|`-Text`|Encrypt/decrypt a string|
|`-Path`|Encrypt/decrypt a file|

> [!note]+ 🔐 AES Configuration  
> The script configures:
> 
> ```text
> Algorithm    → AES
> Mode         → CBC
> Block Size   → 128 bits
> Key Size     → 256 bits
> ```
> 
> The supplied key is processed using SHA-256 to produce the AES key.

---

# 📥 4️⃣ Getting the Encryption Script onto Windows

The encryption script itself can be transferred using **any of the previously discussed file-transfer techniques**.

```text
Previous Transfer Method
          │
          ▼
Invoke-AESEncryption.ps1
          │
          ▼
Windows Host
          │
          ▼
Import PowerShell Module
```

### 📦 Import the Module

```powershell
Import-Module .\Invoke-AESEncryption.ps1
```

> [!success]+ ✅ After Importing  
> The `Invoke-AESEncryption` function becomes available in the PowerShell session.

---

# 🔐 5️⃣ Encrypt a File on Windows

Example:

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt
```

Output:

```text
File encrypted to C:\htb\scan-results.txt.aes
```

### 📁 Result

```text
scan-results.txt
       │
       │ 🔐 Encrypt
       ▼
scan-results.txt.aes
```

The encrypted file uses the original filename with:

```text
.aes
```

added to the end.

---

# 📊 6️⃣ Before vs After Encryption

Example from the material:

```text
Before:

scan-results.txt
     │
     └── 1724 bytes


After:

scan-results.txt.aes
     │
     └── 3448 bytes
```

> [!info]+ 🧠 Important  
> Encryption changes the file contents into ciphertext. The resulting encrypted file can then be transferred using one of the previously discussed transfer methods.

---

# 🔑 7️⃣ Strong & Unique Encryption Passwords

> [!danger]+ 🚨 Critical  
> Use a **strong and unique password for every client/environment**.
> 
> Do **not** reuse one password across multiple penetration tests.

### ❌ Bad Approach

```text
Client A ──┐
Client B ──┼──> Same Password ❌
Client C ──┘
```

If that password is leaked or cracked, files from multiple environments could potentially be exposed.

### ✅ Better Approach

```text
Client A ──> 🔑 Unique Key A
Client B ──> 🔑 Unique Key B
Client C ──> 🔑 Unique Key C
```

> [!tip]+ 🧠 Why?  
> A compromised password should not expose encrypted information belonging to other clients.

---

# 🐧 8️⃣ File Encryption on Linux

## 🔐 OpenSSL

> [!info]+ 📌 OpenSSL  
> **OpenSSL** is frequently included in Linux distributions.
> 
> It is commonly used by system administrators for tasks such as generating security certificates.
> 
> It can also encrypt files before transferring them.

---

# 🔒 9️⃣ Encrypt a File with OpenSSL

Example:

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 \
-in /etc/passwd \
-out passwd.enc
```

The command prompts for an encryption password:

```text
enter aes-256-cbc encryption password:
Verifying - enter aes-256-cbc encryption password:
```

---

# 🧩 🔟 OpenSSL Command Breakdown

```text
openssl
   │
   ▼
enc
   │
   ├── -aes256
   │      └── AES-256 cipher
   │
   ├── -iter 100000
   │      └── Set iteration count
   │
   ├── -pbkdf2
   │      └── Use PBKDF2
   │
   ├── -in /etc/passwd
   │      └── Input file
   │
   └── -out passwd.enc
          └── Output encrypted file
```

### ⚡ Quick Reference

|Option|Meaning|
|---|---|
|`enc`|OpenSSL encryption utility|
|`-aes256`|Use AES-256|
|`-iter 100000`|Set password-based derivation iterations|
|`-pbkdf2`|Use PBKDF2|
|`-in`|Input file|
|`-out`|Output file|

> [!note]+ 🔐 Password Protection  
> OpenSSL prompts for a password when the command is executed.
> 
> The material emphasizes using a **strong and unique password** to reduce the risk of brute-force cracking if an unauthorized party obtains the encrypted file.

---

# 🔓 1️⃣1️⃣ Decrypt with OpenSSL

To decrypt the encrypted file:

```bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 \
-in passwd.enc \
-out passwd
```

You will be prompted for the decryption password:

```text
enter aes-256-cbc decryption password:
```

### 🔄 Encryption / Decryption

```text
             🔐 ENCRYPT
                  │
                  ▼
        /etc/passwd
                  │
                  ▼
             AES-256
                  │
                  ▼
            passwd.enc
                  │
                  │ 🔑 Password
                  ▼
             🔓 DECRYPT
                  │
                  ▼
              passwd
```

---

# 🆚 1️⃣2️⃣ Windows vs Linux

||🪟 Windows|🐧 Linux|
|---|---|---|
|Main tool|PowerShell|OpenSSL|
|Example|`Invoke-AESEncryption`|`openssl enc`|
|Algorithm|AES|AES-256 example|
|Input|String / File|File|
|Output|`.aes` file / Base64 text|Encrypted file|
|Password|Supplied as `-Key`|Interactive prompt|

---

# 🌐 1️⃣3️⃣ Transfer After Encryption

Once the file has been encrypted, we can use the previously discussed transfer methods.

```text
                 📄 Sensitive File
                         │
                         ▼
                    🔐 Encrypt
                         │
                         ▼
                  📦 Encrypted File
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       🐚 Netcat       🌐 HTTP       🖥️ RDP
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                  🖥️ Destination
                         │
                         ▼
                    🔓 Decrypt
                         │
                         ▼
                  📄 Original Data
```

> [!success]+ 🔐 Preferred Transport  
> Even when a file is encrypted, the material recommends using a **secure transport mechanism** when possible:
> 
> - 🔐 HTTPS
>     
> - 📁 SFTP
>     
> - 🔑 SSH
>     
> 
> This provides an additional layer of protection.

---

# 🛡️ 1️⃣4️⃣ Defense-in-Depth for File Transfers

> [!abstract]+ 🧠 Two Layers Are Better Than One

### Layer 1 — Encrypt the File

```text
📄 Data
 ↓
🔐 AES
 ↓
📦 Ciphertext
```

### Layer 2 — Encrypt the Transport

```text
📦 Ciphertext
 ↓
🔐 HTTPS / SSH / SFTP
 ↓
🌐 Network
```

### Combined

```text
        📄 Sensitive Data
                │
                ▼
             🔐 AES
                │
                ▼
       📦 Encrypted File
                │
                ▼
       🔐 HTTPS / SSH / SFTP
                │
                ▼
            🌐 Network
                │
                ▼
         🖥️ Destination
                │
                ▼
             🔓 AES
                │
                ▼
        📄 Original Data
```

> [!tip]+ 💡 Key Concept  
> **File encryption protects the data itself.**
> 
> **Encrypted transport protects the connection carrying the data.**

---

# ⚡ 1️⃣5️⃣ Quick Command Reference

## 🪟 Windows — Import

```powershell
Import-Module .\Invoke-AESEncryption.ps1
```

## 🔐 Windows — Encrypt File

```powershell
Invoke-AESEncryption `
    -Mode Encrypt `
    -Key "<STRONG_UNIQUE_KEY>" `
    -Path .\scan-results.txt
```

## 🔓 Windows — Decrypt File

```powershell
Invoke-AESEncryption `
    -Mode Decrypt `
    -Key "<STRONG_UNIQUE_KEY>" `
    -Path .\scan-results.txt.aes
```

---

## 🐧 Linux — Encrypt

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 \
-in <INPUT_FILE> \
-out <OUTPUT_FILE>
```

## 🔓 Linux — Decrypt

```bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 \
-in <ENCRYPTED_FILE> \
-out <OUTPUT_FILE>
```

---

# 🧠 1️⃣6️⃣ Mental Model

> [!success]+ 🎯 Remember This

```text
       🔴 SENSITIVE DATA
               │
               ▼
        "Can I use secure
         transport?"
               │
        ┌──────┴──────┐
       YES            NO
        │              │
        ▼              ▼
   🔐 HTTPS/SSH/     🔐 Encrypt
      SFTP            File First
        │              │
        └──────┬───────┘
               ▼
            📦 Transfer
               │
               ▼
          🖥️ Destination
               │
               ▼
            🔓 Decrypt
```

> [!warning]+ 🚨 Security Rules to Remember
> 
> 1. 🔐 Protect sensitive data before transferring it.
>     
> 2. 🔑 Use **strong, unique passwords/keys**.
>     
> 3. 🏢 Use a different key for each client/environment.
>     
> 4. 🚫 Don't exfiltrate real PII, financial information, or trade secrets without explicit authorization.
>     
> 5. 🧪 Use dummy data when testing DLP/egress controls.
>     
> 6. 🌐 Prefer **HTTPS, SFTP, or SSH** when available.
>     
> 7. 🛡️ Think of encryption as part of your overall data-protection process.
>     

---

# 🏁 Final Takeaway

> [!quote]+ 💡 The Main Idea  
> ==A file transfer isn't complete from a security perspective just because the file reached the destination.==
> 
> You also need to consider **what happens if someone intercepts the transfer**.
> 
> ```text
> 📄 Sensitive Data
>        │
>        ▼
>      🔐 Protect
>        │
>        ├── 🔐 Encrypt File
>        │
>        └── 🌐 Secure Transport
>                │
>                ▼
>            📦 Transfer
>                │
>                ▼
>           🖥️ Destination
>                │
>                ▼
>             🔓 Decrypt
> ```
> 
> **Windows:** `Invoke-AESEncryption`
> 
> **Linux:** `openssl enc`
> 
> **Preferred transport:** `HTTPS` / `SFTP` / `SSH`
> 
> The goal is to ensure that sensitive assessment data remains protected **both during transfer and at rest while encrypted**.