# 🪟 1. SSH for Windows: `plink.exe`

> [!info] What is Plink?  
> **Plink** stands for **PuTTY Link**. It is the command-line SSH client included with the **PuTTY** package for Windows.
> 
> Functionally, Plink can perform many of the same SSH tunneling tasks we have already seen with Linux `ssh`, including:
> 
> - 🔀 Dynamic port forwarding
>     
> - 🧦 SOCKS proxy creation
>     
> - 🔗 SSH-based pivoting
>     
> - 🖥️ Command-line SSH connections
>     
> 
> The important CPTS concept is:
> 
> **Plink gives us an SSH-based pivoting capability from a Windows machine.**
> 
> ```text
> Linux:
>     ssh -D 9050 user@pivot
> 
> Windows:
>     plink -ssh -D 9050 user@pivot
> ```
> 
> The syntax is different, but the underlying networking concept is essentially the same.

---

# 🧠 2. Why Plink Matters in a Pentest

Historically, Windows did not include a native OpenSSH client, so administrators commonly used **PuTTY** to establish SSH connections.

This creates an interesting pentesting situation.

Imagine:

```text
You compromise a Windows machine
            │
            ▼
Enumerate the host
            │
            ▼
Host is relatively locked down
            │
            ▼
You need this host as a pivot
            │
            ▼
Uploading custom tools may trigger detection
            │
            ▼
Look for existing software
            │
            ▼
PuTTY / plink.exe already exists
            │
            ▼
Use Plink to establish SSH pivot
```

This is an example of **Living off the Land (LOTL)** thinking.

Instead of immediately dropping a custom tunneling tool:

```text
Custom tool
    ↓
Transfer
    ↓
Execute
    ↓
Potential detection
```

we can potentially use software that is **already installed or legitimately available**.

> [!important] CPTS Methodology  
> The lesson isn't simply:
> 
> ```text
> "Use Plink."
> ```
> 
> The broader methodology is:
> 
> ```text
> Need a capability
>       ↓
> Check what the compromised host already has
>       ↓
> Identify legitimate tools that provide that capability
>       ↓
> Use the existing functionality
>       ↓
> Reduce unnecessary tool transfer
> ```
> 
> This is why enumeration of the compromised host matters before deciding how to pivot.

---

# 🔎 3. When Plink Can Be Useful

There are two important scenarios from this section.

## Scenario A — Compromised Windows Host

You compromise a Windows machine and discover:

```text
PuTTY / plink.exe
```

is already present.

You need the Windows machine as a pivot into another network.

You can potentially use:

```text
plink.exe
```

to create an SSH tunnel.

---

## Scenario B — Windows as Your Attack Host

You don't necessarily need to be using Linux as your attacking machine.

For example:

```text
Windows Attack Host
        │
        │ Plink
        ▼
Ubuntu Pivot
        │
        ▼
Internal Network
```

Plink can therefore be useful when **Windows itself is your primary attack platform**.

> [!tip] Important Perspective  
> Pivoting is not inherently a Linux technique.
> 
> The important capability is:
> 
> ```text
> Can this host create the required network path?
> ```
> 
> The operating system and tool are secondary.

---

# 🌐 4. Network Topology

The scenario in this section uses a Windows-based attack host.

Conceptually:

```text
                         ATTACK HOST
                    ┌──────────────────┐
                    │ Windows          │
                    │ 10.10.15.5       │
                    │                  │
                    │ plink.exe        │
                    │ SOCKS :9050      │
                    └────────┬─────────┘
                             │
                             │ SSH
                             │
                             ▼
                    ┌──────────────────┐
                    │ Ubuntu Server    │
                    │ Pivot            │
                    │ 10.129.15.50     │
                    └────────┬─────────┘
                             │
                             │
                             ▼
                    ┌──────────────────┐
                    │ Internal Network │
                    │ 172.16.5.0/23    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Windows A        │
                    │ 172.16.5.19      │
                    │ RDP :3389        │
                    └──────────────────┘
```

