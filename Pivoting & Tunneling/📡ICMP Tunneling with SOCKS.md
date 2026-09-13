# 🖥️ RDP + SOCKS Tunneling with SocksOverRDP

## 🧠 1. The Problem: Pivoting in a Windows-Only Environment

During a penetration test, we may compromise a Windows machine and discover that:

- The compromised network is Windows-heavy.
    
- SSH is unavailable.
    
- We cannot install or use normal SSH-based pivoting.
    
- We can access another Windows machine through **RDP**.
    
- That RDP connection can potentially be used as a transport mechanism for pivoting.
    

This is where **SocksOverRDP** becomes useful.

It uses Windows **Dynamic Virtual Channels (DVC)** from the Remote Desktop Services feature.

```text
Attacker / Foothold
        │
        │ RDP
        ▼
Windows Pivot
        │
        │ Internal network
        ▼
Internal Windows Host
```

The important idea is:

> **Instead of creating a separate SSH tunnel, we use the existing RDP connection itself to carry SOCKS traffic.**

---

# 🔌 2. What Are Dynamic Virtual Channels (DVC)?

RDP supports **Dynamic Virtual Channels**, which allow additional data to be transported through an existing RDP session.

Normal examples include:

- Clipboard transfer
    
- Audio
    
- Other RDP features
    

Conceptually:

```text
                 RDP Connection
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    Clipboard        Audio          DVC
                                      │
                                      ▼
                              Custom traffic
```

SocksOverRDP abuses the DVC mechanism for a legitimate tunneling purpose:

```text
SOCKS traffic
     │
     ▼
Dynamic Virtual Channel
     │
     ▼
RDP connection
     │
     ▼
Windows pivot
```

So RDP becomes the **transport** for the SOCKS connection.

> [!important]  
> The key mental model is:
> 
> **SocksOverRDP does not replace RDP. It uses the existing RDP connection to transport arbitrary packets through an RDP Dynamic Virtual Channel.**

---

# 🧭 3. Lab Topology

The topology from the walkthrough can be represented as:

```text
                         ATTACKER / FOOTHOLD
                         Windows 10
                         10.129.x.x
                              │
                              │ RDP
                              ▼
                     ┌─────────────────┐
                     │   172.16.5.19   │
                     │  Windows Pivot  │
                     │                 │
                     │ SocksOverRDP    │
                     │ Server          │
                     └────────┬────────┘
                              │
                              │ Internal network
                              ▼
                     ┌─────────────────┐
                     │ 172.16.6.155   │
                     │ Internal Host   │
                     │ RDP :3389       │
                     └─────────────────┘
```

The important restriction is:

```text
ATTACKER ─────X─────> 172.16.6.155
```

but:

```text
172.16.5.19 ─────────> 172.16.6.155
```

is possible.

Therefore we use the Windows machine at `172.16.5.19` as the pivot.

---

# 📥 4. Obtain the Required Tools

The attack host needs the required binaries so they can be transferred to the Windows machines.

You need:

1. **SocksOverRDP x64 binaries**
```shell
wget https://www.proxifier.com/download/ProxifierPE.zip -O ProxifierPortable.zip
```
1. **Proxifier Portable**
```shell
wget https://github.com/nccgroup/SocksOverRDP/archive/refs/heads/master.zip -O SocksOverRDP.zip
```
    

The Proxifier package used in the walkthrough is:

```text
ProxifierPE.zip
```

The important methodology is:

```text
Download tools
      ↓
Keep them on attack host
      ↓
RDP into Windows target
      ↓
Transfer required files
      ↓
Configure tunnel
```

> [!tip]  
> In a real engagement, always consider the architecture of the Windows target. The walkthrough uses the x64 version of SocksOverRDP because the target is a 64-bit Windows system.

---

# 📦 5. Transfer SocksOverRDP to the Windows Target

Connect to the target using RDP.

The walkthrough uses:

```text
xfreerdp
```

to connect from the attack host.

Once connected, transfer:

```text
SocksOverRDPx64.zip
```

to the Windows target.

After extraction, the important files include:

```text
SocksOverRDP-Plugin.dll
SocksOverRDP-Server.exe
```

At this point:

```text
Attack Host
    │
    │ RDP file transfer
    ▼
Windows Target
    │
    ├── SocksOverRDP-Plugin.dll
    └── SocksOverRDP-Server.exe
```

