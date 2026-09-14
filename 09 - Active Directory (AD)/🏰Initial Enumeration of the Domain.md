# 🏰 1. Initial Enumeration of the Domain

> [!info] 🎯 Objective  
> At the beginning of an **Active Directory-focused penetration test**, the goal is to progressively build a picture of the internal environment:
> 
> **NETWORK → HOSTS → SERVICES → DOMAIN CONTROLLERS → USERS → VULNERABILITIES → FOOTHOLD**
> 
> We start with little or no authenticated access and progressively gather enough information to obtain a **domain user account, NTLM credentials/hashes, SYSTEM access, or another foothold**.

---

# 🧭 2. Understand the Assessment Setup

Before touching the network, understand **where the attack host is located, what credentials are available, what is in scope, and whether stealth matters**.

### 🖥️ Common Internal Pentest Starting Positions

|Setup|Description|
|---|---|
|Linux pentest VM|VM inside the client's network that calls back to a jump host over VPN/SSH|
|Physical device|Device plugged directly into Ethernet and connected back to the tester|
|Physical laptop|Tester physically present in the office with a laptop connected to Ethernet|
|Cloud VM|Linux VM in Azure/AWS with internal network access|
|VPN|Tester connects remotely to the internal network|
|Corporate laptop|Testing from a corporate laptop connected to the client's VPN|
|Managed workstation|Windows workstation inside the office, potentially with limited/no Internet|
|VDI|Virtual desktop such as Citrix with access to the internal environment|

### 🎭 Testing Approaches

```text
                    PENTEST TYPE
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Black Box      Grey Box      Credentialed
          │              │              │
     Discover       Known ranges      Known user/
     everything     / IPs             credentials
     yourself
```

### 🔊 Evasive vs Non-Evasive

|Approach|Goal|
|---|---|
|**Evasive**|Mimic an attacker while attempting to avoid detection|
|**Non-evasive**|Perform comprehensive testing without attempting to hide activity|
|**Hybrid evasive**|Start quietly, determine detection thresholds, then increase activity|

> [!warning] ⚠️ Scope Comes First  
> A technically successful attack can still be a **failed penetration test** if it violates scope or causes unnecessary disruption.
> 
> Always understand:
> 
> - In-scope IP ranges
>     
> - Out-of-scope systems
>     
> - Allowed attack techniques
>     
> - Whether exploitation is permitted
>     
> - Whether stealth is required
>     
> - Whether denial-of-service-like behavior is prohibited
>     

---

# 🏢 3. Inlanefreight Assessment Setup

The customer selected a **comprehensive grey-box, non-evasive assessment**.

```text
                    INLANEFREIGHT
                         │
             ┌───────────┴───────────┐
             │                       │
       Internal Network         Attack Host
       172.16.5.0/23                 │
                                     │
                              Linux Pentest VM
                                     │
                              SSH / Jump Host
```

### 📋 Provided Information

- Custom pentest VM inside the internal network
    
- Windows attack host available
    
- Standard domain user:
    
    ```text
    htb-student
    ```
    
- Network range:
    
    ```text
    172.16.5.0/23
    ```
    
- **Grey-box** testing
    
- **Non-evasive** testing
    
- No detailed internal network map
    
- Testing begins from an **unauthenticated perspective**
    

> [!important] 🧠 Why Start Unauthenticated?  
> This simulates what an attacker might accomplish after gaining initial network access through:
> 
> - Phishing
>     
> - Physical access
>     
> - Rogue employee
>     
> - Wireless access
>     
> - Internet compromise
>     
> - Other unauthorized access
>     
> 
> If this phase succeeds, the client may provide credentials or a domain-joined system so the assessment can progress faster.

---

# 🎯 4. Initial Enumeration Objectives

At this stage, we are trying to answer four major questions:

|**Data PointDescription**||
|---|---|
|`AD Users`|We are trying to enumerate valid user accounts we can target for password spraying.|
|`AD Joined Computers`|Key Computers include Domain Controllers, file servers, SQL servers, web servers, Exchange mail servers, database servers, etc.|
|`Key Services`|Kerberos, NetBIOS, LDAP, DNS|
|`Vulnerable Hosts and Services`|Anything that can be a quick win. ( a.k.a an easy host to exploit and gain a foothold)|

### 🧠 Overall Mental Model

