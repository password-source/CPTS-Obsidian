# 🌐 1. Attacking DNS

> [!info]  
> **Domain Name System (DNS)** translates human-readable domain names such as `hackthebox.com` into IP addresses such as `104.17.42.72`.
> 
> DNS is primarily associated with **UDP/53**, but **TCP/53 is also fundamental** and is used when DNS communication requires reliable transport, especially for operations such as **DNS zone transfers** or when UDP responses are too large.

---

## 🧠 2. Why DNS Matters During a Pentest

DNS is much more than a simple "domain → IP" lookup service.

A DNS server can reveal:

- Internal and external hostnames
    
- Subdomains
    
- Mail servers
    
- Name servers
    
- Infrastructure naming conventions
    
- Third-party services
    
- Cloud-hosted resources
    
- Potentially forgotten or abandoned services
    
- Internal network structure
    

Because so many network applications depend on DNS, weaknesses in DNS configuration can significantly expand the attack surface.

### 🎯 Pentesting Mindset

Think of DNS as an **information-disclosure and infrastructure-mapping service**:

```text
                    DNS
                     │
          ┌──────────┴──────────┐
          │                     │
     Enumeration            Misconfiguration
          │                     │
   ┌──────┼──────┐        ┌─────┴─────┐
   │      │      │        │           │
Subdomains  Hosts  MX   Zone       Weak DNS
                      Transfer    Configuration
                        │             │
                        ↓             ↓
                  More Attack     DNS Spoofing
                     Surface
```

> [!important]  
> **CPTS Mental Model:** When you discover DNS, don't only ask _"What IP does this domain resolve to?"_ Ask:
> 
> **"What does this DNS infrastructure reveal about the organization, and is it misconfigured?"**

---

# 🔎 3. DNS Enumeration

## 3.1 Initial Nmap Enumeration

Start by determining whether DNS is exposed and identifying the DNS implementation/version.

```bash
nmap -p53 -Pn -sV -sC 10.10.110.213
```

### What the options do

|Option|Meaning|
|---|---|
|`-p53`|Scan port 53|
|`-Pn`|Treat host as up; skip host discovery|
|`-sV`|Detect service/version|
|`-sC`|Run Nmap's default scripts|

Example:

```text
PORT    STATE  SERVICE  VERSION
53/tcp  open   domain   ISC BIND 9.11.3-1ubuntu1.2 (Ubuntu Linux)
```

### 🧠 What did we learn?

```text
53/tcp open
      ↓
DNS service exposed
      ↓
Service = domain
      ↓
Implementation = ISC BIND
      ↓
Version = 9.11.3
      ↓
Potential next steps:
Zone Transfer
DNS enumeration
Subdomain discovery
Version-specific research
```

> [!tip]  
> DNS enumeration should normally be part of your **initial service enumeration** whenever port `53` is discovered.

---

# 🔄 4. DNS Zone Transfers

## 4.1 What Is a DNS Zone?

A **DNS zone** is a portion of the DNS namespace managed by a particular organization or administrator.

For example:

```text
inlanefreight.htb
│
├── admin.inlanefreight.htb
├── hr.inlanefreight.htb
├── support.inlanefreight.htb
└── ...
```

The DNS server maintains records describing these hosts.

A **zone transfer** is a mechanism used to copy DNS zone information from one DNS server to another.

Normally, this should be restricted to authorized DNS servers.

---

## ⚠️ 4.2 Why Is an Unrestricted Zone Transfer Dangerous?

DNS zone transfers do not inherently require authentication.

If the DNS server is incorrectly configured to allow anyone to request a zone transfer, an attacker may be able to retrieve a large amount of DNS information in one operation.

```text
Attacker
   │
   │ AXFR request
   ↓
DNS Server
   │
   │ Entire zone
   ↓
Attacker
```

Instead of discovering hosts one at a time:

```text
admin.target.com
hr.target.com
support.target.com
...
```

you may receive the entire zone.

> [!important]  
> **Zone transfer = potentially massive information disclosure.**
> 
> It can reveal hosts and subdomains that may not be publicly obvious and therefore increase the attack surface.

---

# 🧪 5. Performing a Zone Transfer with DIG

The DNS query type used for a full zone transfer is:

```text
AXFR
```

