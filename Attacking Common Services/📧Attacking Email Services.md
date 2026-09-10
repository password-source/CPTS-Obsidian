# 📧 1. Attacking Email Services

> [!info]  
> **Email services** allow users and systems to send, receive, and store email messages. During a penetration test, email infrastructure can expose useful information about the organization and may contain authentication weaknesses, username enumeration vulnerabilities, password attacks, or dangerous relay configurations.

---

# 🧠 2. How Email Services Work

A typical email flow involves three components/protocols:

```text
┌──────────────┐
│ Email Client │
│ PC / Phone   │
└──────┬───────┘
       │
       │ SMTP
       ↓
┌──────────────┐
│ Mail Server  │
└──────┬───────┘
       │
       │ SMTP
       ↓
┌──────────────┐
│ Other Mail   │
│ Server       │
└──────┬───────┘
       │
       │ IMAP / POP3
       ↓
┌──────────────┐
│ Recipient    │
│ Email Client │
└──────────────┘
```

## 📤 SMTP — Sending Email

**SMTP (Simple Mail Transfer Protocol)** is used to deliver email:

```text
Client → SMTP Server
SMTP Server → SMTP Server
```

It is primarily the **sending/transfer protocol**.

---

## 📥 POP3 — Retrieving Email

**POP3** allows a client to retrieve messages from a mailbox.

By default, POP3 clients generally **remove downloaded messages from the server**, although they can usually be configured to keep copies.

```text
Mail Server
     │
     │ POP3
     ↓
Email Client
     │
     ↓
Local message
```

---

## 📥 IMAP4 — Retrieving Email

**IMAP4** also allows clients to access email stored on a mail server.

Unlike the typical POP3 behavior, IMAP4 normally **keeps messages on the server**.

This makes IMAP particularly useful when accessing the same mailbox from multiple devices.

```text
                  ┌── PC
                  │
Mail Server ──────┼── Phone
                  │
                  └── Tablet
```

> [!important]  
> For pentesting, remember the basic distinction:
> 
> **SMTP = send/transfer**  
> **POP3 = retrieve/download**  
> **IMAP = access/synchronize mailbox**

---

# 🔎 3. Email Enumeration Methodology

Email infrastructure can be complicated because organizations may use:

- Self-hosted mail servers
    
- Microsoft 365
    
- Google Workspace/G-Suite
    
- Zoho
    
- Other cloud providers
    
- Multiple mail servers for redundancy
    

Therefore, the **first question is:**

> **"What mail infrastructure does this organization use?"**

A good starting point is DNS.

```text
Target Domain
      ↓
Query MX records
      ↓
Identify mail provider/server
      ↓
       ┌───────────────┐
       │               │
   Cloud Email     Custom Server
       │               │
       ↓               ↓
Provider-specific   SMTP/POP3/IMAP
enumeration         enumeration
```

---

# 🌐 4. Finding Mail Servers with MX Records

## 4.1 What Is an MX Record?

An **MX (Mail eXchanger)** DNS record identifies the mail server responsible for receiving email for a domain.

Example:

```text
hackthebox.eu
      ↓
aspmx.l.google.com
```

An organization can have multiple MX records for:

- Redundancy
    
- Load balancing
    
- Failover
    

---

## 🔍 4.2 Using `host`

Query MX records:

```bash
host -t MX hackthebox.eu
```

Example:

```text
hackthebox.eu mail is handled by 1 aspmx.l.google.com.
```

Another example:

```bash
host -t MX microsoft.com
```

Output:

```text
microsoft.com mail is handled by 10 microsoft-com.mail.protection.outlook.com.
```

The number is the **MX preference/priority**.

---

# 🔬 5. Using DIG for MX Enumeration

You can also use `dig`:

```bash
dig mx plaintext.do | grep "MX" | grep -v ";"
```

Example:

```text
plaintext.do. 7076 IN MX 50 mx3.zoho.com.
plaintext.do. 7076 IN MX 10 mx.zoho.com.
plaintext.do. 7076 IN MX 20 mx2.zoho.com.
```

Another example:

```bash
dig mx inlanefreight.com | grep "MX" | grep -v ";"
```

Output:

```text
inlanefreight.com. 300 IN MX 10 mail1.inlanefreight.com.
```

