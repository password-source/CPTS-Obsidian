
## 🔍 Enumerating Community Strings (OneSixtyOne)
* **Purpose**: Used to brute-force SNMP community strings for versions 1 and 2c.
* **Installation**: 
  `sudo apt install onesixtyone`
* **Execution Command**: 
  `onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt $TARGETIP$`

---

## 📋 Querying SNMP Data (snmpwalk)
* **Purpose**: Once a valid community string is found, extract detailed system information.
* **Execution Command**: 
  `snmpwalk -v2c -c $comunitystring$ $TARGETIP$`

---
# OIDs (Object Identifiers) Reference

* **What are OIDs**: OIDs (Object Identifiers) are unique, hierarchical addresses structured like dot-separated numbers that point to specific variables or system properties on an SNMP-enabled device.
* **How We Get Them**: Commands like `snmpwalk` automatically traverse and output these OIDs along with their corresponding values, while tools like `braa` brute-force wildcards (e.g., `.1.3.6.*`) to discover active OIDs.
* **How to Identify Them**: When you see an OID, you can identify its meaning by looking up standard enterprise or MIB (Management Information Base) trees, where specific branches translate to hardware, software, or network data.
* **What to Do With It**: Once identified, you can target specific OIDs to extract sensitive details like running processes, installed software, network interfaces, or system configurations.

## 🎯 Brute-Forcing OIDs (Braa)
* **Purpose**: Once a community string is known, use `braa` to brute-force individual OIDs and enumerate the information behind them.
* **Installation**: 
  `sudo apt install braa`
* **Syntax Format**: 
  `braa <community string>@<IP>:.1.3.6.*`
* **Execution Example**: 
  `braa public@10.129.14.128:.1.3.6.*`



example 

# SNMP OID Extraction Example

## 1. Executing the Command (snmpwalk)
* **Command**: 
  `snmpwalk -v2c -c public 10.129.14.128`

## 2. Extracted Output & OIDs
* **System Description (OID: `.1.3.6.1.2.1.1.1.0`)**: 
  `Linux htb 5.11.0-37-generic #41~20.04.2-Ubuntu SMP` -> *Shows the exact OS name and kernel version.*
* **System Uptime (OID: `.1.3.6.1.2.1.1.3.0`)**: 
  `Timeticks: (152839) 0:25:28.39` -> *Shows how long the target machine has been running.*
* **Running Processes (OID: `.1.3.6.1.2.1.25.4.2.1.2`)**: 
  `STRING: "bash"`, `STRING: "sshd"` -> *Enumerates active services and binaries, which can reveal potential vectors or installed software.*