Use `dig`:

```bash
dig AXFR @ns1.inlanefreight.htb inlanefreight.htb
```

### Command structure

```text
dig AXFR @DNS_SERVER DOMAIN
     │      │          │
     │      │          └── Zone to transfer
     │      └───────────── DNS server to query
     └──────────────────── Request full zone transfer
```

Example results may contain:

```text
inlanefreight.htb.         IN SOA  ...
inlanefreight.htb.         IN NS   localhost.
inlanefreight.htb.         IN A    10.129.110.22
admin.inlanefreight.htb.   IN A    10.129.110.21
hr.inlanefreight.htb.      IN A    10.129.110.25
support.inlanefreight.htb. IN A    10.129.110.28
```

### 🎯 What should you look for?

Particularly interesting records include:

- `A` → hostname → IPv4 address
    
- `AAAA` → hostname → IPv6 address
    
- `NS` → authoritative name servers
    
- `MX` → mail servers
    
- `CNAME` → aliases / third-party infrastructure
    
- `TXT` → miscellaneous information
    
- `SOA` → zone authority information
    

---

## 🔌 5.1 Why Does Zone Transfer Use TCP?

Normal DNS queries commonly use:

```text
UDP/53
```

But DNS zone transfers use:

```text
TCP/53
```

because a zone transfer can contain a significant amount of data and requires reliable transmission.

Therefore:

```text
Normal DNS query
       ↓
    UDP/53
       │
       └── TCP fallback when required

Zone Transfer (AXFR)
       ↓
    TCP/53
```

> [!important]  
> This is why discovering **TCP/53** is especially relevant when testing for `AXFR`.

---

# 🦁 6. Using Fierce for DNS Enumeration

`Fierce` can enumerate DNS infrastructure and check for zone transfers.

```bash
fierce --domain zonetransfer.me
```

Example:

```text
NS: nsztm2.digi.ninja. nsztm1.digi.ninja.
SOA: nsztm1.digi.ninja. (81.4.108.41)
Zone: success
```

If the zone transfer succeeds, Fierce may return many DNS records.

These can reveal information such as:

```text
Subdomains
      ↓
Internal naming
      ↓
Mail infrastructure
      ↓
Third-party services
      ↓
Potentially sensitive DNS records
```

For example, a zone may expose:

```text
internal
office
home
support
email
dc-office
canberra-office
...
```

> [!warning]  
> A successful zone transfer does **not automatically mean direct compromise**. Its immediate impact is generally **information disclosure**, but the disclosed information can lead to additional attack paths.

---

# 🕵️ 7. Domain & Subdomain Takeovers

## 7.1 Domain Takeover

A **domain takeover** occurs when an attacker registers a domain that an organization previously used but allowed to expire.

If the attacker obtains the abandoned domain, they may use it to:

- Host malicious content
    
- Send phishing emails
    
- Impersonate the organization
    
- Abuse trust associated with the old domain
    

---

# 🎯 7.2 Subdomain Takeover

A similar problem can occur with **subdomains**.

Consider:

```text
sub.target.com
       │
       │ CNAME
       ↓
anotherdomain.com
```

DNS contains:

```text
sub.target.com.  60  IN  CNAME  anotherdomain.com
```

Now imagine:

```text
anotherdomain.com
        ↓
expires
        ↓
becomes available for registration
        ↓
attacker registers it
        ↓
sub.target.com
        ↓
still points to attacker-controlled domain
```

The attacker can potentially gain control over the content served through:

```text
sub.target.com
```

### 🧠 Core Concept

The important issue is **dangling DNS**:

```text
Organization
     │
     └── sub.target.com
             │
             │ CNAME
             ↓
      Third-party service
             │
             ↓
      Resource deleted
             │
             ↓
       DNS still exists
             │
             ↓
       Attacker claims resource
```

> [!important]  
> The DNS record itself can remain valid even after the external resource it points to has been deleted or abandoned.

---

# 🔎 8. Subdomain Enumeration

Before looking for subdomain takeovers, you first need to discover the organization's subdomains.

One option is **Subfinder**.

```bash
./subfinder -d inlanefreight.com -v
```

Example results:

```text
ns2.inlanefreight.com
www.inlanefreight.com
ns1.inlanefreight.com
support.inlanefreight.com
```

