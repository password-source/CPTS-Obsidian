# 🌐 1. Credential Hunting in Network Traffic

> [!info]  
> **Goal:** Identify credentials and other sensitive information exposed in **unencrypted network traffic**.
> 
> Even though modern applications commonly use TLS/HTTPS, **legacy systems, misconfigured services, and test applications** may still use plaintext protocols. These can expose usernames, passwords, hashes, SNMP community strings, and other sensitive data directly in captured traffic.

---

## 🧠 2. Core Concept — Credentials in Transit

Most modern applications encrypt sensitive traffic:

```text
Client
   │
   │ 🔐 TLS Encryption
   ▼
Server
```

But if a service uses an unencrypted protocol:

```text
Client
   │
   │ ❌ Plaintext traffic
   │   username=admin
   │   password=P@ssw0rd
   ▼
Server

        ↓
   👁️ Attacker capturing traffic
        ↓
   🔑 Credentials exposed
```

### 🎯 Why This Matters

If you obtain a **packet capture (`.pcap` / `.pcapng`)** or have access to a network interface, you can inspect traffic for:

- 👤 Usernames
    
- 🔑 Passwords
    
- 🎫 Authentication headers
    
- 🧬 NTLM hashes
    
- 📡 SNMP community strings
    
- 💳 Credit card numbers
    
- 🔐 Kerberos authentication material
    

> [!warning]  
> **Encryption is protocol-dependent.** Seeing `HTTP`, `FTP`, or traditional `SNMP` traffic does not automatically mean credentials are exposed, but these protocols can carry sensitive information in cleartext when encryption is not otherwise applied.

---

# 🔄 3. Plaintext vs Encrypted Protocols

The following table is an important reference when analyzing network traffic:

|Unencrypted ProtocolEncrypted CounterpartDescription|||
|---|---|---|
|`HTTP`|`HTTPS`|Used for transferring web pages and resources over the internet.|
|`FTP`|`FTPS/SFTP`|Used for transferring files between a client and a server.|
|`SNMP`|`SNMPv3 (with encryption)`|Used for monitoring and managing network devices like routers and switches.|
|`POP3`|`POP3S`|Retrieves emails from a mail server to a local client.|
|`IMAP`|`IMAPS`|Accesses and manages email messages directly on the mail server.|
|`SMTP`|`SMTPS`|Sends email messages from client to server or between mail servers.|
|`LDAP`|`LDAPS`|Queries and modifies directory services like user credentials and roles.|
|`RDP`|`RDP (with TLS)`|Provides remote desktop access to Windows systems.|
|`DNS (Traditional)`|`DNS over HTTPS (DoH)`|Resolves domain names into IP addresses.|
|`SMB`|`SMB over TLS (SMB 3.0)`|Shares files, printers, and other resources over a network.|
|`VNC`|`VNC with TLS/SSL`|Allows graphical remote control of another computer.|

> [!important]  
> **CPTS mental shortcut:** When analyzing traffic, first identify whether the protocol is **encrypted or plaintext**. Plaintext protocols are prime candidates for credential hunting.

---

# 🦈 4. Wireshark

Wireshark is a well-known packet analyzer commonly available on penetration-testing Linux distributions.

It can analyze:

- 🔴 Live network traffic
    
- 📦 Saved packet captures
    
- 🌐 Network protocols
    
- 🔐 Authentication traffic
    
- 🔎 Specific packets/streams
    

Its most important feature for this topic is the **display filter engine**, which lets you narrow thousands of packets down to the traffic you're interested in.

---

## 🔎 5. Essential Wireshark Filters

|Wireshark filterDescription||
|---|---|
|`ip.addr == 56.48.210.13`|Filters packets with a specific IP address|
|`tcp.port == 80`|Filters packets by port (HTTP in this case).|
|`http`|Filters for HTTP traffic.|
|`dns`|Filters DNS traffic, which is useful to monitor domain name resolution.|
|`tcp.flags.syn == 1 && tcp.flags.ack == 0`|Filters SYN packets (used in TCP handshakes), useful for detecting scanning or connection attempts.|
|`icmp`|Filters ICMP packets (used for Ping), which can be useful for reconnaissance or network issues.|
|`http.request.method == "POST"`|Filters for HTTP POST requests. In the case that POST requests are sent over unencrypted HTTP, it may be the case that passwords or other sensitive information is contained within.|
|`tcp.stream eq 53`|Filters for a specific TCP stream. Helps track a conversation between two hosts.|
|`eth.addr == 00:11:22:33:44:55`|Filters packets from/to a specific MAC address.|
|`ip.src == 192.168.24.3 && ip.dst == 56.48.210.3`|Filters traffic between two specific IP addresses. Helps track communication between specific hosts.|

---

## 🎯 6. High-Value Filters for Credential Hunting

