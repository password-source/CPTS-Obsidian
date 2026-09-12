# 🚐 1. SSH Pivoting with Sshuttle

> [!info] What is Sshuttle?  
> **Sshuttle** is a Python-based pivoting tool that uses an existing **SSH connection** to route traffic through a remote host.
> 
> Its major advantage over the `SSH + ProxyChains` approach is that it can automatically configure local firewall/NAT rules so that applications can communicate with the internal network **without explicitly using ProxyChains**.
> 
> The core idea is:
> 
> ```text
> Local machine
>      │
>      │ SSH tunnel
>      ▼
> Ubuntu Pivot
>      │
>      ▼
> Internal Network
> ```
> 
> Instead of configuring every application to use:
> 
> ```text
> SOCKS → ProxyChains
> ```
> 
> Sshuttle transparently redirects traffic destined for the specified internal network through the SSH connection.

---

# 🧠 2. Why Sshuttle Exists

The previous SSH pivoting workflow looked roughly like:

```text
Application
     │
     ▼
ProxyChains
     │
     ▼
SOCKS Proxy
     │
     ▼
SSH Tunnel
     │
     ▼
Pivot
     │
     ▼
Internal Target
```

This works, but there is an extra layer:

```text
ProxyChains
```

Sshuttle attempts to simplify this.

Instead:

```text
Nmap
  │
  │ normal network connection
  ▼
Local OS firewall/NAT rules
  │
  ▼
Sshuttle
  │
  │ SSH
  ▼
Pivot
  │
  ▼
Internal Network
```

So applications such as Nmap do not need to be launched with:

```shell
proxychains
```

> [!important] Core Difference  
> **ProxyChains approach:**
> 
> ```text
> Application → ProxyChains → SOCKS → SSH → Pivot
> ```
> 
> **Sshuttle approach:**
> 
> ```text
> Application → OS traffic interception → Sshuttle → SSH → Pivot
> ```
> 
> This is why the walkthrough can run Nmap directly.

---

# 🌐 3. The Network Topology

The scenario can be visualized as:

```text
                         ATTACK HOST
                    ┌──────────────────┐
                    │ Linux / Pwnbox   │
                    │                  │
                    │ Sshuttle         │
                    └────────┬─────────┘
                             │
                             │ SSH
                             ▼
                    ┌──────────────────┐
                    │ Ubuntu Server    │
                    │ 10.129.202.64    │
                    │                  │
                    │      PIVOT       │
                    └────────┬─────────┘
                             │
                             │ Can reach
                             ▼
                    ┌──────────────────┐
                    │ Internal Network │
                    │ 172.16.5.0/23    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Windows / DC01   │
                    │ 172.16.5.19      │
                    │ RDP :3389        │
                    └──────────────────┘
```

The problem is:

```text
Attack Host ─────X─────► 172.16.5.19
```

but:

```text
Attack Host ─────────► Ubuntu
Ubuntu ──────────────► 172.16.5.19
```

Therefore:

```text
Ubuntu = Pivot
```

Sshuttle creates the network path:

```text
Attack Host
     │
     │ Sshuttle / SSH
     ▼
Ubuntu Pivot
     │
     ▼
172.16.5.0/23
```

---

# 📦 4. Installing Sshuttle

The source installs Sshuttle with:

```shell
k33lwa@htb[/htb]$ sudo apt-get install sshuttle

Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages were automatically installed and are no longer required:
    alsa-tools golang-1.15 golang-1.15-doc golang-1.15-go golang-1.15-src
    golang-1.16-src libcmis-0.5-5v5 libct4 libgvm20 liblibreoffice-java
    libmotif-common libqrcodegencpp1 libunoloader-java libxm4
    linux-headers-5.10.0-6parrot1-common python-babel-localedata
    python3-aiofiles python3-babel python3-fastapi python3-pydantic
    python3-slowapi python3-starlette python3-uvicorn sqsh ure-java
Use 'sudo apt autoremove' to remove them.
Suggested packages:
    autossh
The following NEW packages will be installed:
    sshuttle
0 upgraded, 1 newly installed, 0 to remove and 4 not upgraded.
Need to get 91.8 kB of archives.
After this operation, 508 kB of additional disk space will be used.
Get:1 https://ftp-stud.hs-esslingen.de/Mirrors/archive.parrotsec.org rolling/main amd64 sshuttle all 1.0.5-1 [91.8 kB]
Fetched 91.8 kB in 2s (52.1 kB/s) 
Selecting previously unselected package sshuttle.
(Reading database ... 468019 files and directories currently installed.)
Preparing to unpack .../sshuttle_1.0.5-1_all.deb ...
Unpacking sshuttle (1.0.5-1) ...
Setting up sshuttle (1.0.5-1) ...
Processing triggers for man-db (2.9.4-2) ...
Processing triggers for doc-base (0.11.1) ...
Processing 1 added doc-base file...
Scanning application launchers
Removing duplicate launchers or broken launchers
Launchers are updated
```

The important result is:

```text
Setting up sshuttle
```

Sshuttle is now available on the attack host.

---

# 🔀 5. Running Sshuttle

The source uses:

```shell
k33lwa@htb[/htb]$ sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
```

This single command establishes the SSH-based pivot and configures the local traffic redirection.

Break it down:

```text
sshuttle
   │
   ├── -r ubuntu@10.129.202.64
   │       │
   │       └── SSH connection to pivot
   │
   ├── 172.16.5.0/23
   │       │
   │       └── Network to route through pivot
   │
   └── -v
           │
           └── Verbose output
```

---

# 🧩 6. Understanding `-r`

The option:

```text
-r ubuntu@10.129.202.64
```

specifies the remote SSH server.

Equivalent mental model:

```text
Username = ubuntu
Remote SSH host = 10.129.202.64
```

Sshuttle establishes:

```text
Attack Host
     │
     │ SSH
     ▼
10.129.202.64
Ubuntu Pivot
```

The SSH connection is the **transport mechanism** used by Sshuttle.

> [!important] Sshuttle Is Still SSH-Based  
> Sshuttle does not magically create a completely independent tunnel protocol.
> 
> Its model is essentially:
> 
> ```text
> Local traffic interception
>          ↓
>       Sshuttle
>          ↓
>      SSH session
>          ↓
>     Remote Python
>          ↓
>    Internal network
> ```
> 
> This is why the source says Sshuttle is specifically useful for **SSH pivoting**.

---

# 🌐 7. Understanding `172.16.5.0/23`

The final argument is:

```text
172.16.5.0/23
```

This tells Sshuttle:

> Route traffic destined for this network through the SSH pivot.

So instead of:

```text
Nmap → normal routing table → 172.16.5.19
```

we get:

```text
Nmap
 │
 ▼
Sshuttle firewall/NAT rules
 │
 ▼
SSH tunnel
 │
 ▼
Ubuntu
 │
 ▼
172.16.5.19
```

### `/23` Matters

A `/23` contains:

```text
172.16.4.0 - 172.16.5.255
```

when using the conventional CIDR boundary for that address block.

> [!warning] CPTS Accuracy Point  
> Be careful with the notation used in walkthroughs. `172.16.5.0/23` is not the canonical network address for a `/23`; the containing `/23` is `172.16.4.0/23`.
> 
> In practical tooling, however, Sshuttle may normalize the supplied destination/network specification internally. The important lesson is to understand the actual address range being routed rather than blindly memorizing the string.

---

# 🔊 8. Understanding `-v`

The:

```text
-v
```

option enables verbose output.

This is extremely useful for learning what Sshuttle is doing behind the scenes.

The output shows:

```text
Starting sshuttle proxy
```

then:

```text
Starting firewall manager
```

then:

```text
Connecting to server...
```

then:

```text
Connected to server.
```

and finally:

```text
iptables ...
```

This gives us a very useful view of the architecture.

---

# 🧠 9. What Sshuttle Actually Does Behind the Scenes

The source says:

> "sshuttle creates an entry in our iptables to redirect all traffic to the 172.16.5.0/23 network through the pivot host."

This is the central idea.

Sshuttle automatically creates firewall/NAT rules that intercept traffic destined for the selected network.

Conceptually:

```text
Before Sshuttle:

Nmap
 │
 ▼
Normal routing
 │
 ▼
172.16.5.19
     ❌
```

After Sshuttle:

```text
Nmap
 │
 ▼
iptables NAT rule
 │
 ▼
Sshuttle redirector
 │
 ▼
SSH tunnel
 │
 ▼
Ubuntu
 │
 ▼
172.16.5.19
     ✅
```

Therefore, the application itself doesn't need to know that a proxy exists.

---

# 🔥 10. Understanding the Sshuttle Output

The beginning:

```text
Starting sshuttle proxy (version 1.1.0).
```

means Sshuttle is starting its local proxy/redirecting machinery.

Then:

```text
c : Starting firewall manager with command: ['/usr/bin/python3', '/usr/local/lib/python3.9/dist-packages/sshuttle/__main__.py', '-v', '--method', 'auto', '--firewall']
```

shows that Sshuttle is starting its firewall-management component.

The important word is:

```text
--firewall
```

because this component is responsible for configuring the local traffic interception.

---

# 🧱 11. The Firewall Method

The output contains:

```text
fw: ready method name nat.
```

and:

```text
c : Method: nat.
```

This tells us Sshuttle selected a **NAT-based method** for redirecting the traffic.

The conceptual flow is:

```text
Application
     │
     ▼
iptables NAT
     │
     ▼
Sshuttle redirector
     │
     ▼
SSH connection
```

---

# 🔢 12. Sshuttle's Local TCP Redirector

The output shows:

```text
c : TCP redirector listening on ('::1', 12300, 0, 0).
c : TCP redirector listening on ('127.0.0.1', 12300).
```

This is an internal Sshuttle component.

It listens locally on:

```text
127.0.0.1:12300
```

and also on IPv6 localhost:

```text
::1:12300
```

Traffic intercepted by the firewall rules can be redirected to this local Sshuttle component.

Conceptually:

```text
Traffic → 172.16.5.19:3389
              │
              ▼
         iptables NAT
              │
              ▼
     127.0.0.1:12300
              │
              ▼
          Sshuttle
              │
              ▼
          SSH tunnel
```

> [!important] Don't Confuse Ports  
> There are now multiple different ports in the architecture:
> 
> ```text
> SSH
>     → normal SSH service on the pivot
> 
> 12300
>     → Sshuttle's local redirector
> 
> 3389
>     → actual RDP service on Windows
> ```
> 
> `12300` is **not** the RDP port and is **not** the SSH port.

---

# 🔐 13. Sshuttle Starts the Remote Component

The output:

```text
c : Connecting to server...
ubuntu@10.129.202.64's password:
```

shows Sshuttle connecting to the Ubuntu pivot using SSH.

Then:

```text
s: Running server on remote host with /usr/bin/python3 (version 3.8.10)
```

This is important.

Sshuttle uses a Python component on the remote machine to handle the other side of the tunnel.

Conceptually:

```text
Attack Host
┌────────────────────┐
│ Sshuttle Client    │
└─────────┬──────────┘
          │
          │ SSH
          ▼
┌────────────────────┐
│ Ubuntu Pivot       │
│ Sshuttle Server    │
└─────────┬──────────┘
          │
          ▼
   Internal Network
```

This is why the pivot needs Python available.

---

# 🔗 14. The Connection Is Established

The output:

```text
c : Connected to server.
```

means the SSH-based Sshuttle connection is established.

Now the local firewall rules are configured:

```text
fw: setting up.
```

This is the transition from:

```text
SSH connection established
```