---

# 🧩 6. Register the SocksOverRDP Plugin

On the Windows target, open a command prompt in the SocksOverRDP directory:

```cmd
cd C:\Users\htb-student\Desktop\SocksOverRDP-x64
```

Then register the plugin:

```cmd
regsvr32.exe SocksOverRDP-Plugin.dll
```

### What is happening?

`regsvr32.exe` is a Windows utility used to register DLL components that expose the appropriate registration functionality.

Here it loads/registers:

```text
SocksOverRDP-Plugin.dll
```

This makes the SocksOverRDP plugin available to the RDP client.

The flow is:

```text
SocksOverRDP-Plugin.dll
          │
          ▼
      regsvr32
          │
          ▼
Windows RDP client can use plugin
```

> [!important]  
> `regsvr32` is being run **on the Windows machine**, not on Kali.

---

# 🖥️ 7. Connect to the Pivot Using MSTSC

After the plugin is registered, connect to:

```text
172.16.5.19
```

using Windows' native RDP client:

```text
mstsc.exe
```

Credentials used in the walkthrough:

```text
victor:pass@123
```

When the RDP session starts, SocksOverRDP should indicate that the plugin is enabled.

It will listen locally on:

```text
127.0.0.1:1080
```

This is extremely important.

The SOCKS listener is now:

```text
127.0.0.1:1080
```

on the Windows machine running the RDP session.

---

# 🧠 8. Understand What `127.0.0.1:1080` Means

`127.0.0.1` means:

```text
localhost
```

Port:

```text
1080
```

is being used as the SOCKS listener.

So:

```text
Windows foothold
      │
      └── 127.0.0.1:1080
             │
             ▼
        SocksOverRDP
```

Applications running on that Windows machine can send traffic to:

```text
127.0.0.1:1080
```

and SocksOverRDP will transport that traffic through the RDP connection.

---

# 🖥️ 9. Start SocksOverRDP-Server on the Internal Windows Host

We also need the server component on:

```text
172.16.5.19
```

Transfer either:

```text
SocksOverRDPx64.zip
```

or just:

```text
SocksOverRDP-Server.exe
```

to that machine.

Then start:

```text
SocksOverRDP-Server.exe
```

with **Administrator privileges**.

The important architecture is now:

```text
Windows 10 / Foothold
        │
        │ RDP
        ▼
172.16.5.19
        │
        │ SocksOverRDP Server
        ▼
172.16.6.155
```

> [!important]  
> There are two sides to understand:
> 
> **Plugin/client side:** creates the SOCKS endpoint associated with the RDP session.
> 
> **Server side:** receives the tunneled traffic on the RDP-connected Windows machine and uses it to reach the internal destination.

---

# 🔎 10. Verify the SOCKS Listener

On the Windows foothold, run:

```cmd
netstat -antb | findstr 1080
```

Expected output:

```text
TCP    127.0.0.1:1080    0.0.0.0:0    LISTENING
```

This confirms that something is listening on:

```text
127.0.0.1:1080
```

### Read the output

```text
127.0.0.1:1080
     │
     ├── 127.0.0.1 = local machine only
     └── 1080      = SOCKS listener
```

The state:

```text
LISTENING
```

means the TCP port is ready to accept connections.

> [!tip]  
> `netstat` is being used here as a troubleshooting/verification step. Before trying to proxy traffic, confirm that the expected local listener actually exists.

---

# 🧦 11. Where Proxifier Fits

Now we have:

```text
127.0.0.1:1080
```

but applications do not automatically know that they should send their traffic there.

This is where **Proxifier** is used.

The role of Proxifier is:

```text
Application
     │
     ▼
Proxifier
     │
     ▼
SOCKS5 127.0.0.1:1080
```

Instead of modifying every application's network configuration individually, Proxifier intercepts the application's connections and sends them through the configured proxy.

---

# ⚙️ 12. Configure Proxifier

Transfer:

```text
Proxifier Portable
```

to the Windows foothold.

The walkthrough uses the portable package:

```text
ProxifierPE.zip
```

Configure Proxifier with a SOCKS5 proxy:

```text
Proxy type: SOCKS5
Address:    127.0.0.1
Port:       1080
```

The resulting architecture:

```text
mstsc.exe
    │
    ▼
Proxifier
    │
    ▼
SOCKS5
127.0.0.1:1080
    │
    ▼
SocksOverRDP
    │
    ▼
RDP Dynamic Virtual Channel
    │
    ▼
172.16.5.19
    │
    ▼
172.16.6.155
```

---

# 🔄 13. The Complete Traffic Flow

This is the most important part to understand.

Suppose we want to RDP to:

```text
172.16.6.155:3389
```

The attacker cannot directly reach it.

Instead:

```text
┌───────────────────────────────────────────────────────┐
│                 WINDOWS FOOTHOLD                      │
│                                                       │
│  mstsc.exe                                            │
│      │                                                │
│      ▼                                                │
│  Proxifier                                            │
│      │                                                │
│      ▼                                                │
│  SOCKS5 127.0.0.1:1080                               │
└──────┬────────────────────────────────────────────────┘
       │
       │ SocksOverRDP
       │
       ▼
┌───────────────────────────────────────────────────────┐
│                 172.16.5.19                           │
│                                                       │
│         RDP Dynamic Virtual Channel                   │
│                     │                                 │
│                     ▼                                 │
│           SocksOverRDP-Server.exe                     │
└─────────────────────┬─────────────────────────────────┘
                      │
                      │ TCP/3389
                      ▼
               172.16.6.155
```

In one line:

```text
mstsc → Proxifier → SOCKS 127.0.0.1:1080 → RDP DVC → SocksOverRDP Server → 172.16.6.155:3389
```

---

# 🧠 14. Why This Works

The important insight is that the attacker already has an RDP connection to the pivot.

Instead of trying to establish another network connection through a restricted path:

```text
Attacker ────X────> Internal Host
```

we reuse the existing RDP connection:

```text
Attacker
   │
   │ Existing RDP connection
   ▼
Windows Pivot
   │
   │ Internal network access
   ▼
Internal Host
```

SocksOverRDP effectively turns:

```text
Existing RDP connection
```

into:

```text
SOCKS transport
```

That is the key idea behind this technique.

---

# 🎯 15. Connecting to the Internal RDP Host

Once Proxifier is configured and running, start:

```text
mstsc.exe
```

and connect to:

```text
172.16.6.155
```

Proxifier redirects the connection:

```text
mstsc.exe
   │
   ▼
Proxifier
   │
   ▼
127.0.0.1:1080
   │
   ▼
SocksOverRDP
   │
   ▼
RDP connection
   │
   ▼
172.16.5.19
   │
   ▼
SocksOverRDP-Server.exe
   │
   ▼
172.16.6.155:3389
```

The important part is that `mstsc.exe` does **not** need to know the details of the pivot.

From the application's perspective, it simply wants:

```text
172.16.6.155:3389
```

Proxifier handles the redirection.

---

# 🔀 16. Why This Is Different From SSH Pivoting

In a Linux environment, you might normally think:

```text
SSH
 ↓
-L / -R / -D
 ↓
SOCKS
```

But on a Windows-only network:

```text
SSH may not be available
```

So instead:

```text
RDP
 ↓
Dynamic Virtual Channel
 ↓
SocksOverRDP
 ↓
SOCKS
```

The transport is different, but the underlying pivoting concept is the same.

### Mental comparison

```text
SSH-based pivot:

Application
    ↓
SOCKS
    ↓
SSH
    ↓
Pivot
    ↓
Internal network


RDP-based pivot:

Application
    ↓
SOCKS
    ↓
SocksOverRDP
    ↓
RDP / DVC
    ↓
Pivot
    ↓
Internal network
```

---

# 🧭 17. The CPTS Methodology Behind the Technique

Don't memorize the individual commands first.

Start with the network problem:

```text
I have access to a Windows machine.
              │
              ▼
It can reach an internal network.
              │
              ▼
I need to access an internal host.
              │
              ▼
SSH is unavailable.
              │
              ▼
I already have RDP access.
              │
              ▼
Can RDP carry additional traffic?
              │
              ▼
Yes → Dynamic Virtual Channels
              │
              ▼
SocksOverRDP
              │
              ▼
SOCKS listener
              │
              ▼
Proxifier
              │
              ▼
Internal services
```

This is the same methodology as other pivoting techniques:

> **Find an existing communication path → determine whether it can transport additional traffic → create a proxy/forwarding mechanism → redirect your tools through it.**

---

