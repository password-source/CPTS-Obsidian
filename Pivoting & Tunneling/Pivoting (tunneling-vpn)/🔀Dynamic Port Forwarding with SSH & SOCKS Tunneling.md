# 🔀 1. Dynamic Port Forwarding with SSH & SOCKS Tunneling

> [!info]  
> **Core objective:** Use a compromised Linux host as a **pivot** to reach services and hosts inside an internal network that the attack host cannot directly route to.
> 
> The key progression is:
> 
> **Compromise Pivot → Discover Internal Network → Establish SSH Tunnel → Route Traffic Through Pivot → Enumerate Internal Hosts → Attack Internal Services**

---

# 🧠 2. Port Forwarding in Context

**Port forwarding** redirects communication from one port to another.

The important pentesting idea is that the forwarding host can act as a **bridge** between two networks.

```text
ATTACK HOST
10.10.15.x
     |
     | SSH
     v
COMPROMISED UBUNTU
10.129.202.64
172.16.5.129
     |
     | Internal network
     v
172.16.5.0/23
     |
     +---- Windows
     +---- Web servers
     +---- Databases
     +---- Other internal systems
```

The attack host cannot directly route to `172.16.5.0/23`.

The compromised Ubuntu server can.

Therefore:

> [!important]  
> The compromised machine becomes a **pivot point**. We use it to make the internal network accessible from our attack host.

Port forwarding commonly uses TCP as the communication layer, while protocols such as SSH and SOCKS can encapsulate the forwarded traffic.

This can allow us to bypass network restrictions and use services available through the compromised host to reach otherwise inaccessible systems.

---

# 🔗 3. SSH Local Port Forwarding

## 🎯 The Situation

We have compromised an Ubuntu server:

```text
Attack Host
10.10.15.x
     |
     | SSH :22
     v
Ubuntu / Pivot
10.129.202.64
     |
     | localhost:3306
     v
MySQL
```

We scan the Ubuntu server:

```bash
nmap -sT -p22,3306 10.129.202.64
```

Example result:

```text
PORT     STATE  SERVICE
22/tcp   open   ssh
3306/tcp closed mysql
```

### 🧠 What does this tell us?

`3306` appears **closed from the attack host**.

But this does **not necessarily mean MySQL is unavailable on the Ubuntu machine**.

The module explains that MySQL is hosted locally on the Ubuntu server.

Therefore, instead of connecting directly:

```text
Attack Host ─────X────> Ubuntu:3306
```

we create:

```text
Attack Host:1234
       |
       | SSH tunnel
       v
Ubuntu:3306
       |
       v
MySQL
```

---

# 🔧 4. Creating an SSH Local Port Forward

Command:

```bash
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64
```

### 🔍 Break it down

```text
ssh
│
├── -L
│    └── Local port forwarding
│
├── 1234
│    └── Port opened on ATTACK HOST
│
├── localhost:3306
│    └── Destination from the SSH SERVER's perspective
│
└── ubuntu@10.129.202.64
     └── SSH connection to the pivot
```

The critical mental model is:

> **The first port is local to us. The destination is reached from the SSH server.**

```text
ATTACK HOST
127.0.0.1:1234
      |
      | SSH tunnel
      |
      v
UBUNTU
127.0.0.1:3306
      |
      v
   MySQL
```

So when we connect to:

```text
127.0.0.1:1234
```

our traffic is carried through SSH and ultimately reaches:

```text
127.0.0.1:3306
```

on the Ubuntu server.

---

# 🧪 5. Verifying the Local Port Forward

## Using Netstat

```bash
netstat -antp | grep 1234
```

Example:

```text
tcp        0      0 127.0.0.1:1234    0.0.0.0:*    LISTEN    4034/ssh
tcp6       0      0 ::1:1234          :::*         LISTEN    4034/ssh
```

The important part is:

```text
127.0.0.1:1234 ... LISTEN ... ssh
```

This tells us:

- Port `1234` is listening locally.
    
- The SSH process owns the listener.
    
- The local forwarding tunnel exists.
    

---

## Using Nmap

```bash
nmap -v -sV -p1234 localhost
```

Result:

```text
PORT     STATE SERVICE VERSION
1234/tcp open  mysql   MySQL 8.0.28-0ubuntu0.20.04.3
```

This is especially useful because Nmap is now seeing:

```text
localhost:1234 → MySQL
```

even though the real service is:

```text
Ubuntu localhost:3306
```

> [!success]  
> **Port forwarding changes the perspective of the service.**
> 
> We don't need to make MySQL externally accessible. We make it appear locally accessible through the SSH tunnel.

---

# 🔀 6. Forwarding Multiple Ports

We can create multiple local forwards using multiple `-L` arguments.

```bash
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64
```

This creates:

```text
ATTACK HOST                  UBUNTU
127.0.0.1:1234 ────────────> localhost:3306
127.0.0.1:8080 ────────────> localhost:80
```

So:

|Attack Host Port|Pivot Destination|Purpose|
|---|---|---|
|`1234`|`localhost:3306`|MySQL|
|`8080`|`localhost:80`|Apache/Web server|

### 🧠 Methodology

Use **local forwarding** when:

> **You know exactly which service you want to reach.**

For example:

```text
"I know MySQL is listening on the pivot's localhost:3306."
```

Then:

```text
-L 1234:localhost:3306
```

is appropriate.

---

# 🌐 7. Recognizing When Local Forwarding Is Not Enough

Now consider a different situation.

We discover that the compromised Ubuntu server has:

```text
ens192 → 10.129.202.64
ens224 → 172.16.5.129
```

This is extremely important.

```text
                COMPROMISED UBUNTU
               ┌───────────────────┐
Attack Network │ ens192            │ ens224 │ Internal
10.129.x.x ────┤ 10.129.202.64     │ 172.16.5.129
               └───────────────────┘
                         |
                         v
                   172.16.5.0/23
```

The pivot has access to **two networks**.

The attack host only has access to the first one.

> [!important]  
> Multiple network interfaces on a compromised machine are a major **pivoting indicator**.

---

# 🔎 8. Looking for Pivoting Opportunities

On the compromised Ubuntu host:

```bash
ifconfig
```

Relevant output:

```text
ens192:
    inet 10.129.202.64
    netmask 255.255.0.0

ens224:
    inet 172.16.5.129
    netmask 255.255.254.0

lo:
    inet 127.0.0.1
```

The subnet mask:

```text
255.255.254.0
```

corresponds to:

```text
/23
```

Therefore:

```text
172.16.5.0/23
```

is the internal network.

### 🧠 What should you think when you see this?

Don't immediately attack the current machine again.

Ask:

> **"What other network can this machine see that I cannot?"**

That question leads directly to pivoting.

---

# 🧭 9. Why We Need Dynamic Port Forwarding

Suppose we want to enumerate:

```text
172.16.5.1 - 172.16.5.200
```

or:

```text
172.16.5.0/23
```

The attack host cannot directly scan it because it has no route to that network.

```text
ATTACK HOST
10.10.15.5
     |
     | ❌ No route
     X
172.16.5.0/23
```

But the compromised Ubuntu server can reach it:

```text
ATTACK HOST
     |
     | SSH
     v
UBUNTU
172.16.5.129
     |
     | ✅
     v
172.16.5.0/23
```

The problem is that we don't know **which specific ports/services** we need to forward.

We therefore need something more flexible than:

```text
-L local_port:destination:port
```

We need:

> **Dynamic port forwarding.**

---

# 🧦 10. SOCKS Tunneling

SOCKS allows applications to send traffic through a SOCKS proxy.

The important pentesting model is:

```text
Application
    |
    v
SOCKS Proxy
127.0.0.1:9050
    |
    | SSH
    v
Compromised Pivot
    |
    v
Internal Network
172.16.5.0/23
```