to:

```text
Traffic routing configured
```

---

# 🧱 15. Understanding the `iptables` Rules

The output contains:

```text
fw: iptables -w -t nat -N sshuttle-12300
fw: iptables -w -t nat -F sshuttle-12300
fw: iptables -w -t nat -I OUTPUT 1 -j sshuttle-12300
fw: iptables -w -t nat -I PREROUTING 1 -j sshuttle-12300
```

These commands are automatically generated by Sshuttle.

You don't need to manually reproduce them for normal Sshuttle usage.

Their purpose is to create and connect a dedicated NAT chain:

```text
sshuttle-12300
```

---

# 🧩 16. What `-t nat` Means

```text
-t nat
```

selects the Linux **NAT table**.

Sshuttle uses NAT rules to manipulate/redirection traffic.

Conceptually:

```text
Original destination:
172.16.5.19:3389

        ↓

iptables NAT

        ↓

Sshuttle redirector:
127.0.0.1:12300
```

---

# 🧩 17. What `-N sshuttle-12300` Does

```text
-N sshuttle-12300
```

creates a new custom chain:

```text
sshuttle-12300
```

Think of it as creating a dedicated rule container:

```text
nat table
   │
   └── sshuttle-12300
           │
           ├── exclusion rules
           └── redirect rule
```

---

# 🧩 18. What `-F sshuttle-12300` Does

```text
-F sshuttle-12300
```

flushes existing rules from that chain.

This allows Sshuttle to rebuild its rule set cleanly.

---

# 🧩 19. What `OUTPUT` Does

```text
-I OUTPUT 1 -j sshuttle-12300
```

inserts a jump into the custom Sshuttle chain for locally generated traffic.

Conceptually:

```text
Local application
      │
      ▼
OUTPUT chain
      │
      ▼
sshuttle-12300
```

This is why traffic generated by local tools such as Nmap can be intercepted.

---

# 🧩 20. What `PREROUTING` Does

Sshuttle also adds:

```text
-I PREROUTING 1 -j sshuttle-12300
```

This allows traffic arriving at the machine to be processed through the Sshuttle chain as well.

For the CPTS mental model, the key point is:

> Sshuttle is integrating itself into the host's packet-processing path rather than requiring each application to explicitly use a proxy.

---

# 🚫 21. Why Sshuttle Excludes Localhost

The output includes:

```text
fw: iptables -w -t nat -A sshuttle-12300 -j RETURN -m addrtype --dst-type LOCAL
fw: iptables -w -t nat -A sshuttle-12300 -j RETURN --dest 127.0.0.1/32 -p tcp
```

These rules prevent Sshuttle from redirecting traffic that is intended for local addresses.

Conceptually:

```text
Traffic → local machine
       ↓
     RETURN
       ↓
Do not send through pivot
```

This prevents Sshuttle from accidentally intercepting its own local communications.

---

# 🎯 22. The Important Redirect Rule

The key rule is:

```text
fw: iptables -w -t nat -A sshuttle-12300 -j REDIRECT --dest 172.16.5.0/32 -p tcp --to-ports 12300
```

The important components are:

```text
REDIRECT
--dest 172.16.5.0/32
-p tcp
--to-ports 12300
```

Conceptually:

```text
TCP traffic
     │
     │ destination matches routed network
     ▼
172.16.5.x
     │
     ▼
Redirect
     │
     ▼
127.0.0.1:12300
     │
     ▼
Sshuttle
     │
     ▼
SSH tunnel
```

> [!warning] Read the Output Carefully  
> The source command requested:
> 
> ```text
> 172.16.5.0/23
> ```
> 
> but the displayed rule contains:
> 
> ```text
> 172.16.5.0/32
> ```
> 
> Do not blindly assume the displayed output represents the entire `/23` in a simple one-rule form. Sshuttle's internal rule generation/version behavior can normalize or generate rules differently.
> 
> The important conceptual lesson is that Sshuttle creates firewall/NAT interception rules for the requested routed destination(s).