### 🌐 HTTP

```wireshark
http
```

Shows HTTP traffic.

A useful follow-up is:

```wireshark
http.request.method == "POST"
```

Why?

Many web forms submit credentials using HTTP POST requests:

```text
Browser
   │
   │ POST /login
   │ username=admin
   │ password=Password123
   ▼
Web Server
```

If this occurs over **unencrypted HTTP**, the form data may be visible directly in the packet capture.

---

### 🔎 Searching for Password-Related Strings

Wireshark can search packet contents for specific strings:

```wireshark
http contains "passw"
```

This can locate HTTP packets containing the string `passw`.

For example:

```text
POST /login

username=admin
password=Password123
```

> [!tip]  
> Searching for fragments such as `passw` can be useful because it may match variations such as `password`, `passwd`, or similar fields.

---

## 🖱️ 7. Wireshark — Find Packet

Another way to search packet contents is:

```text
Edit
  ↓
Find Packet
  ↓
Enter search string
  ↓
"passw"
```

This is useful when you don't already know which display filter to use.

> [!important]  
> Learn Wireshark's filter syntax rather than relying exclusively on the GUI. During traffic analysis, filters allow you to rapidly isolate relevant conversations.

---

# 🧵 8. Following a TCP Conversation

A packet capture can contain thousands of individual packets.

Instead of analyzing each packet independently, identify the relevant TCP stream:

```wireshark
tcp.stream eq 53
```

This isolates **TCP stream 53**, allowing you to follow the conversation between the communicating hosts.

### 🧠 Mental Model

```text
PCAP
 │
 ├── TCP Stream 0
 ├── TCP Stream 1
 ├── TCP Stream 2
 ├── ...
 ├── TCP Stream 53  ← 🎯 Interesting conversation
 │       │
 │       ├── Request
 │       ├── Response
 │       ├── POST data
 │       └── Credentials
 │
 └── TCP Stream 54
```

---

# 🧰 9. Pcredz