Instead of defining:

```text
1234 → 3306
8080 → 80
```

we define a **SOCKS listener** and allow applications to request different destinations through it.

This is why it is called **dynamic** port forwarding.

---

# 🔐 11. SOCKS4 vs SOCKS5

|SOCKS Version|Authentication|UDP Support|
|---|---|---|
|SOCKS4|No|No|
|SOCKS5|Yes|Yes|

> [!info]  
> The module focuses on using SOCKS tunneling with SSH and ProxyChains to pivot into the internal network.

---

# 🚇 12. Creating the Dynamic SSH Tunnel

Command:

```bash
ssh -D 9050 ubuntu@10.129.202.64
```

### 🔍 What does `-D` mean?

```text
-D 9050
```

requests **dynamic port forwarding**.

SSH creates a SOCKS listener on:

```text
127.0.0.1:9050
```

Traffic sent to this SOCKS listener is carried through the SSH connection to the pivot.

Conceptually:

```text
                    SSH
Attack Host ==================> Pivot
     |
     |
127.0.0.1:9050
 SOCKS Listener
     |
     v
 ProxyChains / Application
     |
     v
172.16.5.x
```

---

# ⛓️ 13. ProxyChains

Creating the SOCKS listener isn't enough.

Your tools need to know:

> **"Send your TCP traffic through this SOCKS proxy."**

This is where **ProxyChains** comes in.

ProxyChains can redirect application TCP connections through:

- SOCKS4
    
- SOCKS5
    
- TOR
    
- HTTP/HTTPS proxies
    
- chained proxy servers
    

The workflow becomes:

```text
Nmap
  |
  v
ProxyChains
  |
  v
127.0.0.1:9050
  |
  v
SSH SOCKS Tunnel
  |
  v
Ubuntu Pivot
  |
  v
Internal Network
```

---

# ⚙️ 14. Configuring ProxyChains

Configuration file:

```text
/etc/proxychains.conf
```

The module uses:

```text
socks4  127.0.0.1 9050
```

Check the configuration:

```bash
tail -4 /etc/proxychains.conf
```

Example:

```text
# meanwile
# defaults set to "tor"
socks4  127.0.0.1 9050
```

### 🧠 Why does this matter?

ProxyChains needs to know:

```text
Proxy type = SOCKS4
Proxy IP   = 127.0.0.1
Proxy port = 9050
```

So when we execute:

```bash
proxychains <command>
```

the application's TCP connections are redirected through:

```text
127.0.0.1:9050
```

---

# 🔎 15. Scanning the Internal Network Through SOCKS

Now we can use:

```bash
proxychains nmap -v -sn 172.16.5.1-200
```

Conceptually:

```text
Nmap
  |
  v
ProxyChains
  |
  v
127.0.0.1:9050
  |
  v
SSH
  |
  v
Ubuntu Pivot
  |
  v
172.16.5.0/23
```

ProxyChains output shows connections such as:

```text
|S-chain|-<>-127.0.0.1:9050-<><>-172.16.5.5:80-<><>-OK
```

This demonstrates the traffic being sent through the SOCKS listener.

---

# ⚠️ 16. Important Nmap Limitation with ProxyChains

> [!warning]  
> ProxyChains can only perform a **full TCP connect scan** reliably.

Why?

ProxyChains works by redirecting TCP connections.

It does not understand Nmap's lower-level/partial packet techniques.

Therefore, scans such as:

```bash
-sS
```

are not appropriate through ProxyChains.

Use:

```bash
-sT
```

for a TCP connect scan.

### Recommended pattern

```bash
proxychains nmap -sT <target>
```

---

# 🧠 17. Why `-Pn` Is Important

Host-alive checks can cause problems when pivoting.

Windows Defender Firewall may block ICMP requests, meaning:

```text
ping → Windows target
```

may fail even though the machine is actually online.

Therefore:

```bash
-Pn
```