---

# 🧠 23. Why Nmap Works Without ProxyChains

Now we reach the main advantage of Sshuttle.

The source runs:

```shell
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn
```

Notice what is **missing**:

```text
proxychains
```

There is no:

```shell
proxychains nmap ...
```

Why?

Because Sshuttle has already modified the local packet-processing path.

So Nmap simply thinks:

```text
"I am connecting to 172.16.5.19:3389."
```

The OS/Sshuttle machinery handles the redirection underneath.

---

# 🔍 24. Breaking Down the Nmap Command

```shell
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn
```

### `sudo`

Runs Nmap with elevated privileges.

This is relevant because certain Nmap functionality and Sshuttle's networking/firewall interaction require elevated privileges.

---

### `-v`

Verbose output.

```text
-v
```

provides additional information during the scan.

---

### `-A`

Enables aggressive detection features, including:

```text
OS detection
Version detection
Script scanning
Traceroute
```

It therefore attempts to gather significantly more information than a basic port scan.

---

### `-sT`

TCP Connect scan:

```text
-sT
```

This is particularly important when pivoting through user-space TCP forwarding mechanisms.

Instead of crafting raw TCP packets like a SYN scan, Nmap performs normal TCP connections through the operating system.

Conceptually:

```text
Nmap
  │
  │ connect()
  ▼
Operating System
  │
  ▼
Sshuttle interception
  │
  ▼
SSH tunnel
  │
  ▼
Target
```

> [!important] Why `-sT`?  
> This is the same fundamental reason we used `-sT` with ProxyChains in previous pivoting sections.
> 
> User-space proxying/tunneling works naturally with normal TCP connections.
> 
> ```text
> -sT
>     = TCP connect() based
> 
> -sS
>     = SYN/raw packet based
> ```
> 
> When traffic needs to pass through a user-space forwarding mechanism, `-sT` is generally the appropriate choice.

---

### `-p3389`

Only scan port:

```text
3389
```

which is the standard RDP port.

So the question being asked is:

> "Can I reach RDP on `172.16.5.19` through my pivot?"

---

### `172.16.5.19`

The internal Windows target.

The attack host may not have direct routing to this IP, but Sshuttle provides the path.

---

### `-Pn`

Tell Nmap not to perform host discovery first.

Normally Nmap may try to determine whether the host is alive before scanning ports.

With:

```text
-Pn
```

Nmap treats the host as online and proceeds with the requested scan.

This is useful when ICMP/host-discovery traffic may not work through the pivot or may be blocked.

---

# 🖥️ 25. Nmap Result

The scan returns:

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-10-08 10:42 CDT
Nmap scan report for 172.16.5.19
Host is up (0.000051s latency).

PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2025-10-08T16:42:47+00:00; +1h00m05s from scanner time.
| ssl-cert: Subject: commonName=DC01.inlanefreight.local
| Not valid before: 2025-10-07T16:30:30
|_Not valid after:  2026-04-08T16:30:30
|  rdp-ntlm-info:
|   Target_Name: INLANEFREIGHT
|   NetBIOS_Domain_Name: INLANEFREIGHT
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: inlanefreight.local
|   DNS_Computer_Name: DC01.inlanefreight.local
|   Product_Version: 10.0.17763
|_  System_Time: 2025-10-08T16:42:43+00:00
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h00m05s, deviation: 0s, median: 1h00m04s

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.68 seconds
```

The most important result is:

```text
3389/tcp open ms-wbt-server Microsoft Terminal Services
```

Therefore:

```text
172.16.5.19:3389
```

is reachable through the pivot.

---

# 🧠 26. What the Scan Proves

Before Sshuttle:

```text
Attack Host ───X───► 172.16.5.19:3389
```

After Sshuttle:

```text
Attack Host
     │
     │ SSH / Sshuttle
     ▼
Ubuntu Pivot
     │
     ▼
