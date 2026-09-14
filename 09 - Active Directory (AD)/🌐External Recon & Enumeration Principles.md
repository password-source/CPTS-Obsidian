# 🌐 1. External Recon & Enumeration Principles

> [!info]  
> **External reconnaissance** is the process of gathering information about a target before moving into active enumeration and exploitation.
> 
> The objective is to understand the target's externally visible infrastructure, validate scope, discover public information leaks, identify users and technologies, and potentially uncover credentials that can lead to an initial foothold.

---

# ⚡ 2. External Recon Command Cheat Sheet

> [!important]  
> **Quick-reference commands used in this section.**
> 
> These are the commands/dorks worth remembering for CPTS. The rest of the workflow is largely performed through web-based OSINT resources.

## 🔎 2.1 DNS / Host Enumeration

### `nslookup`

```bash
nslookup ns1.inlanefreight.com
```

```bash
nslookup ns2.inlanefreight.com
```

**Purpose:** Resolve a hostname to an IP address and validate DNS information discovered through other sources.

---

## 📄 2.2 Search for Public PDF Files

```text
filetype:pdf inurl:inlanefreight.com
```

**Purpose:** Search-engine dork for PDF files indexed under the target domain.

Useful variations:

```text
filetype:pdf inurl:target.com
filetype:docx inurl:target.com
filetype:xlsx inurl:target.com
filetype:pptx inurl:target.com
```

> [!tip]  
> The important idea is not memorizing every Google dork. Understand the pattern:
> 
> ```text
> filetype:<extension> inurl:<target>
> ```

---

## 📧 2.3 Search for Email Addresses

```text
intext:"@inlanefreight.com" inurl:inlanefreight.com
```

**Purpose:** Search indexed pages for references to the target's email domain.

Mental model:

```text
TARGET DOMAIN
      ↓
SEARCH FOR @domain
      ↓
EMPLOYEE NAMES
      ↓
EMAIL FORMAT
      ↓
USERNAME FORMAT
```

---

## 🔑 2.4 DeHashed Example

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```

**Purpose:** Query breach data for information associated with the target domain.

Example output:

```text
id : 5996447501
email : roger.grimes@inlanefreight.local
username : rgrimes
password : Ilovefishing!
hashed_password : 
name : Roger Grimes
vin : 
address : 
phone : 
database_name : ModBSolutions

id : 7344467234
email : jane.yu@inlanefreight.local
username : jyu
password : Starlight1982_!
hashed_password : 
name : Jane Yu
vin : 
address : 
phone : 
database_name : MyFitnessPal

<SNIP>
```

> [!warning]  
> The credentials shown above are **fictional training data**. Real breach credentials must only be tested against systems explicitly authorized by the engagement.

---

## 🧰 2.5 Tools & Websites Mentioned

> [!info]  
> The following resources are referenced throughout this section. Use the official/current websites where possible.

|Resource|Primary Use|
|---|---|
|**Hurricane Electric BGP Toolkit**|ASN, BGP prefixes, netblocks, DNS and IP-space research|
|**IANA**|Internet number/resource information|
|**ARIN**|IP/ASN registration information for the Americas|
|**RIPE**|IP/ASN registration information for Europe and surrounding regions|
|**DomainTools**|Domain, DNS, WHOIS and historical infrastructure intelligence|
|**PTRArchive**|Historical/reverse DNS information|
|**ICANN Lookup**|Domain registration information|
|**ViewDNS**|Reverse IP, DNS, WHOIS and other DNS-related OSINT|
|**LinkedIn**|Employee and organizational information|
|**GitHub**|Public source code and accidental credential exposure|
|**GreyHat Warfare**|Public cloud-storage/bucket discovery|
|**Have I Been Pwned**|Checking whether email addresses appear in known breaches|
|**DeHashed**|Breach-data searches for emails, usernames, passwords and hashes|
|**linkedin2username**|Generate potential username formats from LinkedIn information|
|**TruffleHog**|Search repositories and other sources for exposed secrets|
|**Google / Search Engine Dorks**|Locate indexed files, pages and information|

---

# 🎯 3. Why External Recon Matters

Before kicking off a pentest, external reconnaissance can help us:

- ✅ Validate information provided in the client's **scoping document**
    
- 🎯 Ensure we are attacking the **correct infrastructure**
    
- 🌐 Discover publicly accessible infrastructure
    
- 🔑 Find leaked credentials
    
- 👤 Identify username/email naming conventions
    
- 🏢 Understand the organization's infrastructure
    
- ☁️ Identify cloud and third-party hosting
    
- 📄 Discover publicly accessible documents
    
- 💻 Find credentials accidentally committed to public repositories
    
- 🔎 Discover internal URLs, hostnames, technologies and services
    

Think of it as:

```text
                    🎯 TARGET
                       │
                       ▼
                🌎 "LAY OF THE LAND"
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       IP SPACE       DNS        PUBLIC DATA
          │            │            │
          ▼            ▼            ▼
      NETBLOCKS      HOSTS       USERS
          │            │            │
          └────────────┼────────────┘
                       ▼
                  🔑 CREDENTIALS
                       │
                       ▼
                  🎯 FOOTHOLD