# 🔍 18. Troubleshooting Methodology

If the pivot doesn't work, don't immediately change tools.

Check the chain from left to right:

```text
1. RDP connection works?
          │
          ▼
2. SocksOverRDP plugin loaded?
          │
          ▼
3. 127.0.0.1:1080 listening?
          │
          ▼
4. SocksOverRDP-Server running?
          │
          ▼
5. Proxifier configured correctly?
          │
          ▼
6. Target reachable from pivot?
          │
          ▼
7. mstsc.exe connection succeeds?
```

For example:

### No `127.0.0.1:1080`

Check:

```cmd
netstat -antb | findstr 1080
```

If nothing appears:

```text
SOCKS listener isn't running
```

Don't troubleshoot the internal host yet.

### SOCKS listener exists but internal host fails

Then investigate:

```text
Proxifier
     ↓
SocksOverRDP
     ↓
Server
     ↓
Pivot → Internal host
```

The failure could be somewhere further down the chain.

---

# ⚡ 19. RDP Performance Considerations

Multiple RDP sessions can become slow.

This is especially noticeable when:

- Multiple RDP sessions are open.
    
- The network connection is slow.
    
- RDP is carrying additional tunneled traffic.
    
- The pivot is under load.
    

Windows RDP provides an **Experience** tab where visual features can be reduced.

The walkthrough recommends setting:

```text
Performance → Modem
```

This reduces bandwidth usage by disabling/reducing some visual features.

Conceptually:

```text
Higher visual quality
       ↓
More bandwidth
       ↓
Slower tunneled RDP
```

Whereas:

```text
Lower visual quality
       ↓
Less bandwidth
       ↓
Better responsiveness
```

> [!tip]  
> When pivoting through RDP, performance is not just a convenience issue. RDP itself is now part of your tunnel, so unnecessary graphical traffic competes with the traffic you are trying to forward.

---

# 🧠 20. The Big Mental Model

Remember the technique as **RDP → DVC → SOCKS → Proxifier → Internal Network**.

```text
                    WINDOWS FOOTHOLD
                  ┌──────────────────┐
                  │                  │
                  │    mstsc.exe     │
                  │        │         │
                  │        ▼         │
                  │    Proxifier     │
                  │        │         │
                  │        ▼         │
                  │ SOCKS5 :1080     │
                  └────────┬─────────┘
                           │
                           │
                     SocksOverRDP
                           │
                           ▼
                    RDP Dynamic
                   Virtual Channel
                           │
                           ▼
                  ┌──────────────────┐
                  │   172.16.5.19    │
                  │   RDP Pivot       │
                  │                  │
                  │ SocksOverRDP     │
                  │ Server           │
                  └────────┬─────────┘
                           │
                           │ TCP
                           ▼
                  ┌──────────────────┐
                  │  172.16.6.155   │
                  │  Internal Host   │
                  │     :3389        │
                  └──────────────────┘
```

---

# 🔑 21. Key Takeaways

> [!success]  
> **1. RDP can be more than a remote desktop connection.**
> 
> Its Dynamic Virtual Channel functionality can transport additional data.

> [!success]  
> **2. SocksOverRDP turns that capability into a SOCKS-based pivot.**

> [!success]  
> **3. Proxifier redirects application traffic into the SOCKS listener.**

> [!success]  
> **4. The pivot host performs the final connection into the internal network.**

> [!success]  
> **5. The attacker does not need direct network access to the final internal host.**

The complete chain is:

```text
Attacker
   │
   │ RDP
   ▼
Windows Pivot
   │
   │ Dynamic Virtual Channel
   ▼
SocksOverRDP
   │
   ▼
SOCKS :1080
   │
   ▼
Proxifier
   │
   ▼
Application
   │
   ▼
Internal Host
```

### 🧠 The CPTS lesson

```text
Don't ask:
"What command do I run?"

Ask:
"What communication path do I already have?"

        ↓

"I have RDP."

        ↓

"Can RDP transport additional traffic?"

        ↓

"Yes — DVC."

        ↓

"Can I turn that into a proxy?"

        ↓

"Yes — SocksOverRDP."

        ↓

"How do I force applications through it?"

        ↓

"Proxifier."

        ↓

"Now enumerate/attack the internal network."
```

This is the **methodology** you want to carry into other pivoting scenarios—not the specific tool names.