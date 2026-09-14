## 🔐 Secure Shell (SSH)
* **Protocol & Port**: Secure Shell enables encrypted direct connections over the standard port **TCP 22**.
* **Protocol Versions**: 
  * **SSH-1**: Outdated and vulnerable to Man-In-The-Middle (MITM) attacks.
  * **SSH-2**: Advanced protocol version providing superior encryption, speed, stability, and security.
* **Footprinting & Auditing (`ssh-audit`)**: 
  * Tool used to check server-side configurations, banners, and supported encryption/key exchange algorithms.
  * **Execution Example**: `git clone https://github.com/jtesta/ssh-audit.git && cd ssh-audit && ./ssh-audit.py <target_ip>`
* **Client-Side Authentication Controls**: 
  * Force specific authentication methods during troubleshooting or brute-forcing: `ssh -v user@<target_ip> -o PreferredAuthentications=password`

---

## 📁 Rsync File Transfer Protocol
* **Overview**: A fast, versatile tool used for local and remote file copying, mirroring, and backups via its delta-transfer algorithm (transfers only file differences). Default port is **TCP 873**.
* **Scanning for Rsync**: 
  `sudo nmap -sV -p 873 <target_ip>`
* **Probing for Accessible Shares**: 
  `nc -nv <target_ip> 873` followed by `#list`
* **Enumerating an Open Share**: 
  `rsync -av --list-only rsync://<target_ip>/<share_name>`

---

## 🖥️ R-Services (Legacy Remote Management)
* **Overview**: A suite of unencrypted legacy remote access protocols spanning ports **512, 513, and 514** that transmit all data (including passwords) in cleartext.
* **Core R-Commands**:
  * `rcp` (Port 514 / `rshd`): Remote file copy command that overwrites destination files without warning.
  * `rsh` (Port 514 / `rshd`): Opens a remote shell relying on trusted host files (`/etc/hosts.equiv` and `.rhosts`) without login prompts.
  * `rexec` (Port 512 / `rexecd`): Executes remote commands requiring unencrypted username/password validation unless overridden by trusted configuration files.
  * `rlogin` (Port 513 / `rlogind`): Remote login utility working similarly to telnet for Unix-like hosts.
* **Access Control & Misconfigurations**: 
  * `/etc/hosts.equiv`: Global configuration listing trusted hosts and users granted automatic access.
  * `.rhosts`: Per-user configuration file mapping trusted client users and IPs/hostnames. Wildcards (`+`) or misconfigurations can allow unauthenticated code execution or login access (e.g., `rlogin <target_ip> -l <username>`).
* **Network User Enumeration**:
  * `rwho`: Lists active interactive user sessions across the network (queries UDP port 513).
  * `rusers -al <target_ip>`: Provides detailed logs of active user sessions, TTYs, idle times, and source hosts.