```text
┌─────────────────────────────────────────────────────┐
│              INITIAL DOMAIN ENUMERATION             │
└─────────────────────────┬───────────────────────────┘
                          │
                          ▼
                 👂 PASSIVE ENUMERATION
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
          ARP           MDNS        LLMNR/NBT-NS
        Wireshark      tcpdump       Responder
            │             │             │
            └─────────────┼─────────────┘
                          ▼
                  🖥️ HOST LIST
                          │
                          ▼
                  🔎 ACTIVE ENUMERATION
                          │
                    fping / Nmap
                          │
                          ▼
              🌐 HOSTS + SERVICES
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
          DNS          Kerberos        LDAP
          SMB          RDP             MSSQL
                          │
                          ▼
                  👤 USER ENUMERATION
                          │
                       Kerbrute
                          │
                          ▼
                 VALID AD USERS
                          │
                          ▼
              🎯 PASSWORD SPRAYING /
              OTHER FOOTHOLD METHODS
```

---

# 👂 5. Passive Host Identification

## 5.1 Wireshark

**Wireshark** allows us to passively observe network traffic.

This is especially useful when we don't yet know what exists on the network.

Common interesting traffic includes:

- **ARP**
    
- **MDNS**
    
- LLMNR
    
- NBT-NS
    
- DNS
    
- SMB
    
- Other broadcast/multicast traffic
    

Because we are connected to a **switched network**, we generally see traffic associated with our current **broadcast domain**, rather than every packet traversing the entire network.

### Starting Wireshark

```bash
sudo -E wireshark
```

Example:

```text
┌─[htb-student@ea-attack01]─[~]
└──╼ $sudo -E wireshark

11:28:20.487     Main Warn QStandardPaths: runtime directory '/run/user/1001' is not owned by UID 0, but a directory permissions 0700 owned by UID 1001 GID 1002
<SNIP>
```

### 🔎 ARP Enumeration

ARP traffic revealed:

```text
172.16.5.5
172.16.5.25
172.16.5.50
172.16.5.100
172.16.5.125
```

ARP is useful because hosts need ARP to resolve:

```text
IP address → MAC address
```

Seeing ARP requests/replies can therefore reveal hosts that are active on the local network.

### 🔎 MDNS Enumeration

MDNS traffic revealed:

```text
ACADEMY-EA-WEB01.local
```

Example:

```text
172.16.5.125 → 224.0.0.251
Protocol: MDNS
Query: AAAA ACADEMY-EA-WEB01.local
```

> [!tip] 💡 Passive Recon Principle  
> Don't immediately start scanning the entire network.
> 
> **Listen first.**
> 
> Network traffic can reveal:
> 
> - IP addresses
>     
> - Hostnames
>     
> - Naming conventions
>     
> - Services
>     
> - Domain-related information
>     
> 
> Then use those discoveries to guide active enumeration.

---

# 🐢 6. Tcpdump — Passive Enumeration Without a GUI

If the attack host does not have a graphical interface, use **tcpdump**.

```bash
sudo tcpdump -i ens224
```

Here:

```text
-i ens224
   │
   └── Capture traffic on interface ens224
```

### 💾 Save Traffic for Later

A major advantage of packet capture is that the traffic can be saved to a **PCAP** file and analyzed later in Wireshark.

```text
Attack Host
    │
    │ tcpdump
    ▼
capture.pcap
    │
    │ transfer
    ▼
Analysis Host
    │
    ▼
Wireshark
```

> [!important] 📁 Always Save Your Captures  
> PCAP files can reveal information that wasn't immediately obvious during the initial review.
> 
> They are also useful as evidence when documenting the engagement.

### Other Tools

- **Wireshark** → GUI packet analysis
    
- **tcpdump** → command-line packet capture
    
- **net-creds** → extract credentials/interesting information from network traffic
    
- **NetMiner** → network analysis
    
- **pktmon.exe** → Windows packet monitoring utility
    

---

# 🧪 7. Responder — Passive Network Analysis

