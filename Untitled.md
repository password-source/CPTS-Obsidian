[



Gemini](https://gemini.google.com/)

[

New chat

](https://gemini.google.com/app)

[

Search chats

](https://gemini.google.com/search)

[

Images

](https://gemini.google.com/images)

[

Library

](https://gemini.google.com/library)

[

New notebook

](https://gemini.google.com/notebooks/create)

[AD Attack Decision Tree

](https://gemini.google.com/app/cde1d822ccc2f9e2)

[SMB Ports and Rpcclient Usage

](https://gemini.google.com/app/4b6ce8364cfbb6a1)

[Understanding TTPs in Active Directory

](https://gemini.google.com/app/0fa119848c8ca447)

[Understanding NTLMv2 Capture Refresh Behavior

](https://gemini.google.com/app/df0e82ae4dc95214)

[Fixing Missing BGP Toolkit DNS Results

](https://gemini.google.com/app/2689c5ba625a0ea5)

[Copyable RDP Text

](https://gemini.google.com/app/f2b8402f178b5eb1)

[Fixing FreeRDP Kerberos Authentication Error

](https://gemini.google.com/app/fb5bec0c67c4a6a1)

[Xfreerdp vs. Xfreerdp3 Differences

](https://gemini.google.com/app/501a4a1ae3151cc7)

[Meterpreter Pivoting on Windows and Linux

](https://gemini.google.com/app/792f4bb471d6bf72)

[Using Plink for RDP Tunneling

](https://gemini.google.com/app/e2a034839b5a5a10)

[Ligolo-ng vs Chisel for Pivoting

](https://gemini.google.com/app/0db9c7817016d8d7)

[Adding Items to Invoice

](https://gemini.google.com/app/81cfd594fe4cec72)

[Decoding Ambiguous Exam Message Context

](https://gemini.google.com/app/373e40975874a544)

[Understanding a Python Function Definition

](https://gemini.google.com/app/c3c25a5b5d533669)

[Is Visual Studio Free?

](https://gemini.google.com/app/ad730df6467ffe6e)

[Fixing MySQL SSL Connection Error

](https://gemini.google.com/app/8fc83be08ef9dfb1)

[Exploiting CoreFTP via HTTP PUT

](https://gemini.google.com/app/0cda9a5980c9965a)

[SMTP vs. POP3 and IMAP

](https://gemini.google.com/app/ea9a3cc3f56ad2ba)

[Modes and Routines on Non-Samsung Android

](https://gemini.google.com/app/7c316664974d6599)

[GitHub Repository Storage Limits

](https://gemini.google.com/app/bc6d34dd354e8621)

[Translating a Typing Typo

](https://gemini.google.com/app/70def6bdf84ac916)

[Penetration Testing Certifications Difficulty Ranking

](https://gemini.google.com/app/0a781b5423efcdc6)

[

](https://accounts.google.com/SignOutOptions?continue=https://gemini.google.com)

# Conversation with Gemini

So, I'm studying CPTS from Hack The Box and now I got confused what should be the map to attack Active Directory. These are the notes I've taken. Just show me a decision tree or a decision map to what should I do at what stage. Just make it briefly and look like a tree in a markdown .md file. Also, don't add anything from the internet or from you, just use the words that are in the in these notes. That's it. And don't go further, just stop to the point where there is a stop point in these notes.

"""

with open("ad_attack_decision_tree.md", "w", encoding="utf-8") as f: f.write(content)

print("Markdown file generated successfully.")

````

```text?code_stdout&code_event_index=1
Markdown file generated successfully.

````

Your Markdown file is ready

![MD icon](https://drive-thirdparty.googleusercontent.com/32/type/text/code)

ad_attack_decision_tree

MD

  

Gemini is AI and can make mistakes.

Analyzing

# Active Directory Attack Decision Tree

```text
🎯 TARGET
   │
   ▼
🌎 PASSIVE RECON / OSINT
   │
   ▼
🌐 ASN / IP SPACE ──→ DNS ENUMERATION ──→ PUBLIC DATA & DOCUMENTS ──→ JOB POSTINGS
   │
   ▼
👤 USERNAME HARVESTING (LinkedIn / linkedin2username)
   │
   ▼
🔑 BREACH DATA (DeHashed)
   │
   ▼
🎯 VALIDATE SCOPE / ROE
   │
   ├─► Not related / Not in scope / Not permitted ──► STOP
   │
   └─► Authorized ──→ 🔎 ACTIVE ENUMERATION & TARGETED TESTING
                          │
                          ▼
                 👤 GET VALID USERNAMES
                          │
                          ▼
            What access or method is available?
              │
              ├─► SMB NULL Session ───────────→ enum4linux / rpcclient
              ├─► LDAP Anonymous Bind ────────→ ldapsearch / windapsearch
              ├─► Kerbrute (No internal access) → Kerbrute userenum
              ├─► Valid Credentials ──────────→ CrackMapExec --users
              └─► Domain-Joined Windows Host ─→ DomainPasswordSpray
                          │
                          ▼
            🔐 CHECK PASSWORD POLICY & LOCKOUT STATE
              │
              ├─► Review lockout threshold, badpwdcount, observation window
              └─► Remove disabled users and users near lockout
                          │
                          ▼
                    🎯 PASSWORD SPRAY
              │
              ├─► Choose one password across many accounts
              ├─► Tool options:
              │     ├─► rpcclient (grep Authority)
              │     ├─► Kerbrute passwordspray
              │     └─► CrackMapExec (grep +)
              └─► Save successful results (e.g. spray_success)
                          │
                          ▼
                 🔎 VALIDATE CREDENTIALS
                          │
                          ▼
               🔐 CREDENTIALED ENUMERATION
                          │
                          ▼
          What else can be tested / targeted?
              │
              ├─► Local Administrator Password Reuse / Hash Spraying
              │     │
              │     ├─► Recover local admin password or NTLM hash
              │     ├─► ⚠️ CRITICAL: Use `--local-auth` with CrackMapExec
              │     └─► Target authorized hosts / subnets (e.g. 172.16.5.0/23)
              │
              └─► STOP (Stop to the point where there is a stop point in these notes)
```

ad_attack_decision_tree.md

Displaying ad_attack_decision_tree.md.