The objective is to reach internal services that the Windows attack host cannot directly access.

---

# 🔀 5. Dynamic Port Forwarding with Plink

The command is:

```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```

This establishes an SSH connection to the Ubuntu server and creates a **dynamic SOCKS proxy** on the Windows machine.

The key option is:

```text
-D 9050
```

which means:

> Create a local SOCKS proxy listening on port `9050`.

So the resulting architecture is:

```text
Windows Attack Host
        │
        │ SOCKS :9050
        ▼
     Plink
        │
        │ SSH tunnel
        ▼
Ubuntu Pivot
        │
        ▼
Internal Network
```

---

# 🧩 6. Breaking Down the Plink Command

```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```

## `plink`

Launches the PuTTY command-line SSH client.

---

## `-ssh`

Specifies that Plink should use the **SSH protocol**.

```text
-ssh
```

So Plink establishes:

```text
Windows ───── SSH ─────► Ubuntu
```

---

## `-D 9050`

This is the important tunneling option.

```text
-D 9050
```

creates a **dynamic port forward**.

Unlike a static port forward such as:

```text
local:3306 → internal-server:3306
```

dynamic forwarding creates a SOCKS proxy capable of handling connections to different destinations.

Conceptually:

```text
                    SOCKS :9050
                         │
                         ▼
Windows Attack Host ── Plink
                         │
                         │ SSH
                         ▼
                    Ubuntu Pivot
                         │
               ┌─────────┼─────────┐
               │         │         │
               ▼         ▼         ▼
           RDP:3389   SMB:445   HTTP:80
```

This is why dynamic forwarding is particularly useful for **pivoting into an entire internal network**.

---

## `ubuntu@10.129.15.50`

This specifies the SSH destination:

```text
Username: ubuntu
Host:     10.129.15.50
```

Therefore Plink establishes:

```text
Windows Attack Host
        │
        │ SSH
        ▼
10.129.15.50
```

The Ubuntu system becomes the SSH endpoint and therefore the gateway through which proxied traffic can travel.

---

# 🧠 7. Why `-D` Instead of `-L`?

This is a critical distinction.

### Local/static forwarding

```text
-L local_port:destination:destination_port
```

creates a specific forwarding relationship.

For example:

```text
localhost:3306
      ↓
internal-server:3306
```

One local port corresponds to a specific destination.

---

### Dynamic forwarding

```text
-D 9050
```

creates a SOCKS proxy.

The application can request different destinations through the proxy:

```text
SOCKS :9050
     │
     ├──► 172.16.5.19:3389
     ├──► 172.16.5.20:445
     ├──► 172.16.5.30:80
     └──► 172.16.5.40:22
```

The SSH server/pivot becomes the point through which those connections are made.

> [!important] Mental Model
> 
> ```text
> -L
>     = "Forward this particular local port to this particular destination."
> 
> -D
>     = "Create a SOCKS proxy through this SSH connection."
> ```
> 
> For broad network pivoting, `-D` is often much more flexible.

---

# 🧦 8. Plink Creates the SOCKS Endpoint

After:

```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```

we have:

```text
Windows Attack Host
┌───────────────────────────┐
│                           │
│  SOCKS Proxy              │
│  127.0.0.1:9050           │
│         │                 │
│         ▼                 │
│       Plink                │
│         │                 │
│         ▼                 │
│     SSH Tunnel             │
└─────────┬─────────────────┘
          │
          ▼
     Ubuntu Pivot
```

The important point is that applications on the Windows machine can now be configured to use:

```text
127.0.0.1:9050
```

as their SOCKS proxy.

---

# 🖥️ 9. Using Proxifier on Windows

The source introduces another Windows tool:

**Proxifier**

Proxifier can force applications that do not natively support proxy configuration to operate through a SOCKS or HTTPS proxy.