```

---

# 🔎 4. What Are We Looking For?

|**Data PointDescription**||
|---|---|
|`IP Space`|Valid ASN for our target, netblocks in use for the organization's public-facing infrastructure, cloud presence and the hosting providers, DNS record entries, etc.|
|`Domain Information`|Based on IP data, DNS, and site registrations. Who administers the domain? Are there any subdomains tied to our target? Are there any publicly accessible domain services present? (Mailservers, DNS, Websites, VPN portals, etc.) Can we determine what kind of defenses are in place? (SIEM, AV, IPS/IDS in use, etc.)|
|`Schema Format`|Can we discover the organization's email accounts, AD usernames, and even password policies? Anything that will give us information we can use to build a valid username list to test external-facing services for password spraying, credential stuffing, brute forcing, etc.|
|`Data Disclosures`|For data disclosures we will be looking for publicly accessible files ( .pdf, .ppt, .docx, .xlsx, etc. ) for any information that helps shed light on the target. For example, any published files that contain `intranet` site listings, user metadata, shares, or other critical software or hardware in the environment (credentials pushed to a public GitHub repo, the internal AD username format in the metadata of a PDF, for example.)|
|`Breach Data`|Any publicly released usernames, passwords, or other critical information that can help an attacker gain a foothold.|

---

# 🗺️ 5. Where Are We Looking?

|**ResourceExamples**||
|---|---|
|`ASN / IP registrars`|[IANA](https://www.iana.org/), [arin](https://www.arin.net/) for searching the Americas, [RIPE](https://www.ripe.net/) for searching in Europe, [BGP Toolkit](https://bgp.he.net/)|
|`Domain Registrars & DNS`|[Domaintools](https://www.domaintools.com/), [PTRArchive](http://ptrarchive.com/), [ICANN](https://lookup.icann.org/lookup), manual DNS record requests against the domain in question or against well known DNS servers, such as `8.8.8.8`.|
|`Social Media`|Searching Linkedin, Twitter, Facebook, your region's major social media sites, news articles, and any relevant info you can find about the organization.|
|`Public-Facing Company Websites`|Often, the public website for a corporation will have relevant info embedded. News articles, embedded documents, and the "About Us" and "Contact Us" pages can also be gold mines.|
|`Cloud & Dev Storage Spaces`|[GitHub](https://github.com/), [AWS S3 buckets & Azure Blog storage containers](https://grayhatwarfare.com/), [Google searches using "Dorks"](https://www.exploit-db.com/google-hacking-database)|
|`Breach Data Sources`|[HaveIBeenPwned](https://haveibeenpwned.com/) to determine if any corporate email accounts appear in public breach data, [Dehashed](https://www.dehashed.com/) to search for corporate emails with cleartext passwords or hashes we can try to crack offline. We can then try these passwords against any exposed login portals (Citrix, RDS, OWA, 0365, VPN, VMware Horizon, custom applications, etc.) that may use AD authentication.|

---

# 🌐 6. Finding Address Spaces

The **Hurricane Electric BGP Toolkit** is useful for identifying:

- ASN
    
- Netblocks
    
- IP ranges
    
- Public infrastructure
    
- DNS information
    
- Hosting relationships
    

The basic workflow:

```text
DOMAIN / IP
     ↓