172.16.5.19:3389
     │
     ▼
RDP
```

Nmap successfully detects:

```text
3389/tcp open
```

Therefore the pivot is working.

This is an important distinction:

> **The scan succeeding proves both the tunnel and the internal network path are functioning.**

---

# 🔎 27. What the RDP Enumeration Tells Us

Nmap's RDP scripts provide additional information:

```text
Target_Name: INLANEFREIGHT
NetBIOS_Domain_Name: INLANEFREIGHT
NetBIOS_Computer_Name: DC01
DNS_Domain_Name: inlanefreight.local
DNS_Computer_Name: DC01.inlanefreight.local
Product_Version: 10.0.17763
```

From this we can derive:

```text
Hostname:
DC01

Domain:
INLANEFREIGHT

DNS:
inlanefreight.local

RDP:
3389/tcp
```

This can feed directly into the next stages of enumeration.

For example:

```text
RDP discovered
      ↓
Hostname discovered
      ↓
Domain discovered
      ↓
Potential AD environment
      ↓
Continue enumeration
```

> [!tip] CPTS Methodology  
> Pivoting is not the end goal.
> 
> The pivot exists to make **new enumeration possible**.
> 
> The workflow is:
> 
> ```text
> Pivot
>   ↓
> Reach internal host
>   ↓
> Enumerate service
>   ↓
> Discover new information
>   ↓
> Use that information for the next attack branch
> ```

---

# ⚠️ 28. Understanding the OS Detection Warning

Nmap reports:

```text
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
```

and:

```text
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
```

This does **not** mean the host is inaccessible.

We successfully reached:

```text
3389/tcp
```

The issue is specifically with **OS fingerprinting**.

Nmap's OS detection works better when it has enough information from multiple port states.

Here we only requested:

```text
-p3389
```

and that port is open.

Nmap therefore lacks the open/closed port combination it wants for a reliable fingerprint.

So:

```text
RDP reachable        ✅
OS fingerprint       ❌ unreliable
```

Do not confuse the two.

---

# 🕐 29. Understanding Clock Skew

Nmap reports:

```text
|_clock-skew: mean: 1h00m05s, deviation: 0s, median: 1h00m04s
```

This indicates the target's system clock differs from the scanner's clock by approximately one hour.

In an Active Directory environment, time synchronization can become particularly important because Kerberos authentication is sensitive to clock differences.

Therefore, this is potentially useful information for later enumeration/attacks.

---

# 🧠 30. Sshuttle vs ProxyChains + SOCKS

This distinction is extremely important for your CPTS pivoting map.

|Approach|Traffic Handling|Application Configuration|Typical Use|
|---|---|---|---|
|SOCKS + ProxyChains|Application explicitly uses proxy wrapper|Run command through `proxychains`|Flexible proxy-based pivot|
|Plink `-D` + Proxifier|Applications redirected to local SOCKS proxy|Configure Proxifier|Windows application pivoting|
|Sshuttle|Local firewall/NAT redirects traffic into SSH tunnel|Usually no proxy wrapper required|Transparent-ish SSH network pivot|
|Socat|Direct socket-to-socket relay|Point application/listener at forwarded port|Specific TCP redirection|

The mental difference is:

```text
ProxyChains:

Nmap
 ↓
ProxyChains
 ↓
SOCKS
 ↓
SSH
 ↓
Pivot
```

versus:

```text
Sshuttle:

Nmap
 ↓
Normal TCP connection
 ↓
iptables/NAT interception
 ↓
Sshuttle
 ↓
SSH
 ↓
Pivot
```

---

# 🔥 31. Why Sshuttle Is Convenient

With ProxyChains, you have to remember:

```shell
proxychains nmap ...
```

and ProxyChains must support the application and protocol being used.

With Sshuttle:

```shell
nmap ...
```

can be used normally for the routed destinations.

This is especially convenient when you need to use several different tools:

```text
Nmap
RDP client
SMB tools
Web tools
Other TCP clients
```

Instead of configuring each one individually:

```text
Tool
 ↓
