# 🌐 1. External Recon & Enumeration Principles

> [!info]  
> **External reconnaissance** is the process of gathering information about a target **before actively attacking or scanning it**.
> 
> The goal is to build a reliable picture of the organization's externally visible infrastructure, identify information leaks, validate the assessment scope, and discover potential paths toward an initial foothold.

## 🧠 1.1 Why External Recon Matters

Before starting a pentest, external reconnaissance can help us:

- ✅ Validate information provided in the client's **scoping document**
    
- 🎯 Ensure we are interacting with the **correct infrastructure**
    
- 🌐 Identify publicly exposed infrastructure the client did not mention
    
- 🔑 Discover **leaked credentials**
    
- 👤 Determine username/email naming conventions
    
- 🏢 Understand the organization's infrastructure and technologies
    
- ☁️ Identify cloud providers and third-party hosting
    
- 📄 Find publicly accessible documents containing sensitive metadata
    
- 💻 Discover leaked credentials or secrets in public repositories
    
- 🔎 Identify external services that may eventually provide a route into the environment
    

> [!important]  
> Think of external recon as **getting the lay of the land**.
> 
> You are trying to understand:
> 
> ```text
> WHO are we attacking?
>        ↓
> WHERE is their infrastructure?
>        ↓
> WHAT technologies/services do they expose?
>        ↓
> WHAT information has leaked publicly?
>        ↓
> HOW is the organization structured?
>        ↓
> CAN any of this help us gain access?
> ```

A simple leak such as a username format from a company website can later become extremely valuable when constructing a password-spraying target list.

More advanced discoveries may include:

- Credentials pushed to GitHub
    
- Internal URLs inside public documents
    
- Intranet references
    
- AD username formats
    
- Publicly exposed cloud storage
    
- Password breach data
    
- Information about internal hardware/software
    
- Security products and infrastructure
    

---

# 🔎 2. What Are We Looking For?

During external reconnaissance, we want to collect several categories of information.

|**Data PointDescription**||
|---|---|
|`IP Space`|Valid ASN for our target, netblocks in use for the organization's public-facing infrastructure, cloud presence and the hosting providers, DNS record entries, etc.|
|`Domain Information`|Based on IP data, DNS, and site registrations. Who administers the domain? Are there any subdomains tied to our target? Are there any publicly accessible domain services present? (Mailservers, DNS, Websites, VPN portals, etc.) Can we determine what kind of defenses are in place? (SIEM, AV, IPS/IDS in use, etc.)|
|`Schema Format`|Can we discover the organization's email accounts, AD usernames, and even password policies? Anything that will give us information we can use to build a valid username list to test external-facing services for password spraying, credential stuffing, brute forcing, etc.|
|`Data Disclosures`|For data disclosures we will be looking for publicly accessible files ( .pdf, .ppt, .docx, .xlsx, etc. ) for any information that helps shed light on the target. For example, any published files that contain `intranet` site listings, user metadata, shares, or other critical software or hardware in the environment (credentials pushed to a public GitHub repo, the internal AD username format in the metadata of a PDF, for example.)|
|`Breach Data`|Any publicly released usernames, passwords, or other critical information that can help an attacker gain a foothold.|

## 🧩 2.1 The Five Major Recon Categories

```text
🌐 IP SPACE
   └── ASN / Netblocks / Cloud / Hosting

🌎 DOMAIN INFORMATION
   └── DNS / Subdomains / Mail / VPN / Websites / Defenses

👤 SCHEMA FORMAT
   └── Emails / Usernames / AD Naming / Password Policy Clues

📄 DATA DISCLOSURES
   └── PDFs / Office Docs / GitHub / Metadata / Internal URLs

🔑 BREACH DATA
   └── Usernames / Passwords / Hashes / Previous Leaks
```

> [!tip]  
> If you get stuck during a pentest, **go back to passive reconnaissance**. A forgotten public document, username format, or breach credential may provide the missing piece needed to move forward.

---

# 🗺️ 3. Where Are We Looking?

Different sources can provide different pieces of the target picture.