tells Nmap:

> **Skip host discovery and treat the target as up.**

This is especially useful when scanning Windows hosts through a pivot.

---

# 🖥️ 18. Scanning an Internal Windows Host

Suppose we identify:

```text
172.16.5.19
```

We scan it through ProxyChains:

```bash
proxychains nmap -v -Pn -sT 172.16.5.19
```

Important options:

|Option|Meaning|
|---|---|
|`proxychains`|Force application traffic through configured proxy|
|`nmap`|Network scanner|
|`-v`|Verbose output|
|`-Pn`|Skip host discovery|
|`-sT`|TCP connect scan|

The scan discovers:

```text
445/tcp   open
135/tcp   open
3389/tcp  open
139/tcp   open
```

This gives us a new attack surface:

```text
172.16.5.19
│
├── 135 RPC
├── 139 NetBIOS/SMB
├── 445 SMB
└── 3389 RDP
```

### 🧠 The key methodology point

Pivoting is **not the attack itself**.

Pivoting gives you **visibility and connectivity** to another network.

Once you discover services, you return to your normal service-specific methodology:

```text
Internal Host
     |
     v
Enumerate Ports
     |
     +── 445 → SMB methodology
     |
     +── 3389 → RDP methodology
     |
     +── 135 → RPC methodology
     |
     +── Web → HTTP methodology
     |
     +── Database → Database methodology
```

---

# 🧰 19. Using Metasploit Through ProxyChains

ProxyChains isn't limited to Nmap.

The module demonstrates launching Metasploit through ProxyChains:

```bash
proxychains msfconsole
```

Now Metasploit's traffic is intended to go through the SOCKS proxy.

The workflow becomes:

```text
Metasploit
    |
    v
ProxyChains
    |
    v
SOCKS :9050
    |
    v
SSH
    |
    v
Pivot
    |
    v
Internal Target
```

---

# 🔍 20. Using Metasploit's `rdp_scanner`

Inside Metasploit:

```text
search rdp_scanner
```

The relevant module:

```text
auxiliary/scanner/rdp/rdp_scanner
```

Select it:

```text
use 0
```

Set the target:

```text
set rhosts 172.16.5.19
```

Run it:

```text
run
```

The module reports:

```text
172.16.5.19:3389 - Detected RDP on 172.16.5.19:3389
(name:DC01)
(domain:DC01)
(domain_fqdn:DC01)
(server_fqdn:DC01)
(os_version:10.0.17763)
(Requires NLA: No)
```

### 🧠 Why use this after Nmap?

Nmap told us:

```text
3389/tcp open
```

The RDP scanner gives additional protocol-specific information:

```text
RDP confirmed
Host name
Domain
OS version
NLA requirement
```

So the methodology is:

```text
Generic Enumeration
        |
        v
3389 discovered
        |
        v
Protocol-specific enumeration
        |
        v
RDP information
        |
        v
Decide next attack/access method
```

---

# 🖥️ 21. Accessing the Internal Windows Host Through the Tunnel

If valid credentials are available:

```text
victor
pass@123
```

we can use `xfreerdp`.

Command:

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

Breakdown:

|Component|Meaning|
|---|---|
|`proxychains`|Send traffic through SOCKS|
|`xfreerdp`|Linux RDP client|
|`/v:172.16.5.19`|RDP target|
|`/u:victor`|Username|
|`/p:pass@123`|Password|

Traffic flow:

```text
xfreerdp
    |
    v
ProxyChains
    |
    v
127.0.0.1:9050
    |
    v
SSH Tunnel
    |
    v
Ubuntu Pivot
172.16.5.129
    |
    v
172.16.5.19:3389
    |
    v
Windows RDP
```

The RDP certificate must be accepted before the session is established.

The result is an RDP session to the internal Windows host even though the attack host has no direct route to it.

---

# 🧠 22. The Critical Mental Model

The easiest way to understand this entire technique is to separate **local forwarding** from **dynamic forwarding**.