Proxy configuration
 ↓
SOCKS
```

the operating system's traffic path can be redirected by Sshuttle.

---

# ⚠️ 32. Important Sshuttle Limitations

The source explicitly points out that Sshuttle is specialized:

> It works for pivoting over SSH and does not provide other options for pivoting over TOR or HTTPS proxy servers.

Therefore:

```text
Sshuttle
   ↓
SSH-based pivoting
```

It is not a universal proxy framework.

If your available route is:

```text
TOR
```

or:

```text
HTTPS proxy
```

you need another technique.

> [!important] Tool Selection  
> Do not ask:
> 
> **"Is Sshuttle the best pivoting tool?"**
> 
> Ask:
> 
> **"What connectivity do I currently have, and what type of traffic do I need to transport?"**
> 
> Then choose:
> 
> ```text
> SSH available?
>     → Sshuttle
> 
> SOCKS needed?
>     → SOCKS + ProxyChains / Proxifier
> 
> One TCP service?
>     → Port forwarding / Socat
> 
> Reverse connection?
>     → SSH -R / Socat / Meterpreter forwarding
> ```

---

# 🧭 33. How the Author Arrived at This Solution

The walkthrough's reasoning can be reconstructed as:

```text
1. Attack host cannot directly access internal network
                    ↓
2. Ubuntu can access internal network
                    ↓
3. Ubuntu can accept SSH connections
                    ↓
4. Need to route internal network traffic
                    ↓
5. Could use SSH + SOCKS + ProxyChains
                    ↓
6. But want to avoid configuring ProxyChains
                    ↓
7. Sshuttle can automate SSH-based traffic redirection
                    ↓
8. Install Sshuttle
                    ↓
9. Connect to Ubuntu using SSH
                    ↓
10. Tell Sshuttle which network to route
                    ↓
11. Sshuttle configures firewall/NAT rules
                    ↓
12. Run Nmap normally
                    ↓
13. Traffic to internal target is redirected
                    ↓
14. Nmap reaches 172.16.5.19:3389
                    ↓
15. Continue internal enumeration
```

This is exactly the kind of reasoning you should retain rather than memorizing the command alone.

---

# 🗺️ 34. CPTS Pivoting Topology

You can now place Sshuttle into the larger pivoting model:

```text
                         NEED INTERNAL ACCESS
                                  │
                                  ▼
                         Find reachable pivot
                                  │
                                  ▼
                         Can SSH reach pivot?
                                  │
                         ┌────────┴────────┐
                         │                 │
                        YES                NO
                         │                 │
                         ▼                 ▼
                    SSH-based          Other pivot
                    techniques         mechanisms
                         │
            ┌────────────┼─────────────┐
            │            │             │
            ▼            ▼             ▼
         SSH -L        SSH -D       Sshuttle
            │            │             │
            ▼            ▼             ▼
       One service    SOCKS proxy   Route network
                         │             │
                         ▼             ▼
                    ProxyChains    Normal apps
                    /Proxifier
```

This is much more useful than memorizing individual commands because it tells you **when to select the technique**.

---

# 🔄 35. Sshuttle Traffic Flow

The cleanest mental model is:

```text
                 LOCAL MACHINE
                       │
                       │
                 Application
                       │
                       ▼
              ┌────────────────┐
              │ iptables / NAT │
              └───────┬────────┘
                      │
                      │ Destination:
                      │ 172.16.5.x
                      ▼
              ┌────────────────┐
              │   Sshuttle     │
              │   :12300       │
              └───────┬────────┘
                      │
                      │ SSH
                      ▼
              ┌────────────────┐
              │ Ubuntu Pivot   │
              └───────┬────────┘
                      │
                      ▼
              ┌────────────────┐
              │ Internal Host  │
              │ 172.16.5.19    │
              │ RDP :3389      │
              └────────────────┘