|**ResourceExamples**||
|---|---|
|`ASN / IP registrars`|[IANA](https://www.iana.org/), [arin](https://www.arin.net/) for searching the Americas, [RIPE](https://www.ripe.net/) for searching in Europe, [BGP Toolkit](https://bgp.he.net/)|
|`Domain Registrars & DNS`|[Domaintools](https://www.domaintools.com/), [PTRArchive](http://ptrarchive.com/), [ICANN](https://lookup.icann.org/lookup), manual DNS record requests against the domain in question or against well known DNS servers, such as `8.8.8.8`.|
|`Social Media`|Searching Linkedin, Twitter, Facebook, your region's major social media sites, news articles, and any relevant info you can find about the organization.|
|`Public-Facing Company Websites`|Often, the public website for a corporation will have relevant info embedded. News articles, embedded documents, and the "About Us" and "Contact Us" pages can also be gold mines.|
|`Cloud & Dev Storage Spaces`|[GitHub](https://github.com/), [AWS S3 buckets & Azure Blog storage containers](https://grayhatwarfare.com/), [Google searches using "Dorks"](https://www.exploit-db.com/google-hacking-database)|
|`Breach Data Sources`|[HaveIBeenPwned](https://haveibeenpwned.com/) to determine if any corporate email accounts appear in public breach data, [Dehashed](https://www.dehashed.com/) to search for corporate emails with cleartext passwords or hashes we can try to crack offline. We can then try these passwords against any exposed login portals (Citrix, RDS, OWA, 0365, VPN, VMware Horizon, custom applications, etc.) that may use AD authentication.|

---

# 🌐 4. Finding Address Spaces

## 🛰️ 4.1 ASN & BGP Enumeration

The **Hurricane Electric BGP Toolkit** is useful for researching:

- IP address space
    
- Netblocks
    
- Autonomous System Numbers (**ASN**)
    
- Public infrastructure
    
- Relationships between IPs and organizations
    

The basic methodology is:

```text
DOMAIN / IP
    ↓
BGP / ASN LOOKUP
    ↓
ASN
    ↓
NETBLOCKS
    ↓
PUBLIC IP SPACE
    ↓
HOSTS / DNS / SERVICES
```

Large organizations frequently have their own ASN and self-host significant portions of their infrastructure.

Smaller organizations may instead use infrastructure belonging to:

- Cloudflare
    
- Google Cloud
    
- AWS
    
- Azure
    
- Other hosting providers
    

### ⚠️ Scope Is More Important Than Discovery

Discovering an IP address **does not automatically mean you are authorized to attack it**.

For example:

```text
Customer
   │
   └── Shared hosting / cloud provider
          │
          ├── Customer infrastructure
          │
          ├── Other customer's infrastructure
          │
          └── Provider infrastructure
```

A penetration tester has authorization from the **customer**, not automatically from the other organizations sharing the provider's infrastructure.

> [!warning]  
> **Never assume that an interesting IP is in scope simply because it belongs to infrastructure associated with the target.**
> 
> Before interacting with newly discovered infrastructure, verify the scope.

Some hosting providers may require additional authorization.

Examples mentioned in the source:

- AWS has specific penetration-testing guidelines and allows testing of some services without prior approval.
    
- Oracle may require a **Cloud Security Testing Notification**.
    
- Other third-party providers may require written authorization.
    

These issues should be handled through:

- Management
    
- Legal
    
- Contracts
    
- The client's authorization
    
- The assessment's rules of engagement
    

> [!important]  
> When in doubt, **stop and clarify the scope in writing before attacking the host**.

---

# 🌐 5. DNS Enumeration

DNS is one of the most useful sources of external information because it can reveal hosts that were not explicitly included in the original scoping document.

Potentially useful information includes:

- A / AAAA records
    
- MX records
    
- NS records
    
- SOA records
    
- Subdomains
    
- Mail servers
    
- DNS servers
    
- Additional externally reachable infrastructure
    

Useful resources include:

- `domaintools`
    
- `viewdns.info`
    
- Manual DNS queries
    
- Public DNS servers such as `8.8.8.8`
    

## 🔎 5.1 DNS as Scope Validation

DNS can reveal:

```text
Target Domain
      │
      ├── Web Server
      ├── Mail Server
      ├── DNS Server
      ├── VPN Portal
      ├── Subdomains
      └── Other Hosts
```

Some discovered hosts may be out of scope.

However, if an interesting subdomain resolves to an **in-scope IP**, it may potentially be relevant to the assessment.

> [!tip]  
> Do not automatically attack every interesting result. Build the list first, then compare it against the **Rules of Engagement / scope**.

---

# 🌍 6. Public Data & OSINT

Public information can reveal much more than simple contact details.

Potential sources include:

- LinkedIn
    
- Twitter
    
- Facebook
    
- Instagram
    
- Indeed
    
- Glassdoor
    
- Company websites
    
- News articles
    
- Job postings
    
- Public documents
    
- GitHub
    
- Cloud storage
    

## 💼 6.1 Job Postings

Job postings can reveal:

- Technologies in use
    
- Versions
    
- Infrastructure architecture
    
- Security products
    
- Backup solutions
    
- Disaster recovery systems
    
- Cloud platforms
    
- Enterprise applications
    
- Organizational structure
    

### Example: SharePoint Administrator

A SharePoint administrator job posting can reveal that the company uses SharePoint and may provide clues about:

- SharePoint versions
    
- Azure
    
- Enterprise applications
    
- Security programs
    
- Backup and disaster recovery
    

If the organization mentions **SharePoint 2013 and SharePoint 2016**, this may suggest multiple versions are present.

That can be important because older versions may contain vulnerabilities that do not exist in newer versions.

> [!important]  
> **Public information is intelligence.**
> 
> A job advertisement that looks harmless to the company may give a penetration tester a technology/version fingerprint.

---

# 🌐 7. Public-Facing Websites

The organization's own website can reveal:

- Contact emails
    
- Phone numbers
    
- Organizational information
    
- Employee names
    
- Organizational charts
    
- Published documents
    
- Internal URLs
    
- Intranet references
    
- Technology information
    
- Links to external services
    

Documents embedded on public websites deserve particular attention.

```text
PUBLIC WEBSITE
      ↓
DOCUMENTS
      ↓
METADATA / LINKS / NAMES
      ↓
INTERNAL INFRASTRUCTURE CLUES
      ↓
USERNAMES / HOSTNAMES / APPLICATIONS
```

---

# 📄 8. Hunting for Publicly Accessible Files

Search engines can be used to locate publicly indexed files.

Example:

```text
filetype:pdf inurl:inlanefreight.com
```

This searches for PDF documents associated with the target domain.

The source example discovered a PDF titled:

```text
corporate goals and strategy - Inlanefreight
```

### Why Save Discovered Files?

When performing recon, save:

- Documents
    
- Screenshots
    
- Scan output
    
- Tool output
    
- URLs
    
- Interesting search results
    

> [!tip]  
> **Document everything as you discover it.**
> 
> Saving evidence immediately prevents losing the location of important information later and helps maintain a comprehensive assessment record.

---

# 📧 9. Hunting Email Addresses

A Google dork can search for references to the organization's email domain.

Example:

```text
intext:"@inlanefreight.com" inurl:inlanefreight.com
```

The purpose is to locate pages containing email addresses associated with the target.

In the example, a contact page exposed multiple employees and their contact information.

This allowed us to identify an apparent naming convention:

```text
first.last@inlanefreight.com
```

For example:

```text
emma.williams@inlanefreight.com
john.smith@inlanefreight.com
david.jones@inlanefreight.com
```

This can later help construct usernames for:

- Password spraying
    
- Credential testing
    
- Credential stuffing
    
- Social engineering/phishing, **if explicitly in scope**
    

> [!important]  
> The valuable discovery is not just the email address itself.
> 
> The real intelligence is the **schema**:
> 
> ```text
> Employee Name
>       ↓
> Email Format
>       ↓
> Username Format
>       ↓
> Candidate User List
>       ↓
> Authentication Testing
> ```

---

# 👤 10. Username Harvesting

A tool such as **linkedin2username** can scrape information from a company's LinkedIn presence and generate possible username formats.

Potential formats include:

```text
flast
first.last
f.last
firstl
```

The generated usernames can then become candidates for later authentication testing.

> [!warning]  
> Username generation only creates **potential identities**. It does not prove that an account exists or that a particular authentication attempt is authorized.
> 
> Any password spraying, brute forcing, or credential testing must remain within the assessment scope and rules of engagement.

---

# 🔑 11. Credential Hunting & Breach Data

Breach data can sometimes provide:

- Usernames
    
- Email addresses
    
- Cleartext passwords
    
- Password hashes
    
- Other leaked information
    

A source such as **DeHashed** can be used to search breach data.

The source notes that many discovered passwords will be old and unusable, but occasionally credentials may still work against externally exposed authentication portals.

Potential targets can include:

- Citrix
    
- RDS
    
- OWA
    
- Microsoft 365
    
- VPN portals
    
- VMware Horizon
    
- Custom applications
    
- Other services using AD authentication
    

> [!warning]  
> Breach credentials are **leads**, not guaranteed valid credentials. Treat them as potentially stale and handle them according to the engagement's authorization and credential-testing rules.

---

# 💻 12. DeHashed Example

Example command from the source:

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```

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

### 🧠 What This Gives the Tester

The result potentially gives:

```text
EMAIL
  ↓
USERNAME
  ↓
PASSWORD / HASH
  ↓
VALIDATE AGE / RELEVANCE
  ↓
CHECK AUTHORIZED EXTERNAL SERVICES
  ↓
POSSIBLE FOOTHOLD
```

The source notes that the example data is **fictional**.

The example script is:

```text
https://github.com/mrb3n813/Pentest-stuff/blob/master/dehashed.py
```

An alternative script mentioned is:

```text
https://github.com/sm00v/Dehashed
```

> [!warning]  
> The source notes that DeHashed's API structure has changed and modifications to the example script may be necessary. Before executing third-party scripts, understand what the script does and what data it sends or retrieves.

---

# 🧩 13. External Recon Methodology

The major lesson is that external enumeration is **iterative**, not a one-time checklist.

A useful mental model:

```text
                 🌐 TARGET
                    │
                    ▼
            🔎 PASSIVE RECON
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     ASN/IP        DNS        PUBLIC DATA
       │            │            │
       ▼            ▼            ▼
   NETBLOCKS     HOSTS       USERS / TECH
       │            │            │
       └────────────┼────────────┘
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
             🧠 BUILD TARGET
                KNOWLEDGE
                    │
                    ▼
             🎯 ACTIVE ENUM
                    │
                    ▼
              NEW DISCOVERY
                    │
                    └──────────────┐
                                   ▼
                            🔄 RE-EVALUATE
```

> [!important]  
> **Enumeration is iterative.**
> 
> A new discovery in one category should cause you to revisit other categories.

For example:

```text
DNS
 ↓
new hostname
 ↓
website discovered
 ↓
public document found
 ↓
employee name discovered
 ↓
username format identified
 ↓
breach database checked
 ↓
credential discovered
 ↓
authorized external service tested
 ↓
new access
 ↓
internal enumeration
```

---

# 🔬 14. Passive → Active Enumeration

The overall methodology begins **wide and passive**, then becomes increasingly targeted.

```text
🌎 START WIDE
     │
     ▼
🌐 ASN / IP / DOMAIN
     │
     ▼
📡 DNS / PUBLIC RECORDS
     │
     ▼
📱 SOCIAL MEDIA
     │
     ▼
📄 DOCUMENTS
     │
     ▼
👤 USERNAME / EMAIL SCHEMA
     │
     ▼
🔑 BREACH DATA
     │
     ▼
🧠 ANALYZE RESULTS
     │
     ▼
🎯 ACTIVE ENUMERATION
     │
     ▼
🔄 REPEAT
```

The principle is:

> **Start with the least intrusive sources and progressively increase interaction as the engagement allows.**

This minimizes unnecessary activity while maximizing intelligence gathered before active testing.

---

# 🧪 15. Example: External Enumeration of `inlanefreight.com`

The source uses `inlanefreight.com` as a fictitious target and explicitly avoids heavy scans such as:

- Nmap
    
- Vulnerability scanners
    

The purpose is to demonstrate passive enumeration methodology.

---

## 🌐 15.1 Step 1 — Check ASN / IP / Domain Data

The BGP Toolkit returned:

```text
IP Address:
134.209.24.248

Mail Server:
mail1.inlanefreight.com

Nameservers:
NS1.inlanefreight.com
NS2.inlanefreight.com
```

Because Inlanefreight is not a large corporation, it was not surprising that it did not have its own ASN.

### 🧠 Why This Matters

We now have several pieces of infrastructure to validate:

```text
inlanefreight.com
      │
      ├── 134.209.24.248
      │
      ├── mail1.inlanefreight.com
      │
      ├── ns1.inlanefreight.com
      │
      └── ns2.inlanefreight.com
```

---

# 🔎 16. Step 2 — Validate the IP with ViewDNS

The ViewDNS reverse-IP results showed:

```text
inlanefreight.com
lycjg.us
```

Both results matched the information previously obtained.

> [!tip]  
> **Do not trust a single data source blindly.**
> 
> Cross-validation increases confidence in your reconnaissance data.

---

# 🌐 17. Step 3 — Validate Nameservers with `nslookup`

The source used:

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

Then:

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

### 🔎 Important Discovery

We now have **two additional IP addresses**:

```text
ns1.inlanefreight.com → 178.128.39.165
ns2.inlanefreight.com → 206.189.119.186
```

These should be added to the investigation list.

However:

> [!warning]  
> **Discovery ≠ authorization.**
> 
> Before scanning or attacking these addresses, determine whether they are within scope.
> 
> In the source example, these IP addresses were **not in scope for scanning**, although passive browsing of websites could be performed to hunt for useful information.

---

# 📄 18. Step 4 — Hunt for Public Documents

Google dork:

```text
filetype:pdf inurl:inlanefreight.com
```

The search returned a PDF:

```text
corporate goals and strategy - Inlanefreight
```

### 🎯 Methodology

When you discover an interesting document:

```text
DISCOVER
   ↓
RECORD URL
   ↓
DOWNLOAD COPY
   ↓
SAVE SCREENSHOT / EVIDENCE
   ↓
ANALYZE CONTENT
   ↓
ANALYZE METADATA
   ↓
EXTRACT NAMES / HOSTNAMES / URLS / TECHNOLOGIES
```

Potential information can include:

- Employee names
    
- Usernames
    
- Internal hostnames
    
- Intranet URLs
    
- Network shares
    
- Software
    
- Hardware
    
- Organizational information
    
- Metadata
    

---

# 📧 19. Step 5 — Hunt for Email Addresses

Google dork:

```text
intext:"@inlanefreight.com" inurl:inlanefreight.com
```

This produced a contact page containing employee information.

From the results, the organization appeared to use:

```text
first.last@inlanefreight.com
```

### 🧠 Why This Matters

A naming convention can transform:

```text
Employee Names
```

into:

```text
Potential Usernames
```

which can later become:

```text
Password-Spraying Candidates
```

provided that such testing is explicitly authorized.

---

# 👤 20. Step 6 — Username Harvesting

The source recommends:

**linkedin2username**

It can generate username combinations from company employee information.

Examples:

```text
flast
first.last
f.last
```

This provides a systematic way to build candidate username lists instead of manually guessing formats.

---

# 🔐 21. Step 7 — Credential Hunting

DeHashed can be used to search breach data for:

```text
Corporate Email
      ↓
Username
      ↓
Cleartext Password
      ↓
Password Hash
```

Example:

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```

Potential results may contain:

```text
email
username
password
hashed_password
name
database_name
```

These credentials may be:

- Old
    
- Reused
    
- Invalid
    
- Valid
    
- Associated with unrelated services
    

Therefore, they must be treated as **intelligence requiring validation**, not automatically valid access.

---

# 🔄 22. The Complete External Recon Workflow

```text
                         🎯 TARGET
                            │
                            ▼
                  📜 SCOPE / ROE REVIEW
                            │
                            ▼
                    🌐 PASSIVE RECON
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          ASN/IP           DNS          PUBLIC DATA
             │              │              │
             ▼              ▼              ▼
        Netblocks        Hosts         Websites
        Providers       MX/NS/A       Social Media
        Cloud           Subdomains    Job Listings
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                     📄 DOCUMENTS
                            │
                            ▼
                  👤 EMAIL / USERNAMES
                            │
                            ▼
                     🔑 BREACH DATA
                            │
                            ▼
                    🧠 CORRELATE DATA
                            │
                            ▼
                    🎯 TARGETED TESTING
                            │
                            ▼
                    🔎 ACTIVE ENUMERATION
                            │
                            ▼
                    🆕 NEW INFORMATION
                            │
                            └─────────────► 🔄 REPEAT
```

---

# 🧠 23. Mental Model — Recon as an Information Funnel

The most important concept is that external enumeration starts **broad** and gradually becomes **specific**.

```text
                    🌎 EVERYTHING
                         │
                         ▼
                  🌐 ORGANIZATION
                         │
                         ▼
                  📡 IP / ASN SPACE
                         │
                         ▼
                    🌍 DOMAINS
                         │
                         ▼
                   🖥️ HOSTS
                         │
                         ▼
                📄 DOCUMENTS / DATA
                         │
                         ▼
                  👤 IDENTITIES
                         │
                         ▼
                 🔑 CREDENTIALS
                         │
                         ▼
                 🎯 VALID TARGETS
                         │
                         ▼
                    🔓 FOOTHOLD
```

Each discovery should answer another question about the target.

> **Recon is not about collecting random facts. It is about connecting facts until they form an attack-relevant picture of the environment.**

---

# 🔁 24. Enumeration Is Iterative

Do not think:

```text
Recon → Finished → Attack
```

Think:

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

For example:

```text
📄 Public PDF
      ↓
👤 Employee Name
      ↓
📧 Email Format
      ↓
👤 Username
      ↓
🔑 Breach Data
      ↓
🔐 Credential
      ↓
🌐 External Login Portal
      ↓
🎯 Authorized Authentication Test
      ↓
🔓 Potential Foothold
```

---

# ⚠️ 25. Scope Is Part of Enumeration

A major CPTS/real-world pentesting lesson is that **finding something does not mean attacking it**.

```text
Discovery
   ↓
Is it related to the client?
   ↓
Is it explicitly in scope?
   ↓
Does the ROE permit interaction?
   ↓
Is third-party authorization required?
   ↓
YES → Continue
NO  → STOP / Escalate / Clarify
```

Examples of potentially dangerous situations:

- Shared hosting
    
- Shared cloud infrastructure
    
- Third-party DNS
    
- CDN infrastructure
    
- Cloud provider systems
    
- Other customers sharing the same infrastructure
    

> [!important]  
> **Your authorization follows the scope, not merely the ownership relationship.**
> 
> If you discover infrastructure belonging to a hosting provider or another organization, do not assume it is yours to test.

---

# 🧠 26. CPTS Enumeration Philosophy

The source ultimately leads into internal Active Directory enumeration, but the same methodology applies throughout the engagement.

### External

```text
PASSIVE
  ↓
ASN / DNS / OSINT
  ↓
PUBLIC DATA
  ↓
IDENTITIES
  ↓
CREDENTIALS
```

### Internal

```text
LOW-PRIVILEGE ACCESS
  ↓
AD ENUMERATION
  ↓
USERS / GROUPS / COMPUTERS
  ↓
SERVICES / SHARES / ACLs
  ↓
ATTACK PATH
  ↓
PRIVILEGE ESCALATION
  ↓
LATERAL MOVEMENT
```

The two phases are connected:

```text
🌐 EXTERNAL RECON
       ↓
Potential User / Credential / Host
       ↓
🔓 INITIAL ACCESS
       ↓
🏢 INTERNAL ENUMERATION
       ↓
🧭 ATTACK PATH DISCOVERY
       ↓
👑 PRIVILEGE / OBJECTIVE
```

---

# 📚 27. Useful Resources Mentioned

### 🌐 ASN / IP

- IANA
    
- ARIN
    
- RIPE
    
- Hurricane Electric BGP Toolkit
    

### 🌎 Domain / DNS

- DomainTools
    
- PTRArchive
    
- ICANN Lookup
    
- ViewDNS
    
- Public DNS servers such as `8.8.8.8`
    

### 👥 Social / Public Information

- LinkedIn
    
- Twitter
    
- Facebook
    
- Instagram
    
- Indeed
    
- Glassdoor
    
- News articles
    

### 💻 Cloud / Development

- GitHub
    
- AWS S3
    
- Azure storage
    
- GreyHat Warfare
    
- Google search / Google Dorks
    

### 🔑 Breach Data

- Have I Been Pwned
    
- DeHashed
    

### 👤 Username Generation

- linkedin2username
    

### 🔐 Secret Hunting

- TruffleHog
    

---

# 📊 28. External Recon Cheat Sheet

|Category|What to Look For|Why It Matters|
|---|---|---|
|🌐 IP Space|ASN, netblocks, public IPs, cloud providers|Defines infrastructure and validates scope|
|🌎 DNS|A/AAAA, MX, NS, SOA, subdomains|Discovers hosts and services|
|👤 Schema|Email and username formats|Builds candidate user lists|
|📄 Documents|PDFs, DOCX, XLSX, PPTX|May expose metadata, internal URLs, users, technology|
|💼 Job Posts|Technologies, versions, infrastructure|Fingerprints the environment|
|📱 Social Media|Employees, technologies, locations|Provides organizational intelligence|
|💻 GitHub|Secrets, credentials, configuration|Can produce direct access|
|☁️ Cloud Storage|Public buckets/containers|May expose sensitive data|
|🔑 Breach Data|Passwords, hashes, usernames|May provide valid credentials|
|🛰️ Hosting|CDN/cloud/shared infrastructure|Critical for scope validation|

---

# ✅ 29. Key Takeaways

> [!success]
> 
> ### 🎯 CPTS-Ready Lessons
> 
> 1. **Start passive and go wide.**
>     
>     - ASN
>         
>     - IP space
>         
>     - DNS
>         
>     - Websites
>         
>     - Social media
>         
>     - Documents
>         
>     - Breach data
>         
> 2. **Validate everything.**
>     
>     - Do not rely on a single source.
>         
>     - Cross-reference DNS, BGP, ViewDNS, websites, and other sources.
>         
> 3. **Build an information chain.**
>     
>     ```text
>     DOMAIN
>       ↓
>     IP
>       ↓
>     HOST
>       ↓
>     DOCUMENT
>       ↓
>     EMPLOYEE
>       ↓
>     USERNAME
>       ↓
>     CREDENTIAL
>       ↓
>     ACCESS
>     ```
>     
> 4. **Public information can directly affect exploitation.**
>     
>     - Job postings reveal technologies.
>         
>     - Documents reveal internal information.
>         
>     - Websites reveal users.
>         
>     - GitHub can leak secrets.
>         
>     - Breach data can reveal credentials.
>         
> 5. **Enumeration is iterative.**
>     
>     - Every discovery can create another enumeration path.
>         
>     - Revisit previous sources as new information becomes available.
>         
> 6. **Scope always comes first.**
>     
>     - A discovered IP is not automatically an authorized target.
>         
>     - Shared infrastructure is particularly dangerous.
>         
>     - Escalate unclear scope before interacting with questionable infrastructure.
>         
> 7. **External recon prepares internal enumeration.**
>     
>     - External usernames and credentials can become the bridge into the internal AD environment.
>         
> 
> ```text
> 🌐 PASSIVE RECON
>       ↓
> 🔎 ENUMERATE
>       ↓
> 🧠 CORRELATE
>       ↓
> 👤 IDENTIFY USERS
>       ↓
> 🔑 IDENTIFY CREDENTIALS
>       ↓
> 🎯 VALIDATE IN-SCOPE SERVICES
>       ↓
> 🔓 INITIAL ACCESS
>       ↓
> 🏢 INTERNAL AD ENUMERATION
> ```