It can also support proxy chaining.

The idea is:

```text
Windows Application
        │
        ▼
    Proxifier
        │
        ▼
SOCKS 127.0.0.1:9050
        │
        ▼
      Plink
        │
        ▼
    SSH Tunnel
        │
        ▼
   Ubuntu Pivot
        │
        ▼
 Internal Target
```

This is particularly useful because normal Windows applications such as RDP clients don't necessarily expose a convenient SOCKS proxy option.

---

# ⚙️ 10. Configure Proxifier

The SOCKS server is configured as:

```text
Address: 127.0.0.1
Port:    9050
Type:    SOCKS4
```

So the configuration is:

```text
127.0.0.1:9050
        │
        ▼
      Plink
        │
        ▼
    SSH Tunnel
```

The `127.0.0.1` is important.

It refers to the **Windows attack host itself**, where Plink is listening.

It does **not** mean that the internal Windows target is localhost.

---

# 🧠 11. Why Proxifier Is Needed

Consider this:

```text
mstsc.exe
   │
   └── wants to connect to 172.16.5.19:3389
```

Normally:

```text
Windows Attack Host
        │
        │ direct connection
        ▼
172.16.5.19:3389
```

may fail because the internal network is unreachable directly.

But we have:

```text
Windows Attack Host
        │
        ▼
127.0.0.1:9050
        │
        ▼
Plink
        │
        ▼
SSH
        │
        ▼
Ubuntu
        │
        ▼
172.16.5.19:3389
```

Proxifier intercepts the application's network connection and sends it through the SOCKS proxy.

So the application does not need to understand the pivot itself.

---

# 🖥️ 12. Starting RDP Through the Pivot

After configuring Proxifier, we can start:

```cmd
mstsc.exe
```

and initiate an RDP connection to the Windows target that allows RDP.

The logical path becomes:

```text
mstsc.exe
    │
    ▼
Proxifier
    │
    ▼
SOCKS 127.0.0.1:9050
    │
    ▼
Plink
    │
    ▼
SSH Tunnel
    │
    ▼
Ubuntu Pivot
    │
    ▼
172.16.5.19:3389
    │
    ▼
Windows A
```

This is the major practical advantage of combining:

```text
Plink + SOCKS + Proxifier
```

You can make a Windows application communicate with internal hosts **without requiring the application itself to understand SSH tunneling**.

---

# 🔄 13. Compare the Traffic Directions

It is useful to compare this with the previous Socat examples.

### Socat Reverse Shell

```text
Target
   │
   ▼
Pivot
   │
   ▼
Attacker
```

### Socat Bind Shell

```text
Attacker
   │
   ▼
Pivot
   │
   ▼
Target
```

### Plink Dynamic Forwarding

The attacker's application initiates a connection through the local SOCKS proxy:

```text
Windows Application
       │
       ▼
SOCKS :9050
       │
       ▼
Plink / SSH
       │
       ▼
Ubuntu Pivot
       │
       ▼
Internal Target
```

The important distinction is that Plink is creating a **general-purpose proxy path**, rather than a one-off reverse/bind shell redirect.

---

# 🧭 14. Where This Fits in the CPTS Pivoting Map

Think about the pivoting techniques as different tools for different requirements:

```text
                    NEED TO PIVOT
                         │
                         ▼
               What do I need?
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
     One service     Many services   Reverse shell
          │              │              │
          ▼              ▼              ▼
      Portfwd /        SOCKS /        SSH -R /
       Socat           ProxyChains    Socat
                         │
                         ▼
                  Windows host?
                         │
                         ▼
                   Plink -D
                         │
                         ▼
                SOCKS on Windows
                         │
                         ▼
                    Proxifier
                         │
                         ▼
             Windows applications
```

The key question is always:

> **What network capability do I need?**

Not:

> **Which command do I remember?**

---

# 🧠 15. Plink vs Linux SSH