Subfinder can gather subdomains from various open sources.

### Methodology

```text
Target domain
     ↓
Enumerate subdomains
     ↓
Collect discovered hosts
     ↓
Inspect DNS records
     ↓
Identify CNAMEs
     ↓
Look for dangling/abandoned services
     ↓
Assess takeover possibility
```

---

# 🧰 9. Subbrute

`Subbrute` is another subdomain enumeration tool.

It is particularly useful when performing DNS brute-forcing with your own resolvers.

Install:

```bash
git clone https://github.com/TheRook/subbrute.git >> /dev/null 2>&1
```

Enter the directory:

```bash
cd subbrute
```

Create a resolver list:

```bash
echo "ns1.inlanefreight.com" > ./resolvers.txt
```

Run the brute-force enumeration:

```bash
./subbrute.py inlanefreight.com -s ./names.txt -r ./resolvers.txt
```

Example:

```text
inlanefreight.com
ns2.inlanefreight.com
www.inlanefreight.com
ms1.inlanefreight.com
support.inlanefreight.com
```

### 🧠 Why use Subbrute?

The important idea is:

```text
Wordlist
   +
DNS Resolver
   +
Target Domain
   ↓
DNS brute-force
   ↓
Potential subdomains
```

This can be particularly useful during **internal penetration tests** where hosts may not have direct Internet access.

---

# 🔗 10. Enumerating CNAME Records

After discovering subdomains, investigate what they point to.

Use:

```bash
host support.inlanefreight.com
```

Example:

```text
support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com
```

This tells us:

```text
support.inlanefreight.com
          │
          │ CNAME
          ↓
inlanefreight.s3.amazonaws.com
```

Now the question becomes:

> **Does the referenced AWS resource still exist?**

If the website returns:

```text
NoSuchBucket
```

the resource may no longer exist.

That creates a potential takeover condition.

---

# ☁️ 11. Example: AWS S3 Subdomain Takeover

The source example shows:

```text
support.inlanefreight.com
        │
        │ CNAME
        ↓
inlanefreight.s3.amazonaws.com
```

The S3 resource returns:

```text
NoSuchBucket
```

This indicates that the referenced bucket may not exist.

The potential attack path is:

```text
CNAME discovered
      ↓
Third-party resource identified
      ↓
Resource appears missing
      ↓
Check whether resource can be legitimately claimed
      ↓
If claimable → potential subdomain takeover
```

> [!warning]  
> A `NoSuchBucket` or similar error is **an indicator**, not by itself proof that takeover is possible. You must verify whether the referenced resource can actually be claimed and whether the DNS/service configuration permits control.

---

# 📚 12. can-i-take-over-xyz

The **can-i-take-over-xyz** project is a useful reference when assessing subdomain takeover possibilities.

It provides information about:

- Services known to be susceptible
    
- Conditions required for takeover
    
- Detection indicators
    
- Assessment guidance
    

The important CPTS concept is not memorizing every service.

Instead, remember:

```text
CNAME
  ↓
Third-party service
  ↓
Resource missing?
  ↓
Is resource claimable?
  ↓
Potential takeover
```

---

# ☠️ 13. DNS Spoofing / DNS Cache Poisoning

DNS spoofing is also referred to as **DNS cache poisoning**.

The basic idea is to cause a victim to receive a false DNS answer.

Instead of:

```text
inlanefreight.com
        ↓
Legitimate IP
```

the victim receives:

```text
inlanefreight.com
        ↓
Attacker IP
```

Therefore:

```text
Victim
  │
  │ DNS lookup
  ↓
False DNS response
  │
  ↓
Attacker-controlled IP
  │
  ↓
Fake / malicious destination
```

This can redirect traffic to fraudulent infrastructure.

---

# 🧠 14. DNS Spoofing Attack Paths

The source identifies two broad paths.

### Path 1 — MITM

```text
Victim
   │
   ↓
Attacker performs MITM
   │
   ↓
DNS communication intercepted/manipulated
   │
   ↓
False DNS response
   │
   ↓
Fraudulent destination
```

### Path 2 — Compromised DNS Server

```text
DNS Server vulnerability
          ↓
Attacker gains control
          ↓
DNS records modified
          ↓
Victims receive malicious resolution
```