## 🔵 Local Port Forwarding

You know:

```text
EXACT destination
```

Example:

```text
Pivot localhost:3306
```

Create:

```bash
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64
```

Mental model:

```text
LOCAL:1234
    ↓
SSH
    ↓
PIVOT:3306
```

Use it when you know exactly which service you need.

---

## 🟣 Dynamic Port Forwarding / SOCKS

You don't know exactly which internal service you'll need.

You want flexible access to:

```text
172.16.5.0/23
```

Create:

```bash
ssh -D 9050 ubuntu@10.129.202.64
```

Configure ProxyChains:

```text
socks4 127.0.0.1 9050
```

Then:

```bash
proxychains <tool> <target>
```

Mental model:

```text
              ┌──> 172.16.5.19:3389
              │
Tool → SOCKS ─┼──> 172.16.5.20:445
              │
              └──> 172.16.5.30:80
                    |
                    v
                  Pivot
```

> [!important]  
> **Local forwarding = one known destination.**
> 
> **Dynamic SOCKS forwarding = flexible access to many destinations.**

---

# 📊 23. Local vs Dynamic Port Forwarding

|Feature|Local Port Forwarding|Dynamic Port Forwarding|
|---|---|---|
|SSH option|`-L`|`-D`|
|Destination|Specific|Dynamic|
|Example|`1234 → 3306`|SOCKS `9050`|
|Best for|Known service|Unknown internal network|
|Multiple targets|Requires multiple forwards|Naturally flexible|
|ProxyChains|Not required|Commonly used|
|Pivoting|Yes|Yes|
|Example tool|MySQL client|Nmap / xfreerdp / Metasploit|

---

# 🧭 24. Complete Pivoting Methodology

The most important thing for CPTS is understanding **how you arrive at the technique**, rather than memorizing commands.

```text
                 COMPROMISED HOST
                        |
                        v
              Check network interfaces
                        |
                        v
              ┌─────────────────────┐
              │ Multiple networks?  │
              └──────────┬──────────┘
                         YES
                          |
                          v
                 Identify internal subnet
                          |
                          v
              Can you access a known service?
                    /               \
                  YES               NO
                   |                 |
                   v                 v
              SSH -L             SSH -D
                   |                 |
                   v                 v
          Known destination      SOCKS listener
                                     |
                                     v
                              ProxyChains
                                     |
                                     v
                            Scan internal hosts
                                     |
                                     v
                             Discover services
                                     |
                    ┌────────────────┼────────────────┐
                    v                v                v
                   SMB              RDP              HTTP
                    |                |                |
                    v                v                v
              SMB methodology  RDP methodology  Web methodology
```

---

# 🔄 25. Full Attack-Chain Mental Model

Think of the operation as **two separate problems**:

### Problem 1 — Connectivity

> "I cannot reach the internal network."

Solution:

```text
SSH Pivot
+
SOCKS
+
ProxyChains
```

### Problem 2 — Exploitation

> "Now that I can reach the internal host, what can I do with the services I discovered?"

Solution:

```text
Enumeration
     ↓
Service Identification
     ↓
Credentials / Misconfiguration / Vulnerability
     ↓
Initial Access
     ↓
Privilege Escalation
     ↓
Further Enumeration
     ↓
New Pivot
```

This distinction is extremely important.

> [!tip]  
> **Pivoting solves reachability. It does not automatically solve exploitation.**
> 
> Once you pivot successfully, you start applying the appropriate methodology to the newly accessible hosts.

---

# 🧪 26. Command Cheat Sheet

### Identify the pivot's interfaces

```bash
ifconfig
```

Look for:

```text
Multiple NICs
Different subnets
Internal IP addresses
```

---

### Scan a known service on the pivot

```bash
nmap -sT -p22,3306 10.129.202.64
```

---

### Local SSH port forward

```bash
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64
```

---

### Verify local forwarding

```bash
netstat -antp | grep 1234
```