BGP LOOKUP
     ↓
ASN
     ↓
PREFIXES / NETBLOCKS
     ↓
PUBLIC IP SPACE
     ↓
HOSTS / SERVICES
```

Large organizations may have their own ASN and self-host much of their infrastructure.

Smaller organizations commonly use infrastructure belonging to:

- Cloudflare
    
- Google Cloud
    
- AWS
    
- Azure
    
- Other hosting providers
    

## ⚠️ 6.1 Scope Consideration

```text
                 CUSTOMER
                    │
                    ▼
             CLOUD / HOSTING
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   CUSTOMER       OTHER       PROVIDER
   SYSTEMS       CUSTOMER     SYSTEMS
```

Finding an IP associated with a customer does **not automatically authorize testing it**.

> [!warning]  
> You have authorization from the customer, not automatically from other customers or the hosting provider.
> 
> Some providers require additional authorization or have specific penetration-testing policies.

If scope is unclear:

```text
DISCOVER HOST
     ↓
CHECK SCOPE
     ↓
AUTHORIZED?
  ↙       ↘
YES        NO
 ↓          ↓
TEST      STOP
           ↓
       ESCALATE /
       CLARIFY
```

---

# 🌐 7. DNS Enumeration

DNS can validate scope and reveal hosts that were not included in the client's original scoping document.

Look for:

- A records
    
- AAAA records
    
- MX records
    
- NS records
    
- SOA information
    
- Subdomains
    
- Mail servers
    
- DNS servers
    
- VPN portals
    
- Other public services
    

Useful resources include **DomainTools**, **ViewDNS**, and direct DNS queries.

## 🔎 7.1 DNS Mental Model

```text
TARGET DOMAIN
      │
      ├── A / AAAA ──→ WEB HOST
      │
      ├── MX ────────→ MAIL SERVER
      │
      ├── NS ────────→ DNS SERVER
      │
      └── SUBDOMAINS ─→ ADDITIONAL HOSTS
```

---

# 💻 8. `nslookup` Validation
### 🧰 Common `nslookup` Record Types

| Record Type | Command                                  | What It Can Reveal                                  |
| ----------- | ---------------------------------------- | --------------------------------------------------- |
| **A**       | `nslookup -type=A inlanefreight.com`     | IPv4 address of the domain                          |
| **AAAA**    | `nslookup -type=AAAA inlanefreight.com`  | IPv6 address                                        |
| **MX**      | `nslookup -type=MX inlanefreight.com`    | Mail servers                                        |
| **NS**      | `nslookup -type=NS inlanefreight.com`    | Authoritative name servers                          |
| **TXT**     | `nslookup -type=TXT inlanefreight.com`   | SPF, verification, security-related text            |
| **CNAME**   | `nslookup -type=CNAME inlanefreight.com` | Canonical/alias target                              |
| **SOA**     | `nslookup -type=SOA inlanefreight.com`   | Zone authority and DNS administration information   |
| **PTR**     | `nslookup -type=PTR 134.209.24.248`      | Reverse DNS hostname                                |
| **ANY**     | `nslookup -type=ANY inlanefreight.com`   | Attempts to request multiple available record types |
After discovering:

```text
ns1.inlanefreight.com
ns2.inlanefreight.com
```

we can independently resolve them.

### NS1

```bash
nslookup ns1.inlanefreight.com
```

Output:

```text
Server:     192.168.186.1
Address:    192.168.186.1#53

Non-authoritative answer:
Name:   ns1.inlanefreight.com
Address: 178.128.39.165
```

### NS2

```bash
nslookup ns2.inlanefreight.com
```

Output:

```text
Server:     192.168.86.1
Address:    192.168.86.1#53