[**Responder**](https://github.com/lgandx/Responder-Windows) can listen for:

- LLMNR
    
- NBT-NS
    
- MDNS
    

Responder can also **poison** these protocols, but at this stage we only want passive analysis.

### 🔇 Analyze Mode

```bash
sudo responder -I ens224 -A
```

Breakdown:

```text
-I ens224
   │
   └── Listen on ens224

-A
   │
   └── Analyze mode
       Passive listening
       No poisoning
```

### 🧠 Why Use Analyze Mode?

```text
Responder normally
       │
       ├── Listen
       └── Poison requests
              │
              ▼
        Possible hashes
```

But during initial passive enumeration:

```text
Responder -A
       │
       ▼
   Listen only
       │
       ▼
 Observe network requests
       │
       ▼
 Discover hosts / names
```

This lets us build a target list without actively poisoning requests.

> [!warning] ⚠️ Don't Confuse Passive Analysis With Poisoning  
> `-A` puts Responder into **Analyze mode**. The purpose here is observation, not credential interception.

---

# 🔎 8. Passive → Active Enumeration

At this point we have discovered hosts using:

```text
Wireshark
   +
tcpdump
   +
Responder
   │
   ▼
Initial Host List
```

Now we validate and expand that list with **active scanning**.

---

# 📡 9. FPing — ICMP Host Discovery

**fping** is useful for quickly checking many IP addresses.

Unlike traditional `ping`, it is designed to efficiently send ICMP requests to multiple targets.

### Command

```bash
fping -asgq 172.16.5.0/23
```

### Important Options

|Option|Meaning|
|---|---|
|`-a`|Show targets that are alive|
|`-s`|Print statistics at the end|
|`-g`|Generate target list from CIDR|
|`-q`|Quiet mode; don't show per-target results|

### Example Output

```text
172.16.5.5
172.16.5.25
172.16.5.50
172.16.5.100
172.16.5.125
172.16.5.200
172.16.5.225
172.16.5.238
172.16.5.240

     510 targets
       9 alive
     501 unreachable
       0 unknown addresses

    2004 timeouts (waiting for response)
    2013 ICMP Echos sent
       9 ICMP Echo Replies received
    2004 other ICMP received

 0.029 ms (min round trip time)
 0.396 ms (avg round trip time)
 0.799 ms (max round trip time)
       15.366 sec (elapsed real time)
```

### 🧠 What We Learned

The `/23` network contains:

```text
172.16.5.0/23
      │
      ▼
510 usable target addresses in this lab
      │
      ▼
9 responded to ICMP
```

The important result is the **live-host list**.

> [!warning] ⚠️ ICMP Is Not Definitive  
> A host that doesn't respond to ICMP is **not necessarily offline**.
> 
> Firewalls may block ICMP while allowing:
> 
> ```text
> TCP 445
> TCP 3389
> TCP 80
> TCP 443
> TCP 88
> ```
> 
> Therefore:
> 
> **fping = initial discovery, not complete host discovery.**

---

# 🗺️ 10. Build the Target List

Combine:

```text
Passive discoveries
      +
fping results
      │
      ▼
hosts.txt
```

Example:

```text
172.16.5.5
172.16.5.25
172.16.5.50
172.16.5.100
172.16.5.125
172.16.5.200
172.16.5.225
172.16.5.238
172.16.5.240
```

This file can then be fed directly into Nmap.

---

# 🔬 11. Nmap — Active Host & Service Enumeration

Now we want to determine:

- What services are running?
    
- Which hosts are Windows?
    
- Which host is the Domain Controller?
    
- Which systems expose SMB?
    
- Which systems expose LDAP?
    
- Which systems expose Kerberos?
    
- Which systems expose DNS?
    
- Which systems expose RDP?
    
- Are there legacy/vulnerable systems?
    
- Which hosts should we investigate further?
    

### Command

```bash
sudo nmap -v -A -iL hosts.txt -oN /home/htb-student/Documents/host-enum
```

### Breakdown

```text
-v
│
└── Verbose output

-A
│
└── Aggressive scan functionality

-iL hosts.txt
│
└── Read targets from hosts.txt

-oN /home/htb-student/Documents/host-enum
│
└── Save normal-format output
```

> [!important] 💾 Save Your Nmap Results  
> A best practice is to use:
> 
> ```text
> -oA
> ```
> 
> This saves results in multiple formats that can later be:
> 
> - Reviewed
>     
> - Parsed
>     
> - Fed into other tools
>     
> - Included in reports
>     
> - Revisited during later enumeration
>     

---

# 🏰 12. Recognizing a Domain Controller From Nmap

One of the most important skills in AD enumeration is learning to recognize a **Domain Controller from its service fingerprint**.

The scan found:

```text
Nmap scan report for inlanefreight.local (172.16.5.5)

PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http
636/tcp  open  ssl/ldap
3268/tcp open  ldap
3269/tcp open  ssl/ldap
3389/tcp open  ms-wbt-server
```

### 🔑 Important AD Ports

|Port|Service|Why It Matters|
|--:|---|---|
|`53`|DNS|Domain DNS infrastructure|
|`88`|Kerberos|AD authentication|
|`135`|MSRPC|Windows RPC|
|`139`|NetBIOS|Legacy Windows networking|
|`389`|LDAP|Directory queries|
|`445`|SMB|File sharing / Windows networking|
|`464`|Kerberos password change|Kerberos password operations|
|`636`|LDAPS|LDAP over TLS|
|`3268`|Global Catalog|Forest-wide directory searches|
|`3269`|Global Catalog over SSL|Secure Global Catalog|
|`3389`|RDP|Remote Windows access|

### 🧠 AD Mental Model

```text
                 DOMAIN CONTROLLER
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
     DNS             Kerberos          LDAP
      53                88              389
       │                │                │
       │                │                └── Directory
       │                └── Authentication
       └── Name resolution

                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
             SMB       RPC       RDP
             445       135       3389
```

### 🏷️ Nmap Also Revealed

```text
NetBIOS Domain:
INLANEFREIGHT

DNS Domain:
INLANEFREIGHT.LOCAL

Computer:
ACADEMY-EA-DC01

FQDN:
ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL

DNS Tree:
INLANEFREIGHT.LOCAL
```

The certificate information also exposed:

```text
DNS:
ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL

Issuer:
INLANEFREIGHT-CA
```

> [!success] 🎯 Major Enumeration Win  
> We have identified the **Domain Controller**:
> 
> ```text
> ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
> 172.16.5.5
> ```
> 
> This becomes one of the most important systems for subsequent AD enumeration.

---

# 🧓 13. Identifying Legacy Hosts

Another host produced:

```text
172.16.5.100
```

with:

```text
80/tcp    open  http    Microsoft IIS httpd 7.5
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
443/tcp   open  https
445/tcp   open  microsoft-ds
1433/tcp  open  ms-sql-s
```

Nmap identified:

```text
Microsoft IIS 7.5
Microsoft SQL Server 2008 R2
Windows Server 2008 R2 Standard
```

And:

```text
Product_Version: 6.1.7600
```

### 🔥 Why This Is Interesting

An old operating system can indicate the presence of **legacy infrastructure**.

Potential historical vulnerabilities include:

```text
EternalBlue
MS08-067
BlueKeep
```

This doesn't mean the host is automatically vulnerable. It means:

```text
Legacy OS
   │
   ▼
Older software/components may exist
   │
   ▼
Investigate patch level + exposed services
   │
   ▼
Potential attack path
```

### 🏭 Why Legacy Systems Exist

Large organizations may have systems tied to:

- Production equipment
    
- HVAC
    
- Industrial systems
    
- Specialized applications
    
- Legacy databases
    
- Hardware that cannot easily be upgraded
    

Replacing these systems can be expensive or disruptive.

> [!warning] 🚨 Never Exploit Legacy Systems Blindly  
> Exploiting an old system can cause:
> 
> - System instability
>     
> - Service crashes
>     
> - Production outages
>     
> - Equipment disruption
>     
> - Loss of customer data
>     
> 
> Before actively exploiting a potentially fragile legacy system, obtain **explicit authorization in writing**.

---

# 🧠 14. Don't Just Look for Vulnerabilities — Look for the Path to the Domain

At this stage, the objective isn't simply:

```text
"Find vulnerable machines."
```

The larger objective is:

```text
Find a way to obtain:
    │
    ├── 👤 Domain credentials
    ├── 🔑 NTLM hash
    ├── 💻 Domain-user shell
    └── ⚙️ SYSTEM on a domain-joined host
```

Once obtained, the assessment changes dramatically.

```text
Unauthenticated
      │
      ▼
Network Enumeration
      │
      ▼
Hosts + Services
      │
      ▼
User Enumeration
      │
      ▼
Valid Domain User
      │
      ▼
Credentialed Enumeration
      │
      ▼
AD Attack Paths
```

---

# 👤 15. Identifying Active Directory Users

A valid domain user is extremely valuable.

Potential starting points include:

```text
Cleartext credentials
        │
        ├── Password
        └── NTLM hash

OR

SYSTEM access
        │
        ▼
Domain enumeration
```

With a valid domain account, we can perform significantly deeper enumeration.

The immediate goal is therefore:

> **Build a list of valid domain usernames.**

---

# ⚔️ 16. Kerbrute — Internal AD Username Enumeration

[**Kerbrute**](https://github.com/ropnop/kerbrute) can enumerate valid AD usernames by interacting with **Kerberos**.

It can be particularly useful from an unauthenticated starting point.

### 🧠 Why Kerberos Helps

The basic idea:

```text
Username candidate
       │
       ▼
Send Kerberos request
       │
       ▼
Does the KDC recognize the account?
       │
       ├───────────────┐
       ▼               ▼
     Valid           Invalid
       │
       ▼
Save username
```

The KDC is the **Key Distribution Center**, normally running on the Domain Controller.

### 📚 Useful Username Wordlists

The source uses username lists from **Insidetrust**, particularly:

```text
jsmith.txt
jsmith2.txt
```

These lists contain statistically likely username formats.

This is useful because organizations commonly follow predictable naming schemes:

```text
john.smith
jsmith
johns
j.smith
```

---

# 🛠️ 17. Installing / Compiling Kerbrute

### Clone the Repository

```bash
sudo git clone https://github.com/ropnop/kerbrute.git
```

### View Build Options

```bash
make help
```

Output:

```text
help:            Show this help.
windows:  Make Windows x86 and x64 Binaries
linux:  Make Linux x86 and x64 Binaries
mac:  Make Darwin (Mac) x86 and x64 Binaries
clean:  Delete any binaries
all:  Make Windows, Linux and Mac x86/x64 Binaries
```

### Compile All Platforms

```bash
sudo make all
```

This creates binaries for:

```text
Windows x86
Windows x64
Linux x86
Linux x64
macOS x86
macOS x64
```

### Compiled Binaries

```bash
ls dist/
```

Example:

```text
kerbrute_darwin_amd64
kerbrute_linux_386
kerbrute_linux_amd64
kerbrute_windows_386.exe
kerbrute_windows_amd64.exe
```

---

# 🧪 18. Test Kerbrute

Run the Linux x64 binary:

```bash
./kerbrute_linux_amd64
```

The tool displays:

```text
Version: dev (9cfb81e) - 02/17/22 - Ronnie Flathers @ropnop

This tool is designed to assist in quickly bruteforcing valid Active Directory accounts through Kerberos Pre-Authentication.
It is designed to be used on an internal Windows domain with access to one of the Domain Controllers.
Warning: failed Kerberos Pre-Auth counts as a failed login and WILL lock out accounts
```

> [!warning] 🔐 Account Lockout Risk  
> Username enumeration and authentication-related attacks must be considered in relation to the organization's **account lockout policy**.
> 
> Failed authentication attempts can contribute to account lockouts.

---

# 📍 19. Put Kerbrute in `$PATH`

Check the current path:

```bash
echo $PATH
```

Move the binary:

```bash
sudo mv kerbrute_linux_amd64 /usr/local/bin/kerbrute
```

Now:

```bash
kerbrute
```

works from any directory.

---

# 👥 20. Enumerate Valid AD Users

Once we know:

```text
Domain:
INLANEFREIGHT.LOCAL

Domain Controller:
172.16.5.5

Username wordlist:
jsmith.txt
```

Run:

```bash
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt -o valid_ad_users
```

### 🧩 Command Breakdown

```text
kerbrute
│
└── Tool

userenum
│
└── Enumerate usernames

-d INLANEFREIGHT.LOCAL
│
└── Target AD domain

--dc 172.16.5.5
│
└── Use this Domain Controller

jsmith.txt
│
└── Username candidates

-o valid_ad_users
│
└── Save valid results
```

### Example Results

```text
2021/11/17 23:01:46 >  Using KDC(s):
2021/11/17 23:01:46 >   172.16.5.5:88
2021/11/17 23:01:46 >  [+] VALID USERNAME:       jjones@INLANEFREIGHT.LOCAL
2021/11/17 23:01:46 >  [+] VALID USERNAME:       sbrown@INLANEFREIGHT.LOCAL
2021/11/17 23:01:46 >  [+] VALID USERNAME:       tjohnson@INLANEFREIGHT.LOCAL
2021/11/17 23:01:50 >  [+] VALID USERNAME:       evalentin@INLANEFREIGHT.LOCAL

<SNIP>

2021/11/17 23:01:51 >  [+] VALID USERNAME:       sgage@INLANEFREIGHT.LOCAL
2021/11/17 23:01:51 >  [+] VALID USERNAME:       jshay@INLANEFREIGHT.LOCAL
2021/11/17 23:01:51 >  [+] VALID USERNAME:       jhermann@INLANEFREIGHT.LOCAL
2021/11/17 23:01:51 >  [+] VALID USERNAME:       whouse@INLANEFREIGHT.LOCAL
2021/11/17 23:01:51 >  [+] VALID USERNAME:       emercer@INLANEFREIGHT.LOCAL
2021/11/17 23:01:52 >  [+] VALID USERNAME:       wshepherd@INLANEFREIGHT.LOCAL
2021/11/17 23:01:56 >  Done! Tested 48705 usernames (56 valid) in 9.940 seconds
```

### 🎯 Result

```text
48,705 usernames tested
        │
        ▼
56 valid AD users
        │
        ▼
valid_ad_users
        │
        ▼
Potential password-spraying targets
```

> [!success] 🚀 Why This Is Valuable  
> We have transformed an enormous list of **possible usernames** into a smaller list of **confirmed domain accounts**.
> 
> This is much more useful for later authentication testing.

---
# 🛠️ 20.5. Kerbrute — GLIBC Compatibility Error Fix

> [!warning] ⚠️ Error: `GLIBC` Version Mismatch  
> If the precompiled Kerbrute binary fails to execute with an error similar to:
> 
> ```text
> GLIBC_2.34 not found
> ```
> 
> or:
> 
> ```text
> GLIBC_2.32 not found
> ```
> 
> the binary was compiled against a **newer version of GNU C Library (GLIBC)** than the version available on the current Linux system.
> 
> **Do not immediately upgrade GLIBC manually.** Replacing/upgrading the system C library can break other applications and potentially the entire OS.

## 🔧 Use an Older Kerbrute Release

Instead of using the incompatible binary, download **Kerbrute v1.0.3**, which can work on older environments.

### 1️⃣ Remove the Existing Binary

```bash
rm -f kerbrute
```

### 2️⃣ Download Kerbrute v1.0.3

```bash
wget https://github.com/ropnop/kerbrute/releases/download/v1.0.3/kerbrute_linux_amd64 -O kerbrute
```

### 3️⃣ Make It Executable

```bash
chmod +x kerbrute
```

### 4️⃣ Test It

```bash
./kerbrute
```

If the binary starts and displays the Kerbrute help/usage information, the compatibility problem is resolved.

### 🧠 Mental Model

```text
Kerbrute binary
      │
      ▼
Compiled against GLIBC 2.34+
      │
      ▼
Your Parrot OS has older GLIBC
      │
      ▼
❌ Binary won't execute
      │
      ▼
Download older Kerbrute release
      │
      ▼
v1.0.3
      │
      ▼
chmod +x
      │
      ▼
./kerbrute
      │
      ▼
✅ Compatible binary
```

> [!tip] 💡 Alternative  
> If an older release still doesn't work, compiling Kerbrute directly on the target Linux environment can avoid many precompiled-binary compatibility problems:
> 
> ```bash
> git clone https://github.com/ropnop/kerbrute.git
> cd kerbrute
> make linux
> ```
> 
> The key idea is **match the binary to the environment**, rather than modifying the system's GLIBC just to run one tool.
# ⚙️ 21. SYSTEM Access as an Alternative Foothold

Another major objective is obtaining:

```text
NT AUTHORITY\SYSTEM
```

on a **domain-joined Windows host**.

### 🧠 Why SYSTEM Is So Valuable

`SYSTEM` is a built-in Windows account with extremely high local privileges.

On a domain-joined machine:

```text
SYSTEM
  │
  ▼
Computer Account
  │
  ▼
Domain Authentication
  │
  ▼
Active Directory Enumeration
```

A computer account is itself an AD security principal.

Therefore:

> **SYSTEM on a domain-joined host can provide capabilities similar to having a low-privileged domain account for many enumeration tasks.**

---

# 💥 22. Potential Ways to Obtain SYSTEM

Possible avenues include:

### Remote Windows Exploitation

```text
MS08-067
EternalBlue
BlueKeep
```

### Service Abuse

Abuse a service running as:

```text
NT AUTHORITY\SYSTEM
```

or exploit privileges such as:

```text
SeImpersonatePrivilege
```

One historical tool mentioned is:

```text
Juicy Potato
```

### Local Privilege Escalation

Exploit local Windows privilege escalation vulnerabilities.

### Local Administrator → PsExec

If local administrator access is obtained on a domain-joined machine:

```text
Local Administrator
        │
        ▼
PsExec
        │
        ▼
SYSTEM shell
```

---

# 🩸 23. What SYSTEM Can Enable

Once SYSTEM is obtained on a domain-joined host, several AD attack paths become available.

```text
                 SYSTEM
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
     AD Enum     Credential    Network
                  Attacks       Attacks
        │           │            │
        ▼           ▼            ▼
   BloodHound   Kerberoasting   Inveigh
   PowerView    ASREPRoasting   SMB Relay
                    │
                    ▼
             Token Impersonation
                    │
                    ▼
               ACL Attacks
```

Potential activities include:

- Enumerating the domain with built-in tools
    
- BloodHound
    
- PowerView
    
- Kerberoasting
    
- ASREPRoasting
    
- Inveigh
    
- Net-NTLMv2 capture
    
- SMB relay
    
- Token impersonation
    
- ACL attacks
    

---

# 🔄 24. The Progressive Enumeration Methodology

The important lesson is **not to throw every tool at the network immediately**.

Use progressive stages:

```text
1️⃣ PASSIVE
   │
   ├── Wireshark
   ├── tcpdump
   └── Responder -A
   │
   ▼
2️⃣ HOST DISCOVERY
   │
   └── fping
   │
   ▼
3️⃣ SERVICE ENUMERATION
   │
   └── Nmap
   │
   ▼
4️⃣ IDENTIFY CRITICAL HOSTS
   │
   ├── DC
   ├── DNS
   ├── SMB
   ├── LDAP
   ├── Kerberos
   ├── SQL
   └── Web
   │
   ▼
5️⃣ USER ENUMERATION
   │
   └── Kerbrute
   │
   ▼
6️⃣ FOOTHOLD
   │
   ├── Valid credentials
   ├── NTLM hash
   ├── SYSTEM
   └── Vulnerable service
   │
   ▼
7️⃣ CREDENTIAL/DOMAIN ENUMERATION
```

### 🧠 The Core Mental Model

> [!important] 🧠 Don't Enumerate Randomly
> 
> Think:
> 
> **LISTEN → DISCOVER → VALIDATE → IDENTIFY → ENUMERATE → REGROUP → ATTACK**
> 
> Each phase gives information that improves the next phase.

---

# 🔁 25. Enumeration Is an Iterative Process

You should expect to return to previous information repeatedly.

For example:

```text
Passive Recon
     │
     ▼
Find Host
     │
     ▼
Nmap
     │
     ▼
Find DNS
     │
     ▼
DNS Enumeration
     │
     ▼
Find Hostname
     │
     ▼
Nmap Host Again
     │
     ▼
Find LDAP/Kerberos
     │
     ▼
User Enumeration
     │
     ▼
Find Credentials
     │
     ▼
Credentialed Enumeration
     │
     ▼
Discover More Hosts
     │
     └───────────────┐
                     ▼
               Enumerate Again
```

> [!tip] 🔄 CPTS Mindset  
> Enumeration is not:
> 
> ```text
> Run Nmap → Done
> ```
> 
> It is:
> 
> ```text
> Discover → Learn → Re-evaluate → Enumerate → Discover more
> ```
> 
> Every new piece of information can change where you look next.

---

# 🔇 26. Noise and Stealth Considerations

The correct scanning methodology depends on the engagement.

### Non-Evasive

If the customer explicitly requested:

```text
Non-evasive
```

then noisy scanning may be acceptable.

### Evasive / Red Team

For a stealth-focused engagement:

```text
Nmap entire network
        │
        ▼
Huge amount of traffic
        │
        ▼
IDS / IPS / SOC detection
```

Therefore:

```text
Evasive engagement
      │
      ▼
Minimize unnecessary traffic
      │
      ▼
Targeted enumeration
      │
      ▼
Avoid obvious scanning patterns
```

> [!warning] 🛡️ Know What Your Tools Actually Do  
> Some Nmap scripts perform more than simple service discovery.
> 
> Vulnerability checks can potentially:
> 
> - Crash services
>     
> - Overload devices
>     
> - Trigger IDS/IPS
>     
> - Disrupt fragile equipment
>     
> 
> This becomes especially important around:
> 
> - PLCs
>     
> - Industrial controllers
>     
> - Sensors
>     
> - Embedded devices
>     
> - Legacy systems
>     

---

# 📝 27. Evidence & Note-Taking

Every phase should produce evidence that can be reused later.

Save:

```text
📁 Nmap results
📁 PCAP files
📁 Responder output
📁 Host lists
📁 Username lists
📁 Screenshots
📁 Tool output
📁 Interesting findings
```

### Example Workflow

```text
hosts.txt
   │
   └── Nmap
        │
        └── host-enum
             │
             ├── DC identified
             ├── SQL identified
             ├── Web server identified
             └── Legacy host identified

valid_ad_users
   │
   └── Later authentication testing
```

> [!important] 📌 Documentation Is Part of Enumeration  
> Don't rely on memory.
> 
> **Save the output when you find it.**
> 
> Information discovered early in the assessment may become critical several hours later.

---

# ⚡ 28. Initial Enumeration Cheat Sheet

### 👂 Passive Discovery

```bash
sudo -E wireshark
```

```bash
sudo tcpdump -i ens224
```

```bash
sudo responder -I ens224 -A
```

### 📡 Active Host Discovery

```bash
fping -asgq 172.16.5.0/23
```

### 🔬 Nmap Host Enumeration

```bash
sudo nmap -v -A -iL hosts.txt -oN /home/htb-student/Documents/host-enum
```

Best-practice output:

```bash
-oA
```

### 🛠️ Kerbrute Setup

```bash
sudo git clone https://github.com/ropnop/kerbrute.git
```

```bash
make help
```

```bash
sudo make all
```

```bash
ls dist/
```

```bash
./kerbrute_linux_amd64
```

```bash
echo $PATH
```

```bash
sudo mv kerbrute_linux_amd64 /usr/local/bin/kerbrute
```

### 👤 AD Username Enumeration

```bash
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt -o valid_ad_users
```

---

# 🧩 29. CPTS Decision Flow

```text
START
  │
  ▼
📋 Read Scope
  │
  ├── What IPs?
  ├── What hosts?
  ├── What techniques?
  └── Evasive or non-evasive?
  │
  ▼
👂 PASSIVE
  │
  ├── Wireshark
  ├── tcpdump
  └── Responder -A
  │
  ▼
📡 ACTIVE HOST DISCOVERY
  │
  └── fping
  │
  ▼
🔬 NMAP
  │
  ├── Identify OS
  ├── Identify services
  ├── Identify DC
  ├── Identify SMB
  ├── Identify LDAP
  ├── Identify Kerberos
  ├── Identify DNS
  └── Identify legacy systems
  │
  ▼
👤 USER ENUMERATION
  │
  └── Kerbrute
  │
  ▼
🎯 VALID USERS
  │
  ├── Password spraying
  ├── Further enumeration
  └── Other credential attacks
  │
  ▼
⚙️ OR FIND SYSTEM
  │
  ├── Remote exploit
  ├── Service abuse
  ├── Local privilege escalation
  └── Local admin → SYSTEM
  │
  ▼
🏰 DOMAIN ENUMERATION
  │
  ├── BloodHound
  ├── PowerView
  ├── Kerberoasting
  ├── ASREPRoasting
  ├── Credential capture
  ├── SMB relay
  ├── Token impersonation
  └── ACL attacks
```

# 🧠 30. Key Takeaways

> [!success] 🎯 CPTS Mental Model  
> **1. Scope first** — know exactly what you are allowed to test.
> 
> **2. Listen before scanning** — ARP, MDNS, LLMNR/NBT-NS and other traffic can reveal hosts and names.
> 
> **3. Validate discoveries** — use `fping` and Nmap to actively confirm what exists.
> 
> **4. Read Nmap intelligently** — don't just count open ports; recognize AD patterns such as:
> 
> ```text
> 53 + 88 + 389 + 445 + 3268
> ```
> 
> which strongly suggests a **Domain Controller**.
> 
> **5. Look for legacy systems** — old Windows/SQL/IIS versions can represent valuable attack paths, but obtain approval before exploitation.
> 
> **6. Find users** — Kerbrute can turn username candidates into validated AD accounts.
> 
> **7. Save everything** — scan results, PCAPs, usernames, screenshots, and tool output are valuable later.
> 
> **8. Think progressively**:
> 
> ```text
> PASSIVE
>    ↓
> HOST DISCOVERY
>    ↓
> SERVICE ENUMERATION
>    ↓
> DOMAIN CONTROLLER
>    ↓
> USER ENUMERATION
>    ↓
> CREDENTIALS / SYSTEM
>    ↓
> CREDENTIALED AD ENUMERATION
> ```
> 
> **9. Regroup constantly** — every discovery can reveal a better path to the next stage.
> 
> **10. The ultimate early objective is a foothold** — valid domain credentials, an NTLM hash, a domain-user shell, or SYSTEM on a domain-joined host.