or:

```bash
nmap -v -sV -p1234 localhost
```

---

### Forward multiple services

```bash
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64
```

---

### Dynamic SSH forwarding

```bash
ssh -D 9050 ubuntu@10.129.202.64
```

---

### Configure ProxyChains

```text
/etc/proxychains.conf
```

Add:

```text
socks4 127.0.0.1 9050
```

---

### Scan internal hosts

```bash
proxychains nmap -v -sn 172.16.5.1-200
```

---

### TCP connect scan through pivot

```bash
proxychains nmap -v -Pn -sT 172.16.5.19
```

---

### Launch Metasploit through ProxyChains

```bash
proxychains msfconsole
```

---

### RDP scanner

```text
search rdp_scanner
use 0
set rhosts 172.16.5.19
run
```

---

### RDP through SOCKS

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

---

# 🧠 27. CPTS Decision Rules

> [!important]  
> Don't memorize this as a collection of unrelated commands. Memorize the **decision logic**.

### 🟢 I compromised a host. What do I check?

```text
ip / ifconfig
        ↓
Network interfaces
        ↓
Multiple subnets?
        ↓
YES
        ↓
Potential pivot
```

### 🟢 I know a specific internal service

Use:

```bash
ssh -L
```

Example:

```bash
ssh -L 1234:localhost:3306 user@pivot
```

### 🟢 I need access to many unknown internal services

Use:

```bash
ssh -D
```

then:

```text
ProxyChains
```

### 🟢 I want to scan an internal Windows host

Think:

```bash
proxychains nmap -Pn -sT <internal-ip>
```

### 🟢 I find a service

Stop thinking about "pivoting" and switch back to your service methodology:

```text
445 → SMB
3389 → RDP
80/443 → Web
22 → SSH
3306 → MySQL
1433 → MSSQL
```

---

# 🧠 28. Final Mental Model

```text
              YOU
               |
               v
       ATTACK / PWNBOX
               |
               | SSH
               v
       COMPROMISED HOST
               |
       ┌───────┴────────┐
       |                |
       v                v
Known service      Internal NIC
       |                |
       v                v
     -L             -D SOCKS
       |                |
       v                v
Specific port      ProxyChains
                        |
                        v
                 Internal Network
                        |
             ┌──────────┼──────────┐
             v          v          v
            SMB        RDP        HTTP
             |          |          |
             v          v          v
          Service-specific attack
                        |
                        v
                 New credentials /
                 access / pivot
```

> [!success]
> 
> ### 🎯 Key Takeaways
> 
> - **Port forwarding** redirects traffic through another host.
>     
> - **SSH `-L`** creates a local forward to one known destination.
>     
> - **SSH `-D`** creates a dynamic SOCKS proxy.
>     
> - **ProxyChains** forces supported application TCP traffic through the SOCKS proxy.
>     
> - A compromised host with **multiple NICs/subnets** is a strong indication that it may be useful as a pivot.
>     
> - `-sT` is the appropriate Nmap scan type for the ProxyChains workflow described here.
>     
> - `-Pn` is useful when ICMP/host discovery cannot reliably determine whether a target is alive.
>     
> - Pivoting gives you **network reachability**; it does not replace service-specific enumeration or exploitation.
>     
> - The fundamental decision is:
>     
> 
> ```text
> Known destination?
>     └── YES → SSH -L
> 
> Unknown/multiple internal destinations?
>     └── YES → SSH -D + SOCKS + ProxyChains
> ```
> 
> **The real pentesting thought process is:**
> 
> ```text
> Compromise
>    ↓
> Inspect interfaces/routes
>    ↓
> Discover hidden network
>    ↓
> Establish pivot
>    ↓
> Enumerate internal hosts
>    ↓
> Identify services
>    ↓
> Apply the appropriate attack methodology
>    ↓
> Gain new access
>    ↓
> Re-enumerate
>    ↓
> Pivot again if necessary
> ```