Non-authoritative answer:
Name:   ns2.inlanefreight.com
Address: 206.189.119.186
```

This gives us:

```text
ns1.inlanefreight.com → 178.128.39.165
ns2.inlanefreight.com → 206.189.119.186
```

> [!important]  
> These newly discovered IPs should be added to the investigation notes, but **must be checked against scope before scanning or attacking them**.

---

# 📄 9. Public Data & OSINT

Public information can reveal:

- Organization structure
    
- Employees
    
- Technologies
    
- Security products
    
- Software versions
    
- Hardware
    
- Cloud platforms
    
- Email formats
    
- Internal infrastructure
    
- Potential credentials
    

Important sources:

- LinkedIn
    
- Twitter
    
- Facebook
    
- Instagram
    
- Indeed
    
- Glassdoor
    
- Company websites
    
- Job advertisements
    
- News articles
    
- Public documents
    
- GitHub
    

---

# 💼 10. Job Postings as Technology Recon

A job posting can reveal technologies that the company uses.

For example, a **SharePoint Administrator** posting may expose:

- SharePoint
    
- SharePoint versions
    
- Azure
    
- Enterprise applications
    
- Security programs
    
- Backup systems
    
- Disaster recovery
    

If the company mentions:

```text
SharePoint 2013
SharePoint 2016
```

we may infer that multiple versions could exist.

That matters because:

```text
PUBLIC JOB POSTING
       ↓
TECHNOLOGY
       ↓
VERSION
       ↓
POTENTIAL ATTACK SURFACE
```

> [!tip]  
> Job postings are often overlooked because they do not look like traditional security information. For a penetration tester, they can provide valuable **technology fingerprinting**.

---

# 🌐 11. Public-Facing Websites

Company websites may expose:

- Contact emails
    
- Phone numbers
    
- Employee names
    
- Organizational charts
    
- Documents
    
- Internal URLs
    
- Intranet references
    
- Technology information
    

Public documents deserve special attention because they may contain metadata or references that were never intended for external users.

```text
PUBLIC WEBSITE
      ↓
DOCUMENT
      ↓
METADATA / LINKS
      ↓
EMPLOYEE / HOSTNAME / URL
      ↓
INTERNAL ENVIRONMENT CLUES
```

---

# 📄 12. Hunting for Files

Example Google dork:

```text
filetype:pdf inurl:inlanefreight.com
```

The example returned:

```text
corporate goals and strategy - Inlanefreight
```

When discovering useful files:

```text
DISCOVER
   ↓
RECORD URL
   ↓
DOWNLOAD
   ↓
SAVE LOCALLY
   ↓
ANALYZE CONTENT
   ↓
ANALYZE METADATA
   ↓
EXTRACT INTELLIGENCE
```

Look for:

- Internal URLs
    
- Intranet names
    
- Usernames
    
- Employee names
    
- Network shares
    
- Software
    
- Hardware
    
- Credentials
    
- Organizational information
    

> [!important]  
> Save documents, screenshots, scan results and tool output as soon as you encounter them. Good assessment documentation prevents important evidence from being lost later.

---

# 📧 13. Hunting Email Addresses

Example:

```text
intext:"@inlanefreight.com" inurl:inlanefreight.com
```

This discovered a contact page containing employees and their email addresses.

The apparent naming convention was:

```text
first.last@inlanefreight.com
```

This provides an important transition:

```text
EMPLOYEE NAME
      ↓
EMAIL ADDRESS
      ↓
EMAIL SCHEMA
      ↓
USERNAME SCHEMA
      ↓
USERNAME LIST
```

Potential usernames can then be used for authorized authentication testing such as password spraying.

---

# 👤 14. Username Harvesting

The source mentions **linkedin2username**.

The idea is to take employee information and generate possible username combinations:

```text
flast
first.last
f.last
```

For example:

```text
John Smith
   ↓
jsmith
john.smith
j.smith
```

> [!warning]  
> These are **candidate usernames**, not proof that accounts exist.
> 
> Authentication testing must remain within the engagement's scope and rules of engagement.

---

# 🔑 15. Credential Hunting

**DeHashed** can be used to search breach data for:

- Email addresses
    
- Usernames
    
- Cleartext passwords
    
- Password hashes
    
- Other leaked information
    

Example:

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```

The source notes that many credentials discovered through breach data will be old.

However, occasionally a leaked password may still work against an externally exposed service using AD authentication.

Potential services include:

```text
Citrix
RDS
OWA
Microsoft 365
VPN
VMware Horizon
Custom Applications
```

Mental model:

```text
BREACH DATA
     ↓
USERNAME
     ↓
PASSWORD / HASH
     ↓
CHECK AGE / RELEVANCE
     ↓
AUTHORIZED EXTERNAL SERVICE
     ↓
VALIDATE CREDENTIAL
     ↓
POTENTIAL FOOTHOLD
```