```

The key insight:

> **The application does not need to know about the SSH tunnel.**

---

# 🧠 36. The Three Layers You Should Separate

When studying Sshuttle, separate these three things mentally:

## Layer 1 — Application

Example:

```text
Nmap
```

It simply requests:

```text
Connect to 172.16.5.19:3389
```

---

## Layer 2 — Local Traffic Redirection

Sshuttle configures:

```text
iptables
NAT
REDIRECT
```

to intercept the traffic.

---

## Layer 3 — SSH Transport

Sshuttle transports the intercepted traffic through:

```text
SSH
   ↓
Ubuntu
   ↓
Internal network
```

So:

```text
Application
     ↓
Traffic interception
     ↓
Sshuttle
     ↓
SSH
     ↓
Pivot
     ↓
Target
```

Keeping these layers separate makes Sshuttle much easier to understand.

---

# 📌 37. Command Cheat Sheet

### Install Sshuttle

```shell
sudo apt-get install sshuttle
```

### Create SSH-based pivot

```shell
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
```

### Important options

```text
-r
    Remote SSH host

172.16.5.0/23
    Destination network to route through pivot

-v
    Verbose output
```

### Test internal RDP reachability

```shell
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn
```

### Result

```text
172.16.5.19:3389
        ↓
      OPEN
        ↓
Microsoft Terminal Services / RDP
```

---

# 🧠 38. Sshuttle vs the Techniques Immediately Before It

The progression of the previous sections is important:

```text
SSH Reverse Port Forward
        ↓
Specific reverse connection
```

```text
Meterpreter Port Forwarding
        ↓
Forward through an existing Meterpreter session
```

```text
Socat
        ↓
Direct TCP redirection
```

```text
Plink
        ↓
Windows-based SSH dynamic forwarding
```

```text
Sshuttle
        ↓
SSH-based network routing without
manually wrapping applications in ProxyChains
```

So Sshuttle is another answer to the same larger problem:

> **"How do I make my attacking machine reach a network that it cannot directly route to?"**

---

# 🏆 39. Final CPTS Mental Model

> [!success] Remember This
> 
> **Sshuttle = SSH + automatic local traffic redirection.**
> 
> The fundamental workflow is:
> 
> ```text
> Attacker cannot reach internal network
>                 ↓
>        Find SSH-accessible pivot
>                 ↓
>              Ubuntu
>                 ↓
>       Run Sshuttle through SSH
>                 ↓
>       Specify internal network
>                 ↓
>      Sshuttle configures NAT rules
>                 ↓
>        Local traffic is intercepted
>                 ↓
>        Traffic enters Sshuttle
>                 ↓
>          SSH transports it
>                 ↓
>            Ubuntu pivot
>                 ↓
>          Internal target
> ```
> 
> In this example:
> 
> ```text
> sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
> ```
> 
> creates the conceptual path:
> 
> ```text
> Nmap
>   │
>   ▼
> iptables/NAT
>   │
>   ▼
> Sshuttle
>   │
>   ▼
> SSH
>   │
>   ▼
> Ubuntu Pivot
>   │
>   ▼
> 172.16.5.19:3389
> ```
> 
> And unlike the previous SOCKS/ProxyChains approach:
> 
> ```text
> ❌ proxychains nmap ...
> ```
> 
> is not required.
> 
> You can simply use:
> 
> ```text
> nmap ...
> ```
> 
> because Sshuttle has already inserted itself into the local traffic path.
> 
> **The CPTS lesson is not "memorize Sshuttle."**
> 
> The lesson is:
> 
> ```text
> Identify inaccessible network
>          ↓
> Identify pivot
>          ↓
> Confirm SSH connectivity
>          ↓
> Decide whether you need
> network-wide routing or one service
>          ↓
> If SSH + network-wide routing:
>          ↓
>       Sshuttle
>          ↓
> Automatically redirect traffic
>          ↓
> Enumerate internal targets normally
> ```