The easiest way to understand Plink is to map it directly to the SSH concepts already learned.

|Linux SSH|Windows Plink|Purpose|
|---|---|---|
|`ssh`|`plink`|SSH client|
|`-D 9050`|`-D 9050`|Dynamic SOCKS forwarding|
|SSH server/pivot|SSH server/pivot|Tunnel endpoint|
|ProxyChains|Proxifier|Route applications through SOCKS|
|Linux application|Windows application|Application using the tunnel|

So:

```text
Linux:
ssh -D 9050 ubuntu@10.129.15.50

Windows:
plink -ssh -D 9050 ubuntu@10.129.15.50
```

Both produce the same fundamental concept:

```text
Local SOCKS Proxy
       │
       ▼
SSH Tunnel
       │
       ▼
Pivot
       │
       ▼
Internal Network
```

---

# 🧩 16. Why This Is a Living-off-the-Land Technique

Suppose you've compromised a Windows server.

You need a pivot, but your first thought might be:

```text
Download custom tunneling tool
        ↓
Transfer to Windows
        ↓
Execute
```

However, host security may make that risky.

Instead, enumerate:

```text
Installed software
      │
      ├── PuTTY?
      ├── plink.exe?
      ├── Other legitimate network utilities?
      └── Existing administrative tools?
```

If `plink.exe` is already available:

```text
Existing legitimate tool
        ↓
Configure SSH tunnel
        ↓
Create SOCKS proxy
        ↓
Pivot
```

This can reduce the need to introduce an additional executable.

> [!warning] Security Perspective  
> "Living off the land" does **not** mean a technique is invisible.
> 
> Existing administrative tools can still generate:
> 
> - Process creation events
>     
> - Network connections
>     
> - SSH authentication logs
>     
> - Command-line telemetry
>     
> - Endpoint detection alerts
>     
> 
> The CPTS lesson is to understand **why using existing tooling can sometimes reduce the footprint**, not to assume it guarantees evasion.

---

# 🗺️ 17. Complete End-to-End Mental Model

Start with the problem:

```text
Windows Attack Host
        │
        │ ❌ Cannot directly reach
        ▼
172.16.5.19:3389
```

Find a pivot:

```text
Windows Attack Host
        │
        │ SSH
        ▼
Ubuntu Pivot
        │
        │ Can reach
        ▼
172.16.5.19:3389
```

Create dynamic forwarding:

```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```

Now:

```text
Windows Attack Host
       │
       │ 127.0.0.1:9050
       ▼
    SOCKS / Plink
       │
       │ SSH
       ▼
Ubuntu Pivot
       │
       │ RDP
       ▼
172.16.5.19:3389
```

Add Proxifier:

```text
mstsc.exe
    │
    ▼
Proxifier
    │
    ▼
127.0.0.1:9050
    │
    ▼
Plink
    │
    ▼
SSH
    │
    ▼
Ubuntu
    │
    ▼
172.16.5.19:3389
```

The application behaves as though it has a path to the internal service, even though the actual network path is being provided by the SSH tunnel.

---

# 🧠 18. The Most Important Concept: Local vs Remote Perspective

A common source of confusion in pivoting is asking:

> "Why are we using `127.0.0.1`?"

Because the SOCKS listener exists **on the machine running Plink**.

In this scenario:

```text
Windows Attack Host
10.10.15.5
```

runs:

```text
plink
```

which creates:

```text
127.0.0.1:9050
```

Therefore:

```text
Application → 127.0.0.1:9050
```

means:

> "Send this application's traffic to the SOCKS proxy running locally on my Windows machine."

Then Plink carries it through SSH:

```text
127.0.0.1:9050
        │
        ▼
     Plink
        │
        ▼
      SSH
        │
        ▼
Ubuntu
```

And only **after reaching Ubuntu** does the connection proceed toward the internal destination.

---

# 🔥 19. The CPTS Methodology Behind the Walkthrough