---

# 🧪 16. Example — `inlanefreight.com`

The example intentionally avoids heavy scanning such as:

```text
Nmap
Vulnerability Scanners
```

The objective is to demonstrate **passive enumeration**.

---

## 🌐 16.1 BGP Information

BGP information revealed:

```text
IP Address:
134.209.24.248

Mail Server:
mail1.inlanefreight.com

Nameservers:
NS1.inlanefreight.com
NS2.inlanefreight.com
```

---

## 🔎 16.2 ViewDNS Validation

Reverse-IP results returned:

```text
inlanefreight.com
lycjg.us
```

The information matched the previously discovered IP data.

> [!tip]  
> **Cross-validation matters.**
> 
> When multiple independent sources agree, confidence in your reconnaissance data increases.

---

## 🌐 16.3 Nameserver Validation

```bash
nslookup ns1.inlanefreight.com
```

→ `178.128.39.165`

```bash
nslookup ns2.inlanefreight.com
```

→ `206.189.119.186`

The IPs are recorded for further investigation, subject to scope.

---

## 📄 16.4 File Discovery

```text
filetype:pdf inurl:inlanefreight.com
```

Discovered:

```text
corporate goals and strategy - Inlanefreight
```

---

## 📧 16.5 Email Discovery

```text
intext:"@inlanefreight.com" inurl:inlanefreight.com
```

Discovered employee contact information and the apparent:

```text
first.last
```

naming convention.

---

## 👤 16.6 Username Generation

Use **linkedin2username** to generate candidate formats such as:

```text
flast
first.last
f.last
```

---

## 🔑 16.7 Breach Data

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```

Potential output:

```text
email
username
password
hashed_password
name
database_name
```

---

# 🧠 17. The Complete External Recon Workflow

```text
                    🎯 TARGET
                       │
                       ▼
                📜 SCOPE / ROE
                       │
                       ▼
                🌎 PASSIVE RECON
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
    🌐 ASN/IP         🌐 DNS          📱 OSINT
       │               │               │
       ▼               ▼               ▼
   NETBLOCKS         HOSTS          EMPLOYEES
   PROVIDERS         MX/NS          TECHNOLOGY
   CLOUD             SUBDOMAINS     JOB POSTS
       │               │               │
       └───────────────┼───────────────┘
                       ▼
                  📄 DOCUMENTS
                       │
                       ▼
                👤 USERNAMES
                       │
                       ▼
                 🔑 BREACH DATA
                       │
                       ▼
                 🧠 CORRELATION
                       │
                       ▼
               🎯 TARGETED TESTING
                       │
                       ▼
              🔎 ACTIVE ENUMERATION
                       │
                       ▼
                🆕 NEW DISCOVERY
                       │
                       └─────────────→ 🔄 REPEAT
```

---

# 🔄 18. Enumeration Is Iterative

Never think of enumeration as:

```text
Recon → Finished → Attack
```

Instead:

```text
Recon
 ↓
Discovery
 ↓
Analysis
 ↓
New Lead
 ↓
Recon Again
 ↓
New Discovery
 ↓
Targeted Enumeration
 ↓
New Lead
 ↓
Repeat
```

Example:

```text
📄 PUBLIC PDF
      ↓
👤 EMPLOYEE NAME
      ↓
📧 EMAIL FORMAT
      ↓
👤 USERNAME
      ↓
🔑 BREACH DATA
      ↓
🔐 CREDENTIAL
      ↓
🌐 EXTERNAL LOGIN
      ↓
🎯 AUTHORIZED TEST
      ↓
🔓 INITIAL ACCESS
      ↓
🏢 INTERNAL ENUMERATION
```

---

# ⚠️ 19. Scope Decision Tree

```text
                    🔎 DISCOVERY
                         │
                         ▼
                 Is it related to
                   the customer?
                    /       \
                  NO         YES
                  │           │
                 STOP         ▼
                       Is it in scope?
                         /       \
                       NO         YES
                       │           │
                      STOP         ▼
                           Does the ROE
                           permit testing?
                             /      \
                           NO        YES
                           │          │
                          STOP        ▼
                                  🎯 TEST
