

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