> [!important]  
> DNS spoofing is fundamentally about **manipulating name resolution**, not attacking the application directly.

---

# 🧪 15. Local DNS Cache Poisoning with Ettercap

For a local-network scenario, tools such as **Ettercap** or **Bettercap** can be used as part of a MITM-based DNS spoofing attack.

The source demonstrates Ettercap.

---

## 15.1 Configure DNS Spoofing Rules

Edit:

```bash
/etc/ettercap/etter.dns
```

Example:

```text
inlanefreight.com      A   192.168.225.110
*.inlanefreight.com    A   192.168.225.110
```

This specifies that requests for:

```text
inlanefreight.com
```

and:

```text
*.inlanefreight.com
```

should resolve to:

```text
192.168.225.110
```

### 🧠 Translation

```text
Victim asks:

"What is the IP of inlanefreight.com?"

        ↓

Spoofed DNS response:

"inlanefreight.com = 192.168.225.110"

        ↓

Victim connects to attacker-controlled host
```

---

# 🎯 16. Ettercap MITM Targeting

The source workflow is:

```text
1. Start Ettercap
        ↓
2. Scan for live hosts
        ↓
3. Identify victim
        ↓
4. Set victim as Target 1
        ↓
5. Set default gateway as Target 2
        ↓
6. Enable dns_spoof plugin
        ↓
7. Victim performs DNS lookup
        ↓
8. Spoofed DNS response redirects victim
```

The source example uses:

```text
Victim:
192.168.152.129

Gateway:
192.168.152.2

Attacker DNS destination:
192.168.225.110
```

The `dns_spoof` plugin sends fake DNS responses causing:

```text
inlanefreight.com
        ↓
192.168.225.110
```

instead of the legitimate destination.

> [!warning]  
> DNS spoofing and MITM techniques should only be performed in an authorized lab or penetration test. On a real network, manipulating another user's DNS resolution can disrupt traffic and compromise communications.

---

# 🖥️ 17. Verifying the Spoof

If the attack succeeds, visiting:

```text
http://inlanefreight.com/
```

from the victim machine may reach the attacker's fake page.

DNS resolution can also be verified with:

```cmd
ping inlanefreight.com
```

Example:

```text
Pinging inlanefreight.com [192.168.225.110] with 32 bytes of data:
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
```

The important verification is:

```text
Expected legitimate IP
        ≠
Observed IP

Observed IP
        =
Attacker-controlled IP
```

---

# 🧭 18. DNS Attack Methodology / Decision Tree

This is the part to remember for CPTS: **don't memorize tools independently; connect the technique to what you discovered.**

```text
                         🌐 DNS DISCOVERED
                                │
                                ↓
                    🔎 Enumerate DNS service
                                │
                                ↓
                       TCP/UDP 53 available?
                                │
                                ↓
                     Identify DNS implementation
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
          🔄 Zone Transfer?              🌐 Domain Mapping
                 │                             │
             Try AXFR                         ↓
                 │                       Enumerate subdomains
          ┌──────┴──────┐                │
          │             │                ↓
       SUCCESS         FAIL          Inspect CNAMEs
          │             │                │
          ↓             │                ↓
   Dump DNS zone       │       Third-party service?
          │             │          │           │
          ↓             │         YES           NO
   Extract hosts       │          │
   /subdomains         │          ↓
          │             │     Resource missing?
          │             │          │
          │             │       YES ↓
          │             │    Check takeover
          │             │
          └─────────────┴──────────────────────┐
                                               │
                                               ↓
                                    🕵️ DNS Spoofing?
                                               │
                                      Local/MITM access?
                                               │
                                               ↓
                                     Authorized scenario
                                               │
                                               ↓
                                       DNS manipulation
```

---

# 📊 19. DNS Attack Cheat Sheet