The author's thought process can be reconstructed as:

```text
1. Need access to internal network
             ↓
2. Current Windows host cannot reach it directly
             ↓
3. Find a host that can reach internal network
             ↓
4. Ubuntu can reach the internal network
             ↓
5. SSH is available
             ↓
6. Need access to multiple internal services
             ↓
7. Static forwarding would require one tunnel per service
             ↓
8. Dynamic forwarding is more flexible
             ↓
9. Use Plink -D 9050
             ↓
10. Windows applications need to use SOCKS
             ↓
11. Use Proxifier
             ↓
12. Configure SOCKS = 127.0.0.1:9050
             ↓
13. Launch mstsc.exe
             ↓
14. RDP traffic travels through the pivot
```

This is exactly the kind of reasoning to retain for the CPTS methodology map.

---

# 🧠 20. One-Service vs Many-Service Pivoting

This distinction will become very useful when building the larger pentesting map.

### One specific service

If you only need:

```text
172.16.5.19:3389
```

a static port forward can be appropriate:

```text
Local Port
    ↓
Specific Internal IP:Port
```

### Multiple services

If you need to interact with:

```text
172.16.5.19:3389
172.16.5.20:445
172.16.5.30:80
172.16.5.40:22
```

a SOCKS proxy is more flexible:

```text
                 SOCKS
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   RDP :3389    SMB :445    SSH :22
```

Therefore:

> [!success] Decision Rule
> 
> ```text
> One known service
>     → Static port forwarding
> 
> Multiple internal services
>     → SOCKS / dynamic forwarding
> ```
> 
> And on a Windows system:
> 
> ```text
> Plink -D
>     ↓
> SOCKS proxy
>     ↓
> Proxifier
>     ↓
> Windows applications
> ```

---

# 📌 21. Command Cheat Sheet

### Create a dynamic SOCKS tunnel from Windows

```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```

### SOCKS endpoint

```text
127.0.0.1:9050
```

### Proxifier configuration

```text
Address: 127.0.0.1
Port:    9050
Type:    SOCKS4
```

### Launch Windows RDP client

```cmd
mstsc.exe
```

### Resulting traffic path

```text
mstsc.exe
   ↓
Proxifier
   ↓
127.0.0.1:9050
   ↓
Plink
   ↓
SSH
   ↓
Ubuntu Pivot
   ↓
Internal Network
   ↓
RDP Target
```

---

# 🧠 22. Final Mental Model

> [!success] Remember This  
> **Plink is essentially the Windows equivalent of using SSH for tunneling.**
> 
> The specific command:
> 
> ```cmd
> plink -ssh -D 9050 ubuntu@10.129.15.50
> ```
> 
> means:
> 
> ```text
> Windows
>    │
>    │ Create SOCKS :9050
>    ▼
> Plink
>    │
>    │ SSH
>    ▼
> Ubuntu Pivot
>    │
>    ▼
> Internal Network
> ```
> 
> Then:
> 
> ```text
> Windows Application
>        ↓
>    Proxifier
>        ↓
>  SOCKS :9050
>        ↓
>      Plink
>        ↓
>    SSH Tunnel
>        ↓
>   Ubuntu Pivot
>        ↓
> Internal Service
> ```
> 
> The real CPTS lesson is:
> 
> ```text
> Compromised Windows host
>          ↓
> Need pivot
>          ↓
> Enumerate available tooling
>          ↓
> PuTTY / Plink available?
>          ↓
> Use Plink instead of introducing another tool
>          ↓
> Need multiple internal connections?
>          ↓
> Dynamic forwarding (-D)
>          ↓
> SOCKS proxy
>          ↓
> Proxifier for Windows applications
>          ↓
> Reach internal services
> ```
> 
> **Don't memorize Plink as a random Windows command.**
> 
> Understand it as:
> 
> **Windows → Plink SOCKS → SSH → Pivot → Internal Network.**