[PCredz GitHub repository](https://github.com/lgandx/PCredz?utm_source=chatgpt.com)

**Pcredz** is a tool designed to extract credentials and other sensitive authentication information from:

- 📡 Live network traffic
    
- 📦 Packet capture files
    

Instead of manually searching through packets in Wireshark, Pcredz can automatically scan traffic for known credential patterns.

---

## 🔑 10. What Pcredz Can Extract

Pcredz supports extraction of:

- 💳 Credit card numbers
    
- 📧 POP credentials
    
- 📤 SMTP credentials
    
- 📥 IMAP credentials
    
- 📡 SNMP community strings
    
- 📁 FTP credentials
    
- 🌐 HTTP NTLM/Basic authentication headers
    
- 📝 HTTP form credentials
    
- 🔐 NTLMv1/v2 hashes
    
    - DCE-RPC
        
    - SMBv1/v2
        
    - LDAP
        
    - MSSQL
        
    - HTTP
        
- 🎫 Kerberos **AS-REQ Pre-Auth etype 23** hashes
    

> [!important]  
> Pcredz is particularly useful when you have a **PCAP** and want to quickly determine whether authentication material or credentials are present without manually inspecting every packet.

---

# 🛠️ 11. Running Pcredz Against a PCAP

Pcredz can be installed by either:

1. Cloning the repository and installing its dependencies.
    
2. Using the provided Docker container.
    

Once installed, the following command can scan a packet capture:

```bash
./Pcredz -f demo.pcapng -t -v
```

### 🔎 Important Options

|Option|Meaning|
|---|---|
|`-f demo.pcapng`|Analyze the specified packet capture file|
|`-t`|Enable the relevant scanning functionality used by Pcredz|
|`-v`|Verbose output|

---

# 📊 12. Understanding Pcredz Output

Example:

```text
Pcredz 2.0.2
Author: Laurent Gaffie
Please send bugs/comments/pcaps to: laurent.gaffie@gmail.com
This script will extract NTLM (HTTP,LDAP,SMB,MSSQL,RPC, etc), Kerberos,
FTP, HTTP Basic and credit card data from a given pcap file or from a live interface.

CC number scanning activated

Unknown format, trying TCPDump format

[1746131482.601354] protocol: udp 192.168.31.211:59022 > 192.168.31.238:161
Found SNMPv2 Community string: s3cr...SNIP...

[1746131482.601640] protocol: udp 192.168.31.211:59022 > 192.168.31.238:161
Found SNMPv2 Community string: s3cr...SNIP...

<SNIP>

[1746131482.658938] protocol: tcp 192.168.31.243:55707 > 192.168.31.211:21
FTP User: le...SNIP...
FTP Pass: qw...SNIP...

demo.pcapng parsed in: 1.82 seconds (File size 15.5 Mo).
```

### 🧠 Reading the Output

The output immediately tells us several important things:

```text
protocol: udp
192.168.31.211:59022
        ↓
192.168.31.238:161

Found SNMPv2 Community string
        ↓
🔑 SNMP credential-like authentication material
```

And later:

```text
protocol: tcp
192.168.31.243:55707
        ↓
192.168.31.211:21

FTP User: le...SNIP...
FTP Pass: qw...SNIP...
```

Port `21` identifies **FTP**, and Pcredz has recovered both the FTP username and password from the traffic.

---

# 🔄 13. Credential Hunting Workflow

A useful CPTS workflow is:

```text
                📦 PCAP / LIVE TRAFFIC
                         │
                         ▼
                 🔎 Identify Protocol
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
       🔐 Encrypted             ❌ Plaintext
             │                       │
       Usually harder                ▼
       to inspect             🦈 Wireshark
                                     │
                       ┌─────────────┴─────────────┐
                       ▼                           ▼
                Display Filters               Pcredz
                       │                           │
                       ▼                           ▼
                Find packets              Automatic extraction
                       │                           │
                       └─────────────┬─────────────┘
                                     ▼
                              🔑 Credentials
                                     │
                                     ▼
                         🎯 Validate / Reuse
```

---

# 🧠 14. CPTS Mental Model

Think of network credential hunting as:

```text
📦 CAPTURE
    ↓
🌐 IDENTIFY PROTOCOL
    ↓
🔐 Is it encrypted?
    ↓
   NO
    ↓
🔎 FILTER TRAFFIC
    ↓
📝 SEARCH FOR AUTH DATA
    ↓
🔑 EXTRACT CREDENTIALS
    ↓
🎯 VALIDATE / USE
```

### The Key Question

Whenever you receive a packet capture, ask:

> **"What protocols are communicating, and is authentication data traveling in cleartext?"**

Then:

```text
HTTP?  → 🔎 Inspect requests / POST data
FTP?   → 🔎 Look for USER / PASS
SNMP?  → 🔎 Look for community strings
LDAP?  → 🔎 Look for authentication material
SMB?   → 🔎 Look for NTLM authentication
HTTP?  → 🔎 Look for Basic / NTLM / form credentials
Kerberos? → 🔎 Look for supported authentication material
```

---

# ⚡ 15. Wireshark Credential-Hunting Cheat Sheet

|Objective|Filter / Technique|
|---|---|
|Find HTTP|`http`|
|Find HTTP POST requests|`http.request.method == "POST"`|
|Search HTTP packets for password-related strings|`http contains "passw"`|
|Find traffic for an IP|`ip.addr == 56.48.210.13`|
|Find traffic on HTTP port|`tcp.port == 80`|
|Follow a specific TCP conversation|`tcp.stream eq 53`|
|Find DNS traffic|`dns`|
|Find ICMP traffic|`icmp`|
|Find SYN packets|`tcp.flags.syn == 1 && tcp.flags.ack == 0`|
|Find traffic between two hosts|`ip.src == 192.168.24.3 && ip.dst == 56.48.210.3`|
|Find packets by MAC address|`eth.addr == 00:11:22:33:44:55`|
|Automated credential extraction|`./Pcredz -f demo.pcapng -t -v`|

---

# ⚠️ 16. Important Distinction — Plaintext vs Hashes

Not everything found in network traffic is necessarily a plaintext password.

You may encounter:

```text
🔑 Plaintext password
      ↓
   password123

🧬 NTLM hash
      ↓
   Authentication material

📡 SNMP community string
      ↓
   Community-based authentication

🎫 Kerberos authentication material
      ↓
   Potentially crackable / reusable material
```

> [!warning]  
> **Credential hunting is broader than searching for `username=password`.** Network captures can contain hashes, authentication headers, community strings, and other material that may enable authentication or offline attacks.

---

# 🎯 17. Key Takeaways

> [!success]
> 
> - **Plaintext protocols can expose credentials directly in network traffic.**
>     
> - **Wireshark** allows detailed packet filtering and inspection.
>     
> - `http.request.method == "POST"` is useful for finding HTTP form submissions.
>     
> - `http contains "passw"` can search HTTP packets for password-related strings.
>     
> - `tcp.stream eq <number>` isolates a specific TCP conversation.
>     
> - **Pcredz** automates credential extraction from live traffic and PCAP files.
>     
> - Pcredz can identify **FTP credentials, HTTP authentication data, SNMP community strings, NTLM hashes, Kerberos hashes**, and other sensitive information.
>     
> - When given a PCAP, first identify the **protocols**, determine whether traffic is **encrypted**, then search the interesting streams for authentication material.
>     

## 🧠 One-Line Exam Mental Model

**PCAP → Identify Protocol → Check Encryption → Filter → Inspect/Extract Authentication Material → Validate Credentials**