|Finding / Situation|Technique|Tool / Command|Objective|
|---|---|---|---|
|TCP/UDP 53 open|DNS enumeration|`nmap -p53 -Pn -sV -sC TARGET`|Identify DNS service/version|
|DNS server discovered|Zone transfer|`dig AXFR @DNS_SERVER DOMAIN`|Attempt to dump entire DNS zone|
|Need broader DNS enumeration|DNS enumeration|`fierce --domain DOMAIN`|Discover DNS infrastructure / test zone transfer|
|Need subdomains|Passive enumeration|`subfinder -d DOMAIN -v`|Discover subdomains|
|Internal DNS / custom resolver|DNS brute force|`subbrute.py DOMAIN -s names.txt -r resolvers.txt`|Discover subdomains|
|Subdomain discovered|CNAME enumeration|`host SUBDOMAIN`|Identify third-party destination|
|CNAME points to missing resource|Subdomain takeover assessment|Service-specific checks|Determine whether resource can be claimed|
|Local network + MITM position|DNS spoofing|Ettercap / Bettercap|Return fraudulent DNS responses|

---

# 🔀 20. The Most Important DNS Distinctions

|Technique|What Is Being Attacked?|What You Need|Main Result|
|---|---|---|---|
|**DNS Enumeration**|DNS information|DNS server / domain|Discover infrastructure|
|**Zone Transfer (AXFR)**|DNS configuration|Misconfigured DNS server|Potentially dump entire zone|
|**Subdomain Enumeration**|Domain namespace|Target domain / DNS sources|Discover subdomains|
|**Subdomain Takeover**|Dangling DNS → external service|CNAME + abandoned claimable resource|Gain control of subdomain|
|**DNS Spoofing**|DNS resolution|MITM/control over DNS path or server|Redirect victim to false IP|

---

# 🧠 21. CPTS Mental Model

The complete thought process should be:

```text
                 DNS / 53
                    │
                    ↓
             "What is here?"
                    │
                    ↓
          Enumerate DNS service
                    │
                    ↓
        "Can I get more information?"
                    │
              ┌─────┴─────┐
              ↓           ↓
            AXFR       Subdomains
              │           │
              ↓           ↓
        Zone records    CNAMEs
                          │
                          ↓
                 Third-party service
                          │
                          ↓
                 Resource abandoned?
                          │
                          ↓
                  Takeover possible?
```

A separate branch exists for local-network attacks:

```text
Local network access
        ↓
MITM position
        ↓
Manipulate DNS responses
        ↓
Victim resolves domain
        ↓
Attacker-controlled IP
        ↓
Redirect victim
```

### 🔑 The key question at every stage

> **"What did I just discover, and what does that discovery allow me to test next?"**

For example:

```text
53/tcp open
   ↓
DNS server
   ↓
Try AXFR
   ↓
Subdomains discovered
   ↓
Inspect CNAMEs
   ↓
Third-party resource found
   ↓
Resource missing
   ↓
Assess takeover
```

That is the **methodology**, rather than simply memorizing `dig`, `fierce`, `subfinder`, and `Ettercap`.

---

# ✅ 22. Key Takeaways

- 🌐 DNS translates domain names into IP addresses and commonly operates on **UDP/53**, with **TCP/53** also being important.
    
- 🔎 Use Nmap `-sC` and `-sV` for initial DNS service enumeration.
    
- 🔄 **AXFR** is the DNS query type used for zone transfers.
    
- ⚠️ An unrestricted zone transfer can disclose an organization's DNS namespace.
    
- 🧰 `dig`, `fierce`, `subfinder`, and `subbrute` provide different ways to enumerate DNS information.
    
- 🔗 **CNAME records** are particularly important when assessing subdomain takeovers.
    
- ☁️ Third-party/cloud services can leave **dangling DNS records** when resources are deleted.
    
- 🕵️ DNS spoofing/cache poisoning attempts to make victims resolve legitimate domains to fraudulent IP addresses.
    
- 🎯 Local DNS spoofing can be performed in an authorized lab using MITM tooling such as Ettercap.
    
- 🧠 The core CPTS workflow is:
    

```text
DNS discovered
     ↓
Enumerate
     ↓
Try AXFR
     ↓
Enumerate subdomains
     ↓
Inspect DNS records / CNAMEs
     ↓
Identify third-party infrastructure
     ↓
Assess takeover opportunities
     ↓
If local MITM conditions exist → assess DNS spoofing
```

> [!success]  
> **CPTS Exam Mental Model:**  
> **DNS is an attack-surface map.** First extract information from it. Then follow what that information points to—hosts, subdomains, mail systems, third-party services, dangling resources, or opportunities to manipulate name resolution.