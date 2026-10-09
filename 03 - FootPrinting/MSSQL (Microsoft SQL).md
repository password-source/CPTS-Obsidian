## 🔍 Footprinting & Scanning (Nmap & Metasploit)
* **Comprehensive Nmap Script Scan**: 
  `sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 <target_ip>`[cite: 2]
* **Metasploit Auxiliary Scanner (`mssql_ping`)**: 
  * Set RHOSTS: `set rhosts <target_ip>`[cite: 2]
  * Run scan to gather server, instance name, version, and port details[cite: 2].

---

## 🔌 Connecting to MSSQL (Impacket)
* **Locating mssqlclient.py**: 
  `locate mssqlclient`[cite: 2]
* **Connecting with Windows Authentication / Domain Account**: 
  `python3 mssqlclient.py <domain>/<username>@<target_ip> -windows-auth`[cite: 2]
* **Connecting with Local/SQL Authentication**: 
  `python3 mssqlclient.py <username>:<password>@<target_ip> -windows-auth`[cite: 2]

'xp_cmdshell whoami /priv'


---
