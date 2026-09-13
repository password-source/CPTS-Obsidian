# 🪟 1. Port Forwarding with Windows `netsh`

> [!info] What is `netsh`?  
> `netsh.exe` is a built-in Windows command-line utility used to configure and inspect Windows networking.
> 
> For CPTS/pivoting purposes, some particularly useful capabilities include:
> 
> - 🔎 Finding routes
>     
> - 🧱 Viewing firewall configuration
>     
> - 🌐 Configuring proxies
>     
> - 🔀 Creating port-forwarding rules
>     
> 
> In this section, the important feature is **`interface portproxy`**, which allows a Windows machine to listen on one IP/port and forward received TCP connections to another IP/port.

---

# 🌐 2. The Network Scenario

We have compromised a Windows 10 workstation belonging to an IT administrator.

The compromised Windows machine has two network positions:

```text
Windows 10 Pivot
10.129.15.150
172.16.5.19
```

The internal Windows server is:

```text
172.16.5.25
```

and RDP is running on:

```text
TCP/3389
```

The attack host cannot directly reach the internal server:

```text
ATTACK HOST
10.10.15.5
      │
      │ ❌ Cannot directly reach
      ▼
172.16.5.25:3389
```

But the compromised Windows workstation can reach it:

```text
ATTACK HOST
10.10.15.5
      │
      ▼
Windows Pivot
10.129.15.150
172.16.5.19
      │
      │ ✅ Internal access
      ▼
172.16.5.25:3389
```

Therefore, we can use the Windows workstation as a **pivot**.

---

# 🧠 3. The Important Difference: Port Forwarding vs SOCKS

With Rpivot, SSH `-D`, or a Metasploit SOCKS proxy, we created a **SOCKS proxy** that could potentially handle connections to multiple destinations.

`netsh portproxy` works differently.

It creates a **specific listening port that forwards to a specific destination**.

Think:

```text
SOCKS:

Application
    │
    ▼
SOCKS Proxy
    │
    ├────► Target A
    ├────► Target B
    └────► Target C
```

versus:

```text
Netsh portproxy:

10.129.15.150:8080
          │
          │ fixed forwarding rule
          ▼
172.16.5.25:3389
```

So the rule is essentially:

> **"Whenever someone connects to this IP/port on the pivot, forward that TCP connection to this other IP/port."**

---

# 🔀 4. Creating the Port Forward

On the compromised Windows pivot:

```cmd
C:\Windows\system32> netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25
```

This creates the following rule:

```text
LISTEN
10.129.15.150:8080
       │
       │ forward
       ▼
CONNECT
172.16.5.25:3389
```

### Breaking down the command

|Option|Meaning|
|---|---|
|`netsh.exe`|Windows network configuration utility|
|`interface portproxy`|Access the port-proxy functionality|
|`add`|Create a new forwarding rule|
|`v4tov4`|IPv4 listener → IPv4 destination|
|`listenport=8080`|Port the pivot listens on|
|`listenaddress=10.129.15.150`|IP address the pivot listens on|
|`connectport=3389`|Destination port|
|`connectaddress=172.16.5.25`|Destination IP|

The most important relationship is:

```text
listenaddress:listenport
            ↓
10.129.15.150:8080
            │
            ▼
connectaddress:connectport
            ↓
172.16.5.25:3389
```

> [!important]  
> `8080` is **not** the port running RDP.
> 
> RDP is still running on:
> 
> ```text
> 172.16.5.25:3389
> ```
> 
> `8080` is simply the **new entry point exposed by the pivot**.

---

# 🔍 5. Why Did We Choose Port 8080?

There is nothing inherently special about `8080` here.

The logic is:

```text
Original service:
172.16.5.25:3389
```

The attacker needs a port on the pivot through which they can reach that service:

```text
Pivot:
10.129.15.150:8080
```

Therefore:

```text
10.129.15.150:8080
          │
          ▼
172.16.5.25:3389
```

You could potentially choose another available listening port instead.

The important thing is the **mapping**, not the number `8080`.

---

# 🔎 6. Verify the Port Forward

After creating the rule:

```cmd
C:\Windows\system32> netsh.exe interface portproxy show v4tov4
```

Output:

```text
Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  ---------------
10.129.15.150   8080        172.16.5.25     3389
```

This confirms:

```text
10.129.15.150:8080
        │
        ▼
172.16.5.25:3389
```

> [!tip] CPTS habit  
> Whenever you create a forwarding rule, **verify it immediately**.
> 
> Don't assume the configuration succeeded just because the command returned without an obvious error.

---

# 🔄 7. What Happens When We Connect?

Once the rule exists, our attack host connects to:

```text
10.129.15.150:8080
```

The Windows pivot receives the connection.

`netsh portproxy` then forwards it to:

```text
172.16.5.25:3389
```

The complete traffic path is:

```text
┌────────────────┐
│   Attack Host  │
│   10.10.15.5   │
└───────┬────────┘
        │
        │ RDP connection
        │ destination:
        │ 10.129.15.150:8080
        ▼
┌────────────────────────┐
│   Windows Pivot        │
│   10.129.15.150        │
│   172.16.5.19          │
│                        │
│   netsh portproxy      │
└──────────┬─────────────┘
           │
           │ forwarded connection
           ▼
┌────────────────────────┐
│ Internal Windows Server│
│   172.16.5.25:3389     │
│        RDP             │
└────────────────────────┘
```

The key mental model is:

> **The attacker connects to the pivot, while the pivot connects to the internal service.**

---

# 🖥️ 8. Connecting Through the Port Forward

From the attack host, we can use RDP against the pivot's listening port.

For example:

```bash
xfreerdp /v:10.129.15.150:8080 /u:victor /p:'pass@123'
```

The important destination is:

```text
10.129.15.150:8080
```

not:

```text
172.16.5.25:3389
```

because the attack host is using the **port-forward entry point**.

The Windows pivot then translates the connection internally:

```text
Attacker thinks:

10.129.15.150:8080
        │
        ▼
     netsh
        │
        ▼
172.16.5.25:3389
```

> [!warning] Source screenshot vs command  
> The source screenshot shows a connection to `10.129.42.198:8080`, while the written scenario uses `10.129.15.150` as the pivot. The important concept is unchanged: **connect to the pivot's listening IP/port, which `netsh` forwards to the internal RDP service.**

---

# 🧩 9. Why This Technique Is Useful

One major advantage is that `netsh` is **built into Windows**.

You don't necessarily need to upload:

```text
socat
chisel
plink
rpivot
```

or another third-party pivoting utility.

If you already have command execution on a Windows machine, you may be able to use:

```text
netsh.exe
```

as a **living-off-the-land** technique.

This can be particularly useful when:

- 🪟 The compromised machine is Windows.
    
- 🔒 You want to avoid transferring additional binaries.
    
- 🌐 The compromised machine can reach an internal service.
    
- 🔀 You only need to forward a specific TCP service.
    
- 🧰 Built-in Windows functionality is preferable to introducing another tool.
    

---

# 🧠 10. The Methodology Behind Finding This Solution

Don't memorize:

```cmd
netsh interface portproxy add v4tov4 ...
```

Instead, derive it from the topology.

### Step 1 — Find the target

You discover:

```text
172.16.5.25:3389
```

RDP is available.

### Step 2 — Test reachability

Your attack host:

```text
10.10.15.5 ──❌──► 172.16.5.25:3389
```

doesn't have direct access.

### Step 3 — Look for a compromised host with access

You have:

```text
Windows Pivot
10.129.15.150
172.16.5.19
```

Because it has an internal interface:

```text
172.16.5.19
```

it may be able to reach:

```text
172.16.5.25
```

### Step 4 — Decide how to expose the service

You don't necessarily need a full network-level route.

You only need:

```text
Attack Host
      │
      ▼
Pivot
      │
      ▼
RDP Server
```

Therefore, a **specific TCP port forward** is sufficient.

### Step 5 — Look for native functionality

Because the pivot is Windows, ask:

> **Does Windows have a built-in TCP port-forwarding mechanism?**

Yes:

```text
netsh interface portproxy
```

### Step 6 — Create the mapping

```text
10.129.15.150:8080
        ↓
172.16.5.25:3389
```

### Step 7 — Connect

```text
xfreerdp → 10.129.15.150:8080
```

The pivot handles the internal connection.

---

# 🔄 11. Traffic Direction

This is especially important when learning pivoting.

```text
Attack Host
10.10.15.5
     │
     │ TCP connection
     ▼
Pivot
10.129.15.150:8080
     │
     │ netsh forwards
     ▼
Internal Server
172.16.5.25:3389
```

There are effectively two connections:

```text
Connection 1:
Attack Host → Pivot:8080

Connection 2:
Pivot → Internal Server:3389
```

`netsh` bridges these two sides.

> [!important]  
> The pivot is the machine that can see both sides of the network. That is why it can act as the bridge.

---

# 🆚 12. Netsh vs Rpivot vs SOCKS

|Technique|What it provides|Typical use|
|---|---|---|
|**Netsh portproxy**|Specific port → specific destination|RDP, SMB, web server, database|
|**Rpivot**|SOCKS proxy|Multiple internal destinations|
|**SSH `-L`**|Specific local port → remote destination|Forward one service|
|**SSH `-D`**|SOCKS proxy|Multiple destinations|
|**Socat**|Socket relay|Specific service/connection|
|**Sshuttle**|Network-level-ish routing through SSH|Larger internal subnet access|
|**Metasploit `portfwd`**|Specific port forwarding through Meterpreter|Specific internal service|

The important distinction is:

```text
Specific service:
        ↓
Port Forward
        ↓
Netsh / SSH -L / Socat / Meterpreter portfwd
```

versus:

```text
Many internal destinations:
        ↓
SOCKS
        ↓
Rpivot / SSH -D / Metasploit SOCKS
```

---

# 🪟 13. Why Windows `netsh` Is Interesting for CPTS

This technique fits directly into the **Living off the Land** concept.

Instead of:

```text
Compromise Windows
       ↓
Upload third-party pivot tool
       ↓
Execute tool
       ↓
Configure tunnel
```

you can potentially do:

```text
Compromise Windows
       ↓
Use built-in netsh.exe
       ↓
Create portproxy
       ↓
Forward required service
```

This can reduce the number of artifacts you introduce onto the target.

> [!warning] Important limitation  
> `netsh portproxy` is **not a general SOCKS proxy**.
> 
> If you need to reach many different internal IPs and ports dynamically, a SOCKS-based pivot or routing solution is generally more appropriate.
> 
> `netsh portproxy` is most naturally suited to **known, specific TCP forwarding mappings**.

---

# 🧹 14. Removing a Port Forward

If you need to remove the forwarding rule later, use the corresponding `delete` operation.

For the rule created above:

```cmd
netsh.exe interface portproxy delete v4tov4 listenport=8080 listenaddress=10.129.15.150
```

The logic is:

```text
ADD:
listen 10.129.15.150:8080
        ↓
172.16.5.25:3389

DELETE:
remove the listener
10.129.15.150:8080
```

You can then verify the remaining rules with:

```cmd
netsh.exe interface portproxy show v4tov4
```

---

# 🧭 15. CPTS Pivoting Decision Tree

When you discover an internal service:

```text
                 INTERNAL SERVICE
                       │
                       ▼
          Can attack host reach it?
                 /           \
               YES            NO
                │              │
                ▼              ▼
          Access directly   Need pivot
                               │
                               ▼
                    Find compromised host
                               │
                               ▼
                    Can pivot reach target?
                         /          \
                       NO            YES
                       │              │
                       ▼              ▼
                  Find another      Need to
                     path           expose access
                                      │
                         ┌────────────┴────────────┐
                         │                         │
                    One service               Many targets
                         │                         │
                         ▼                         ▼
                  Port forwarding               SOCKS
                         │                         │
               ┌─────────┼─────────┐       ┌──────┼──────┐
               │         │         │       │      │      │
             Netsh     SSH -L    Socat   Rpivot SSH -D Metasploit
```

This is the useful methodology to remember.

---

# 🎯 16. The Core Mental Model

> [!success] Don't memorize the `netsh` syntax first. Memorize the problem it solves.

You have:

```text
ATTACKER
   │
   │ ❌
   ▼
INTERNAL SERVER
```

But:

```text
ATTACKER
   │
   ▼
WINDOWS PIVOT
   │
   │ ✅
   ▼
INTERNAL SERVER
```

So you create:

```text
ATTACKER
   │
   │ connect to
   ▼
PIVOT:8080
   │
   │ netsh portproxy
   ▼
INTERNAL_SERVER:3389
```

The complete transformation is:

```text
"I can't reach 172.16.5.25:3389"
                  │
                  ▼
      "The Windows pivot can."
                  │
                  ▼
      "I need a TCP forward."
                  │
                  ▼
      "Windows has netsh portproxy."
                  │
                  ▼
10.129.15.150:8080 → 172.16.5.25:3389
                  │
                  ▼
      xfreerdp → pivot:8080
                  │
                  ▼
              RDP access
```

# 📌 17. Key Takeaways

- 🪟 `netsh.exe` is a built-in Windows networking utility.
    
- 🔀 `netsh interface portproxy` can create TCP port-forwarding rules.
    
- `v4tov4` means **IPv4 listener → IPv4 destination**.
    
- `listenaddress` + `listenport` define where the pivot accepts connections.
    
- `connectaddress` + `connectport` define where those connections are forwarded.
    
- In this example:
    

```text
10.129.15.150:8080
        ↓
172.16.5.25:3389
```

- The attacker connects to the **pivot's listening port**, not directly to the internal RDP server.
    
- `netsh portproxy` is a **specific port-forwarding mechanism**, not a SOCKS proxy.
    
- It is useful for **living off the land** because `netsh.exe` is built into Windows.
    
- It is particularly useful when you know exactly which internal service you need to reach.
    
- For many destinations, consider a SOCKS or routing-based pivot instead.
    
- Always verify a forwarding rule with:
    

```cmd
netsh.exe interface portproxy show v4tov4
```

> [!important] CPTS takeaway  
> **Topology first → determine reachability → identify the required traffic → choose the smallest pivot mechanism that solves the problem.**
> 
> If you only need:
> 
> ```text
> Attack Host → Pivot → Internal RDP
> ```
> 
> a simple port forward may be enough.
> 
> If you need:
> 
> ```text
> Attack Host → Pivot → many internal hosts/services
> ```
> 
> a SOCKS proxy or routing-based pivot is usually the better model.