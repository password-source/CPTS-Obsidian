## 🖥️ Remote Desktop Protocol (RDP)
* **Overview**: A Microsoft-developed protocol for graphical remote access to Windows systems, operating at the application layer over **TCP port 3389** (and optionally connectionless UDP port 3389).
* **Network Requirements**: Requires network and server firewalls to allow incoming connections, alongside a public IP address and port forwarding if Network Address Translation (NAT) is present.
* **Security & Authentication**: 
  * Integrates Transport Layer Security (TLS/SSL) since Windows Vista, though many systems accept lower encryption levels via RDP Security.
  * Identity-providing certificates are typically **self-signed by default**, generating browser or client warnings regarding name mismatches.
  * Windows servers come with Remote Desktop installed by default, typically configured to restrict connections exclusively to hosts utilizing **Network Level Authentication (NLA)**.
* **Footprinting & Enumeration Tools**:
  * **Nmap**: Scans service details, NLA configuration, and NTLM target info using scripts like `--script rdp*`. Note that packet tracing (`--packet-trace`) reveals tracking cookies (e.g., `mstshash=nmap`) that can be flagged by EDR solutions.
  * **RDP Security Check (`rdp-sec-check.pl`)**: A Cisco CX Security Labs Perl script used to unauthentically identify supported RDP protocols (SSL, Hybrid/NLA, RDP security layers) and encryption methods.
* **Connecting via Linux**:
  * Tools such as `xfreerdp`, `rdesktop`, and `Remmina` provide GUI access. 
  * **Example (`xfreerdp`)**: `xfreerdp3 /u:user /p:"password" /v:<target_ip>`

---

## ⚡ Windows Remote Management (WinRM)
* **Overview**: A command-line remote management protocol based on the Simple Object Access Protocol (SOAP). Enabled by default starting with Windows Server 2012.
* **Ports**: Operates over **TCP port 5985** (HTTP) and **TCP port 5986** (HTTPS).
* **Companion Components**: **Windows Remote Shell (WinRS)** allows arbitrary command execution on remote hosts and is included by default on Windows 7 and newer.
* **Footprinting & Interaction**:
  * **Nmap**: `nmap -sV -sC <target_ip> -p5985,5986`
  * **PowerShell**: Use the `Test-WsMan` cmdlet to verify remote endpoint reachability.
  * **Evil-WinRM**: A dedicated pentesting tool used to interact with WinRM endpoints on Linux (`evil-winrm -i <target_ip> -u <user> -p <password>`).

---

## 📊 Windows Management Instrumentation (WMI)
* **Overview**: Microsoft's implementation and extension of the Common Information Model (CIM) and Web-Based Enterprise Management (WBEM) standards, granting read and write access to nearly all Windows settings. Typically accessed via PowerShell, VBScript, or WMIC.
* **Port & Connection Behavior**: Initialization occurs over **TCP port 135**, after which communication shifts to a random high port.
* **Interaction Tool (Impacket)**:
  * **wmiexec.py**: Executes commands against a target utilizing WMI/SMB authentication.
  * **Example**: `/usr/share/doc/python3-impacket/examples/wmiexec.py user:"password"@<target_ip> "hostname"`