```

> [!important]  
> **Discovery does not equal authorization.**
> 
> This is one of the most important real-world penetration-testing principles in the entire reconnaissance phase.

---

# 🧠 20. Mental Model — External Recon as an Information Funnel

```text
                    🌎 EVERYTHING
                         │
                         ▼
                    ORGANIZATION
                         │
                         ▼
                   🌐 IP / ASN
                         │
                         ▼
                     🌍 DNS
                         │
                         ▼
                    🖥️ HOSTS
                         │
                         ▼
               📄 DOCUMENTS / DATA
                         │
                         ▼
                    👤 USERS
                         │
                         ▼
                   🔑 CREDENTIALS
                         │
                         ▼
                 🎯 VALID TARGET
                         │
                         ▼
                    🔓 FOOTHOLD
```

> [!success]  
> The goal is **not** to collect random information.
> 
> The goal is to connect individual discoveries until they form an **attack-relevant picture of the environment**.

---

# 📚 21. CPTS External Recon Cheat Sheet

|Stage|Question|Examples|
|---|---|---|
|🌐 IP Space|Where does the target's infrastructure live?|ASN, netblocks, cloud providers|
|🌎 DNS|What hosts/services exist?|A, AAAA, MX, NS, subdomains|
|🏢 Organization|How is the company structured?|Employees, offices, departments|
|💼 Technology|What technologies are used?|SharePoint, Azure, enterprise applications|
|📄 Documents|What has been publicly exposed?|PDF, DOCX, XLSX, PPTX|
|👤 Identity|How are users named?|`first.last`, `flast`, `f.last`|
|💻 Development|Did developers leak anything?|GitHub, code, configuration|
|☁️ Cloud|Is storage publicly exposed?|S3, Azure storage|
|🔑 Breach Data|Are credentials publicly available?|Passwords, hashes, usernames|
|🎯 Validation|Can the information be used?|Authorized authentication testing|
|🔄 Iteration|What did the discovery reveal next?|Return to recon and enumerate again|

---

# 🧠 22. Final CPTS Mental Model

```text
             🌐 EXTERNAL RECON
                     │
                     ▼
          🔎 FIND INFORMATION
                     │
                     ▼
          🧬 CORRELATE INFORMATION
                     │
                     ▼
           👤 IDENTIFY USERS
                     │
                     ▼
         🔑 IDENTIFY CREDENTIALS
                     │
                     ▼
          🎯 VALIDATE SCOPE
                     │
                     ▼
       🔐 TEST AUTHORIZED SERVICES
                     │
                     ▼
             🔓 FOOTHOLD
                     │
                     ▼
          🏢 INTERNAL ENUMERATION
                     │
                     ▼
             🔄 REPEAT CYCLE
```

> [!success]
> 
> ### 🎯 Key Takeaways
> 
> - **Start passive and go wide.**
>     
> - Use **ASN/IP → DNS → OSINT → documents → users → breach data** as a natural progression.
>     
> - Use `nslookup` to validate DNS discoveries.
>     
> - Use search-engine dorks such as:
>     
>     ```text
>     filetype:pdf inurl:target.com
>     intext:"@target.com" inurl:target.com
>     ```
>     
> - Public documents can reveal **metadata, usernames, internal URLs, software and infrastructure**.
>     
> - Job postings can fingerprint **technologies and versions**.
>     
> - Email naming conventions can become **username-generation rules**.
>     
> - Breach data can provide potential **passwords, hashes and usernames**.
>     
> - Always **cross-reference discoveries** instead of trusting one source.
>     
> - Enumeration is **iterative**: every discovery can create another enumeration path.
>     
> - **Scope and authorization override everything else.**
>     
> 
> ```text
> 🌐 RECON
>   ↓
> 🔎 ENUMERATE
>   ↓
> 🧠 CORRELATE
>   ↓
> 👤 IDENTIFY
>   ↓
> 🔑 CREDENTIALS
>   ↓
> 🎯 VALIDATE SCOPE
>   ↓
> 🔓 ACCESS
>   ↓
> 🏢 INTERNAL ENUMERATION
>   ↓
> 🔄 REPEAT
> ```