Now resolve the mail server:

```bash
host -t A mail1.inlanefreight.htb.
```

Example:

```text
mail1.inlanefreight.htb has address 10.129.14.128
```

---

# 🧠 6. Why MX Enumeration Matters

The MX record tells you **which attack surface you are dealing with**.

For example:

```text
MX Record
   │
   ├── aspmx.l.google.com
   │        ↓
   │     Google
   │
   ├── microsoft-com.mail.protection.outlook.com
   │        ↓
   │     Microsoft 365
   │
   ├── mx.zoho.com
   │        ↓
   │     Zoho
   │
   └── mail1.target.htb
            ↓
       Custom mail server
```

Different providers require different enumeration and attack techniques.

> [!important]  
> **Do not immediately attack SMTP/POP3/IMAP just because you found an MX record.**
> 
> First determine whether the organization uses a **cloud provider** or a **custom mail server**.

---

# 📡 7. Common Mail Service Ports

For a custom mail server, enumerate the following:

|**PortService**||
|---|---|
|`TCP/25`|SMTP Unencrypted|
|`TCP/143`|IMAP4 Unencrypted|
|`TCP/110`|POP3 Unencrypted|
|`TCP/465`|SMTP Encrypted|
|`TCP/587`|SMTP Encrypted/[STARTTLS](https://en.wikipedia.org/wiki/Opportunistic_TLS)|
|`TCP/993`|IMAP4 Encrypted|
|`TCP/995`|POP3 Encrypted|

### 🧠 Memorize the important relationships

```text
SMTP
├── 25   → SMTP
├── 465  → SMTP encrypted
└── 587  → SMTP + STARTTLS

IMAP
├── 143  → IMAP
└── 993  → IMAP encrypted

POP3
├── 110  → POP3
└── 995  → POP3 encrypted
```

---

# 🔎 8. Nmap Mail Server Enumeration

Scan the common mail ports:

```bash
sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 10.129.14.128
```

Example:

```text
PORT   STATE SERVICE VERSION
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: mail1.inlanefreight.htb, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING,
```

### 🎯 Important Discovery

The Nmap output reveals that the SMTP server supports:

```text
VRFY
```

That is immediately interesting because `VRFY` can potentially be used for **username enumeration**.

This demonstrates the general methodology:

```text
Port discovered
      ↓
Service identified
      ↓
Nmap scripts enumerate capabilities
      ↓
Interesting command discovered
      ↓
Test whether it can disclose usernames
```

---

# ⚠️ 9. Email Server Misconfigurations

A major email attack surface is **authentication**.

Potential weaknesses include:

- Anonymous SMTP access
    
- Username enumeration
    
- Weak credentials
    
- Password reuse
    
- Password spraying
    
- Brute forcing
    
- Open SMTP relay
    

The methodology is:

```text
Email service
     ↓
Enumerate capabilities
     ↓
Can usernames be discovered?
     ↓
Valid usernames
     ↓
Credential attacks
     ↓
Valid credentials
     ↓
Mailbox / service access
```

---

# 👤 10. SMTP Username Enumeration

SMTP may expose commands that allow us to determine whether a username exists.

The important commands are:

```text
VRFY
EXPN
RCPT TO
```

---

# 🔍 11. VRFY

`VRFY` asks the SMTP server to verify whether a particular user exists.

Connect to SMTP:

```bash
telnet 10.10.110.20 25
```

Example:

```text
220 parrot ESMTP Postfix (Debian/GNU)
```

Test a username:

```text
VRFY root
```

Response:

```text
252 2.0.0 root
```

Test another:

```text
VRFY www-data
```

Response:

```text
252 2.0.0 www-data
```

Now test an invalid user:

```text
VRFY new-user
```

Response:

```text
550 5.1.1 <new-user>: Recipient address rejected: User unknown in local recipient table
```

### 🧠 Interpretation

```text
VRFY username
      ↓
SMTP server
      ↓
┌───────────────┬────────────────┐
│ User exists   │ User unknown   │
│ 252 response  │ 550 response   │
└───────────────┴────────────────┘
```

> [!important]  
> The exact response codes can vary depending on the mail server implementation. The important concept is that **different responses can disclose whether an account exists**.

---

# 📋 12. EXPN

`EXPN` is similar to `VRFY`, but can be more dangerous.

It can expand a distribution list and potentially reveal the users belonging to it.

Example:

```text
EXPN john
```

Response:

```text
250 2.1.0 john@inlanefreight.htb
```

A distribution list:

```text
EXPN support-team
```

could return:

```text
250 2.0.0 carol@inlanefreight.htb
250 2.1.5 elisa@inlanefreight.htb
```

### 🎯 Why is EXPN interesting?

One query can potentially reveal **multiple valid usernames**.

```text
support-team
      ↓
Distribution list
      ↓
┌───────────┐
│ carol     │
│ elisa     │
└───────────┘
```

> [!warning]  
> Distribution aliases such as `all` can potentially expose many users at once if `EXPN` is enabled.

---

# 📬 13. RCPT TO

`RCPT TO` specifies the recipient of an email.

It can also reveal whether a recipient exists, depending on the SMTP implementation.

Example:

```text
MAIL FROM:test@htb.com
```

Response:

```text
250 2.1.0 test@htb.com... Sender ok
```

Then:

```text
RCPT TO:julio
```

Response:

```text
550 5.1.1 julio... User unknown
```

Try another:

```text
RCPT TO:john
```

Response:

```text
250 2.1.5 john... Recipient ok
```

### 🧠 Interpretation

```text
RCPT TO:user
      ↓
SMTP server
      ↓
Recipient accepted?
      │
 ┌────┴────┐
 ↓         ↓
YES       NO
 ↓         ↓
Likely    Likely
valid     invalid
user      user
```

---

# 📥 14. POP3 Username Enumeration

SMTP isn't the only protocol that can disclose usernames.

Depending on the implementation, POP3 may reveal whether a user exists through the `USER` command.

Connect:

```bash
telnet 10.10.110.20 110
```

Server:

```text
+OK POP3 Server ready
```

Test:

```text
USER julio
```

Response:

```text
-ERR
```

Then:

```text
USER john
```

Response:

```text
+OK
```

The server behavior may indicate that `john` exists.

```text
USER john
   ↓
+OK
   ↓
Potential valid account
```

> [!important]  
> Username enumeration is valuable because it changes the next phase of the attack.
> 
> Instead of guessing credentials against random usernames:
> 
> **Enumerate → obtain valid usernames → attack credentials.**

---

# 🤖 15. Automating SMTP Enumeration

Manually testing users is inefficient.

`smtp-user-enum` can automate enumeration.

It supports:

```text
VRFY
EXPN
RCPT
```

### Syntax

```bash
smtp-user-enum -M MODE -U USERLIST -D DOMAIN -t TARGET
```

Example:

```bash
smtp-user-enum -M RCPT -U userlist.txt -D inlanefreight.htb -t 10.129.203.7
```

Important options:

|Option|Purpose|
|---|---|
|`-M`|Enumeration method: `VRFY`, `EXPN`, or `RCPT`|
|`-U`|Username list|
|`-D`|Email domain|
|`-t`|Target mail server|

Example results:

```text
10.129.203.7: jose@inlanefreight.htb exists
10.129.203.7: pedro@inlanefreight.htb exists
10.129.203.7: kate@inlanefreight.htb exists
```

Now we have:

```text
Unknown usernames
       ↓
smtp-user-enum
       ↓
Valid usernames
       ↓
Credential attack
```

---

# ☁️ 16. Cloud Email Enumeration

Modern organizations frequently use cloud email services.

Examples from the source include:

- Microsoft 365
    
- Google/G-Suite
    
- Zoho
    

The attack methodology changes because you may not be directly attacking a traditional SMTP server.

```text
Domain
  ↓
MX enumeration
  ↓
Identify provider
  ↓
Cloud provider?
  ↓
Provider-specific enumeration
```

---

# 🪟 17. Microsoft 365 Enumeration

`O365spray` can be used to validate whether a domain uses Office 365 and enumerate accounts.

First validate the domain:

```bash
python3 o365spray.py --validate --domain msplaintext.xyz
```

Example:

```text
[VALID] The following domain is using O365: msplaintext.xyz
```

Now enumerate users:

```bash
python3 o365spray.py --enum -U users.txt --domain msplaintext.xyz
```

Example:

```text
[VALID] lewen@msplaintext.xyz
[VALID] juurena@msplaintext.xyz
```

The tool reports:

```text
Valid Accounts: 2
```

### 🧠 Methodology

```text
Target domain
      ↓
Validate O365
      ↓
Enumerate candidate usernames
      ↓
Valid accounts
      ↓
Password attack
```

---

# 🔐 18. Password Attacks Against Email

Once valid usernames are identified, credentials become the next attack surface.

Hydra can target services such as:

- SMTP
    
- POP3
    
- IMAP4
    

Example against POP3:

```bash
hydra -l "marlin@inlanefreight.htb" -P pws.list -f 10.129.203.12 pop3
```

Important options:

|Option|Meaning|
|---|---|
|`-L users.txt`|Username list|
|`-p 'Company01!'`|Single password|
|`-f`|Stop after finding a valid credential|
|`pop3`|Attack POP3|

Example:

```text
[110][pop3] host: 10.129.42.197 login: john password: Company01!
1 of 1 target successfully completed, 1 valid password found
```

---

# 🧠 19. Password Spraying vs Brute Force

The same distinction applies here as with other network services.

### Password Spraying

```text
john      ─┐
jose       │
kate       ├──→ Company01!
pedro      │
admin     ─┘
```

One password → many users.

### Brute Force

```text
john
 │
 ├── password1
 ├── password2
 ├── password3
 ├── password4
 └── ...
```

One user → many passwords.

> [!warning]  
> Email providers often implement lockout, throttling, MFA, and automated attack detection. Credential attacks must therefore be carefully controlled during authorized assessments.

---

# ☁️ 20. Cloud Password Spraying

Cloud email providers may block generic tools such as Hydra or expose different authentication mechanisms.

For Microsoft 365, the source demonstrates `o365spray`.

Example:

```bash
python3 o365spray.py --spray -U usersfound.txt -p 'March2022!' --count 1 --lockout 1 --domain msplaintext.xyz
```

Important concepts in the command:

```text
--spray
    ↓
Password spraying mode

-U usersfound.txt
    ↓
Username list

-p 'March2022!'
    ↓
Password to spray

--count 1
    ↓
One password per spray

--lockout 1
    ↓
Lockout-related safety setting
```

Example result:

```text
[VALID] lewen@msplaintext.xyz:March2022!
```

The tool reports:

```text
Valid Credentials: 1
```

### 🧠 Cloud Attack Chain

```text
MX
 ↓
Microsoft 365 identified
 ↓
O365-specific enumeration
 ↓
Valid usernames
 ↓
Controlled password spray
 ↓
Valid credentials
 ↓
Cloud account access
```

---

# 🔓 21. Protocol-Specific Attack: SMTP Open Relay

An **open relay** is an SMTP server configured to allow unauthenticated users to relay email through it.

Normally:

```text
Unauthorized User
       │
       X
       ↓
SMTP Server
```

With an open relay:

```text
Attacker
   │
   │ unauthenticated email
   ↓
Open SMTP Relay
   │
   ↓
Recipient
```

The relay can make the email appear to originate from the relay server.

---

# ⚠️ 22. Why Is an Open Relay Dangerous?

An attacker could potentially abuse an open relay to:

- Send spoofed emails
    
- Impersonate addresses
    
- Send phishing messages
    
- Abuse the organization's mail infrastructure
    
- Damage the organization's reputation
    

Example attack concept:

```text
Attacker
   │
   │ "From: notifications@company.com"
   ↓
Open Relay
   │
   ↓
Employees
   │
   ↓
Trust the apparent sender
```

> [!important]  
> An open relay is primarily a **mail trust/configuration weakness**. It does not necessarily mean the attacker has compromised the mail server itself.

---

# 🔎 23. Detecting an Open Relay

Nmap provides the `smtp-open-relay` script.

```bash
nmap -p25 -Pn --script smtp-open-relay 10.10.11.213
```

Example:

```text
PORT   STATE SERVICE
25/tcp open  smtp
|_smtp-open-relay: Server is an open relay (14/16 tests)
```

The important finding is:

```text
smtp-open-relay: Server is an open relay
```

---

# 📧 24. Sending Mail Through an Open Relay

`swaks` can be used as an SMTP testing client.

The source example:

```bash
swaks --from notifications@inlanefreight.com --to employees@inlanefreight.com --header 'Subject: Company Notification' --body 'Hi All, we want to hear from you! Please complete the following survey. http://mycustomphishinglink.com/' --server 10.10.11.213
```

The command specifies:

```text
--from
    ↓
Sender address

--to
    ↓
Recipient

--header
    ↓
Email header / subject

--body
    ↓
Email contents

--server
    ↓
SMTP server
```

The SMTP conversation demonstrates:

```text
MAIL FROM
    ↓
RCPT TO
    ↓
DATA
    ↓
Message
    ↓
.
    ↓
250 OK
```

A successful:

```text
250 OK
```

indicates that the SMTP server accepted the message.

> [!warning]  
> Open-relay testing should be performed only against authorized recipients and infrastructure. Sending deceptive or phishing-style messages outside an approved engagement can cause real-world harm.

---

# 🧭 25. Complete Email Attack Methodology

This is the important CPTS workflow to remember:

```text
                    📧 EMAIL ATTACK SURFACE
                              │
                              ↓
                       🔎 Find MX records
                              │
                              ↓
                    Identify mail provider
                              │
              ┌───────────────┴────────────────┐
              │                                │
          ☁️ CLOUD                         🖥️ CUSTOM
              │                                │
              ↓                                ↓
   Provider-specific tools             Scan mail ports
              │                                │
              ↓                                ↓
      Enumerate accounts              SMTP / POP3 / IMAP
              │                                │
              │                         Enumerate capabilities
              │                                │
              │                    ┌───────────┼───────────┐
              │                    ↓           ↓           ↓
              │                  VRFY        EXPN       RCPT TO
              │                    │           │           │
              │                    └───────────┴───────────┘
              │                                │
              │                                ↓
              │                       Valid usernames
              │                                │
              └────────────────┬───────────────┘
                               ↓
                       🔐 Credential Attack
                               │
                    ┌──────────┴──────────┐
                    ↓                     ↓
               Password Spray          Brute Force
                    │                     │
                    └──────────┬──────────┘
                               ↓
                        Valid Credentials
                               │
                               ↓
                         Mail Access
```

There is another branch for SMTP configuration:

```text
SMTP/25
  ↓
Check capabilities
  ↓
Open relay?
  ↓
YES
  ↓
Assess relay abuse
```

---

# 📊 26. Email Attack Cheat Sheet

|Finding|What to Do|Tool / Command|Goal|
|---|---|---|---|
|Domain discovered|Query MX|`host -t MX DOMAIN`|Identify mail server/provider|
|Need detailed MX information|Query DNS|`dig mx DOMAIN`|Enumerate mail infrastructure|
|Custom mail server identified|Scan mail ports|`nmap -Pn -sV -sC -p25,143,110,465,587,993,995 TARGET`|Identify services/capabilities|
|SMTP `VRFY` enabled|Username enumeration|`VRFY username`|Determine whether user exists|
|SMTP `EXPN` enabled|Distribution-list enumeration|`EXPN list`|Discover users/aliases|
|SMTP `RCPT TO` behavior leaks users|Username enumeration|`RCPT TO:user`|Determine valid recipients|
|POP3 leaks account existence|Username enumeration|`USER username`|Identify valid users|
|Many usernames to test|Automate enumeration|`smtp-user-enum`|Build valid username list|
|Microsoft 365 identified|Cloud enumeration|`o365spray --validate` / `--enum`|Identify valid accounts|
|Valid usernames obtained|Password attack|`hydra` / provider-specific tooling|Find weak credentials|
|SMTP/POP3/IMAP exposed|Password spraying|Appropriate protocol/tool|Test known/common password|
|SMTP accepts unauthenticated relay|Open relay assessment|`nmap --script smtp-open-relay`|Confirm relay misconfiguration|
|Open relay confirmed|SMTP testing|`swaks`|Verify authorized relay behavior|

---

# 🔀 27. What Finding Leads to What Technique?

|What You Find|Next Question|Technique|
|---|---|---|
|MX record|Who handles our email?|`host` / `dig`|
|Google MX|Is this Google-hosted email?|Google-specific enumeration|
|Microsoft MX|Is this Microsoft 365?|`o365spray` / provider-specific enumeration|
|Zoho MX|Is this Zoho?|Provider-specific enumeration|
|Custom mail server|What protocols are exposed?|Nmap|
|SMTP/25|Can SMTP disclose users?|`VRFY`, `EXPN`, `RCPT TO`|
|POP3/110|Does `USER` disclose accounts?|POP3 enumeration|
|Valid usernames|Can credentials be guessed safely?|Password spraying / brute force|
|Open relay|Can unauthenticated mail be relayed?|`smtp-open-relay`, `swaks`|

---

# 🧠 28. The Core Methodology

The most important thing is **not memorizing individual commands**.

Think in terms of:

```text
             FIND
              ↓
        Mail infrastructure
              ↓
           IDENTIFY
              ↓
      Cloud or custom server?
              ↓
         ENUMERATE
              ↓
       Users + capabilities
              ↓
           ATTACK
              ↓
     Credentials / relay
              ↓
           ACCESS
```

For example:

```text
MX
 ↓
mail1.inlanefreight.htb
 ↓
25/tcp
 ↓
Postfix
 ↓
VRFY supported
 ↓
Username enumeration
 ↓
jose / pedro / kate
 ↓
Password spray
 ↓
Valid credentials
 ↓
Mail access
```

That is the **attack chain**.

---

# 🧠 29. CPTS Mental Model

When you encounter an email service, ask these questions in order:

```text
1️⃣ What mail server/provider does the domain use?
                ↓
2️⃣ What ports/protocols are exposed?
                ↓
3️⃣ What features/capabilities does the server expose?
                ↓
4️⃣ Can I enumerate valid usernames?
                ↓
5️⃣ Do I now have a useful username list?
                ↓
6️⃣ Can credentials be tested safely?
                ↓
7️⃣ Does the server have dangerous configuration?
                ↓
8️⃣ Can the resulting access lead somewhere else?
```

### 🎯 Remember the progression

> **MX → Mail Server → Protocol → Enumeration → Users → Credentials → Access**

And for SMTP:

> **SMTP → VRFY/EXPN/RCPT → Valid Users → Password Attack**

And for cloud:

> **MX → Cloud Provider → Provider-Specific Enumeration → Valid Accounts → Password Attack**

And for relay:

> **SMTP/25 → Open Relay Test → Relay Confirmed → Assess Impact**

---

# ✅ 30. Key Takeaways

- 📧 **SMTP** is primarily used to send/transfer email.
    
- 📥 **POP3** and **IMAP4** are used to retrieve/access email.
    
- 🌐 **MX records** are the starting point for identifying an organization's mail infrastructure.
    
- 🔎 `host -t MX` and `dig mx` can identify mail providers and servers.
    
- 📡 Important mail ports include `25`, `110`, `143`, `465`, `587`, `993`, and `995`.
    
- 👤 SMTP may leak valid usernames through **VRFY**, **EXPN**, or **RCPT TO**.
    
- 📥 POP3 implementations may leak usernames through the `USER` command.
    
- 🤖 `smtp-user-enum` automates SMTP username enumeration.
    
- ☁️ Cloud email services require **provider-specific enumeration techniques**.
    
- 🔐 Valid usernames can become the input for controlled password spraying or brute-force attacks.
    
- ⚠️ An **open SMTP relay** allows unauthenticated email relaying and can enable spoofing/phishing abuse.
    
- 🧪 `nmap --script smtp-open-relay` can test for open relay behavior.
    
- 📤 `swaks` can be used as an SMTP testing client.
    
- 🧠 The fundamental CPTS methodology is:
    

```text
MX
 ↓
Identify provider
 ↓
Enumerate mail services
 ↓
Enumerate capabilities
 ↓
Discover valid users
 ↓
Attack credentials
 ↓
Test dangerous configurations
 ↓
Use obtained access to continue the assessment
```

> [!success]  
> **CPTS Mental Model:** Don't start with "Which email tool should I run?"
> 
> Start with **"What mail infrastructure did I discover?"**
> 
> Then let the finding determine the next technique:
> 
> **MX → Provider → Protocol → Capability → Users → Credentials → Access.**