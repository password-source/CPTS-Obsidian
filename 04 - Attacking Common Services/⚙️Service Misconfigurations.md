# ⚙️ 1. Service Misconfigurations

> [!info]  
> **Service misconfiguration** occurs when an administrator, developer, or support technician configures an application, website, desktop, server, or service incorrectly, creating unintended paths to unauthorized access.
> 
> For a pentester, misconfigurations are important because they can provide **access without exploiting a software vulnerability**.

### 🧠 Core Mental Model

```text
Service
   │
   ▼
How is it configured?
   │
   ├── 🔑 Authentication
   │      ├── Default credentials
   │      ├── Weak credentials
   │      └── Anonymous access
   │
   ├── 🔐 Access Rights
   │      └── Too much access
   │
   └── ⚙️ Defaults
          ├── Unnecessary services
          ├── Debugging
          ├── Admin interfaces
          └── Informative errors

             ↓

        💥 Attack Surface
             ↓
      Unauthorized Access
```

---

# 🔑 2. Authentication Misconfigurations

Authentication is one of the first things to check when interacting with a service.

## 2.1 🔐 Default Credentials

Older applications and devices may ship with predefined usernames and passwords.

The security problem occurs when administrators **leave those credentials unchanged**.

Modern software often forces administrators to configure credentials during installation, but default credentials can still exist, particularly in older applications.

### Common Credentials to Test

```text
admin:admin
admin:password
admin:<blank>
root:12345678
administrator:Password
```

> [!important]  
> After identifying a service and obtaining its **banner/version information**, check whether that product has known default credentials.
> 
> If no known defaults exist, test common weak username/password combinations where permitted by the assessment.

### 🎯 Pentesting Flow

```text
🔎 Identify Service
       ↓
🏷️ Grab Banner / Version
       ↓
🔑 Search/Identify Default Credentials
       ↓
❌ No Default Credentials?
       ↓
🔓 Test Common Weak Credentials
       ↓
✅ Valid Login?
       ↓
📂 Enumerate What the Account Can Access
```

---

# 🧪 3. Weak or Blank Passwords

A service does not need to have predefined default credentials to be vulnerable.

An administrator might configure:

- A weak password
    
- A predictable password
    
- An empty password
    
- A temporary password that was never changed
    

The source highlights combinations such as:

```text
admin:admin
admin:password
admin:<blank>
root:12345678
administrator:Password
```

> [!tip]  
> **Default credentials** means the vendor supplied them.
> 
> **Weak credentials** means the administrator/user configured something insecure.
> 
> Both should be considered during authentication testing.

---

# 👤 4. Anonymous Authentication

Another common service misconfiguration is **anonymous authentication**.

Normally:

```text
Client
  │
  │ Username + Password
  ▼
Service
  │
  ▼
Access
```

With anonymous access:

```text
Client
  │
  │ No credentials
  ▼
Service
  │
  ▼
Access granted
```

This means anyone who can reach the service over the network may be able to interact with it without authentication.

> [!warning]  
> **Never assume authentication is required simply because a service normally supports authentication.**
> 
> Always test whether the service has been configured to allow anonymous access.

### 🧠 Question to Ask

> **"Can I interact with this service without providing credentials?"**

If yes:

```text
Anonymous Access
      ↓
Enumerate
      ↓
What can I read?
      ↓
What can I write/upload?
      ↓
Can I execute or modify anything?
      ↓
Can I obtain credentials or sensitive information?
```

---

# 🔐 5. Misconfigured Access Rights

Authentication only answers:

> **"Who are you?"**

Authorization/access rights answer:

> **"What are you allowed to do?"**

A service can therefore have correct authentication but still be dangerously misconfigured.

### Example

Suppose an FTP account is intended only to:

```text
📤 Upload files
```

But the account is accidentally given:

```text
📤 Upload
📥 Read
📂 Browse entire FTP server
```

The account may now expose information that was never intended for that user.

Potentially accessible information could include:

- Configuration files
    
- Plaintext credentials
    
- Usernames
    
- Proprietary information
    
- Personally identifiable information (**PII**)
    
- Credentials for other services
    

> [!important]  
> **Low-privileged credentials can still be extremely valuable.**
> 
> Don't stop enumerating after obtaining a normal user account. Determine **exactly what that account can access**.

---

# 🧩 6. Authentication vs Authorization

|Concept|Question|Pentesting Goal|
|---|---|---|
|🔑 **Authentication**|Who are you?|Obtain or test credentials|
|🔐 **Authorization**|What are you allowed to access?|Identify excessive permissions|
|👤 **Anonymous Access**|Can you access without proving who you are?|Enumerate unauthenticated functionality|
|⚙️ **Misconfiguration**|Was the service configured securely?|Find unintended access paths|

### 🧠 Mental Model

```text
             SERVICE
                │
       ┌────────┴────────┐
       ▼                 ▼
 AUTHENTICATION      AUTHORIZATION
 "Who are you?"      "What can you do?"
       │                 │
       ▼                 ▼
 Credentials        Permissions
       │                 │
       └────────┬────────┘
                ▼
          Actual Access
```

---

# 🛡️ 7. Access Control Strategies

Administrators should deliberately define access rights instead of granting broad permissions.

Two approaches mentioned in the source are:

- **Role-Based Access Control (RBAC)**
    
- **Access Control Lists (ACLs)**
    

### 🧠 RBAC

Access is associated with a **role**.

```text
User
  ↓
Role
  ↓
Permissions
```

Example concept:

```text
Employee
   ↓
Employee Role
   ↓
Read Company Documents

Administrator
   ↓
Admin Role
   ↓
Read + Modify + Manage
```

### 🧠 ACL

Permissions are associated with specific users/groups and resources.

```text
Resource
   │
   ├── User A → Read
   ├── User B → Read/Write
   └── Admin  → Full Control
```

> [!tip]  
> From a pentesting perspective, the important thing is not memorizing the access-control model itself.
> 
> The key question is:
> 
> **"Does this account have more access than it should?"**

---

# ⚙️ 8. Unnecessary Defaults

Initial software/device configurations commonly contain defaults designed primarily for **usability**, not maximum security.

These may include:

- Settings
    
- Features
    
- Files
    
- Accounts
    
- Credentials
    
- Ports
    
- Services
    
- Documentation
    
- Sample applications
    

Leaving these defaults unchanged can unnecessarily increase the attack surface.

```text
Default Installation
        │
        ├── Unused Features
        ├── Unused Ports
        ├── Default Accounts
        ├── Sample Pages
        ├── Debugging
        └── Excessive Privileges
                 │
                 ▼
          Larger Attack Surface
```

> [!warning]  
> **Every enabled feature is another potential attack surface.**
> 
> A secure production system should expose only the functionality that is actually required.

---

# 🕵️ 9. What Attackers Look For

When encountering a newly discovered service, think:

```text
🔎 SERVICE DISCOVERED
        │
        ▼
🏷️ Identify Product + Version
        │
        ▼
🔑 Authentication
        │
        ├── Default credentials?
        ├── Weak credentials?
        ├── Blank password?
        └── Anonymous access?
        │
        ▼
🔐 Authorization
        │
        ├── What can current user read?
        ├── What can current user write?
        ├── What can current user modify?
        └── Can user access other users' data?
        │
        ▼
⚙️ Configuration
        │
        ├── Unnecessary features?
        ├── Admin interface?
        ├── Debug mode?
        ├── Directory listing?
        └── Informative errors?
        │
        ▼
💥 Attack Surface
```

---

# 🌐 10. Security Misconfiguration — OWASP Context

**Security Misconfiguration** is included in the **OWASP Top 10**.

The source highlights several examples of configuration problems:

- Unnecessary features are enabled or installed, including unnecessary ports, services, pages, accounts, or privileges.
    
- Default accounts and passwords remain enabled.
    
- Error handling exposes stack traces or overly informative error messages.
    
- Upgraded systems have newer security features disabled or configured insecurely.
    

> [!important]  
> This gives a useful enumeration mindset:
> 
> **Don't only search for vulnerabilities in the software itself. Search for dangerous configuration choices around the software.**

---

# 🛡️ 11. Preventing Service Misconfiguration

The general defensive strategy is:

> **Reduce the attack surface and allow only required behavior.**

Anything the application does not need should ideally be disabled.

The source recommends controls such as:

```text
🔒 Lock down critical infrastructure
          ↓
🚫 Disable unnecessary communication
          ↓
🚫 Disable admin interfaces when unnecessary
          ↓
🐞 Disable debugging
          ↓
🔑 Remove default credentials
          ↓
📂 Prevent unauthorized access / directory listing
          ↓
🔍 Regular scans + audits
```

### Important Hardening Actions

- Disable unnecessary administrative interfaces.
    
- Turn off debugging.
    
- Disable/remove default usernames and passwords.
    
- Prevent unauthorized access.
    
- Disable directory listing where unnecessary.
    
- Regularly scan and audit systems.
    
- Apply security fixes and configuration updates.
    

---

# 🏗️ 12. Secure Installation & Hardening Principles

The source highlights several principles for creating secure environments.

## 🔁 12.1 Repeatable Hardening

A secure configuration should be **repeatable**.

```text
Secure Baseline
      │
      ├── Development
      ├── QA
      └── Production
```

Development, QA, and production should be configured consistently, while using **different credentials** in each environment.

Automation helps minimize human error when deploying new systems.

---

## 🧹 12.2 Minimal Platform

Install only what is required.

Remove or avoid:

- Unused features
    
- Unused components
    
- Documentation not required on production systems
    
- Sample applications
    
- Unused frameworks
    

### Mental Model

```text
More Components
      ↓
More Attack Surface
      ↓
More Potential Vulnerabilities

Minimal Components
      ↓
Smaller Attack Surface
      ↓
Lower Risk
```

---

# 🔄 13. Configuration + Patch Management

Configuration security is not a one-time task.

Administrators should review and update configurations alongside:

- Security advisories
    
- Updates
    
- Patches
    
- Security notes
    
- Vulnerable/outdated component management
    

Cloud storage permissions should also be reviewed, such as **S3 bucket permissions**.

> [!warning]  
> A system can be patched and still be insecure because its **configuration** is wrong.

---

# 🧱 14. Segmented Application Architecture

Applications should be separated into appropriate security boundaries.

The source mentions:

- Segmentation
    
- Containerization
    
- Cloud security groups
    
- ACLs
    

```text
             APPLICATION
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   Component A Component B Component C
       │          │          │
       └───🔒 Security Boundaries ───┘
```

The goal is to prevent compromise of one component or tenant from automatically providing unrestricted access to others.

---

# 🌐 15. Security Directives

Security-related directives can also be sent to clients.

One example mentioned is:

> **Security headers**

These allow the server/application to communicate security-related behavior to clients.

---

# 🤖 16. Automated Configuration Verification

Manual configuration review is useful, but automated verification helps ensure that secure settings remain effective across environments.

```text
Secure Configuration
        ↓
Automated Verification
        ↓
Configuration Valid?
    ┌───┴───┐
   YES      NO
    │        │
    ▼        ▼
 Continue   Fix
```

> [!success]  
> The goal is not simply to configure a system securely once.
> 
> The goal is to make **secure configuration reproducible, testable, and continuously verifiable**.

---

# 🧭 17. CPTS Enumeration Checklist

When you discover a service, use this sequence:

|Priority|Question|What to Look For|
|---|---|---|
|1️⃣|🔎 What is the service?|Product, version, banner|
|2️⃣|🔑 Does it require authentication?|Login / anonymous access|
|3️⃣|🔐 Are default credentials possible?|Vendor defaults|
|4️⃣|🧪 Are weak credentials possible?|Common/blank passwords|
|5️⃣|👤 What does the account access?|Files, shares, records|
|6️⃣|📂 Are permissions excessive?|Read/write/admin access|
|7️⃣|⚙️ Are unnecessary features enabled?|Ports, services, pages, accounts|
|8️⃣|🐞 Is debugging enabled?|Debug output / stack traces|
|9️⃣|📋 Is directory listing enabled?|File/directory disclosure|
|🔟|🌐 Can data reach another service?|Local/network destinations|

---

# ⚡ 18. "I Found a Service — What Do I Do?"

```text
                 🔎 SERVICE
                     │
                     ▼
             Identify + Version
                     │
                     ▼
              🔐 AUTHENTICATION
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Default     Weak      Anonymous
       Creds       Creds       Access
          │          │          │
          └──────────┼──────────┘
                     ▼
              👤 AUTHORIZED?
                     │
                     ▼
             🔐 ACCESS RIGHTS
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
         Read       Write      Admin
          │          │          │
          └──────────┼──────────┘
                     ▼
               ⚙️ CONFIGURATION
                     │
          ┌──────────┼───────────┐
          ▼          ▼           ▼
       Defaults   Debugging   Unnecessary
       /Samples                Features
                     │
                     ▼
                💥 IMPACT
```

---

# 🧠 19. Key CPTS Takeaways

> [!success]
> 
> ### 🔑 Remember These Rules
> 
> **1. Authentication ≠ Authorization**
> 
> Getting a valid login does not mean the account is configured correctly.
> 
> **2. Always test defaults**
> 
> Especially for older applications, devices, and services.
> 
> **3. Test anonymous access**
> 
> A service may allow access without credentials even when authentication exists as an option.
> 
> **4. Enumerate permissions after login**
> 
> A low-privileged account may have access to highly sensitive information because of excessive permissions.
> 
> **5. Look for unnecessary defaults**
> 
> Unused ports, services, accounts, features, pages, and privileges increase the attack surface.
> 
> **6. Look for information disclosure**
> 
> Debugging, stack traces, directory listings, configuration files, and error messages may reveal useful information.
> 
> **7. Think attack surface**
> 
> Every unnecessary component or permission creates another potential path to compromise.
> 
> **8. Misconfiguration does not require a software vulnerability**
> 
> The application may be perfectly patched but still exploitable because it was configured insecurely.

---

# 🏆 20. One-Minute Cheat Sheet

```text
⚙️ SERVICE MISCONFIGURATION
│
├── 🔑 AUTHENTICATION
│   ├── Default credentials?
│   ├── Weak credentials?
│   ├── Blank passwords?
│   └── Anonymous access?
│
├── 🔐 AUTHORIZATION
│   ├── What can the account READ?
│   ├── What can it WRITE?
│   ├── What can it MODIFY?
│   └── Does it have excessive privileges?
│
├── ⚙️ DEFAULTS
│   ├── Unnecessary ports?
│   ├── Unnecessary services?
│   ├── Default accounts?
│   ├── Sample pages/files?
│   └── Unused features?
│
├── 🐞 INFORMATION DISCLOSURE
│   ├── Debugging enabled?
│   ├── Stack traces?
│   ├── Verbose errors?
│   └── Directory listing?
│
└── 💥 IMPACT
    ├── Sensitive files
    ├── Credentials
    ├── Other services
    ├── Privilege escalation
    └── Further access
```

> [!important]  
> **CPTS Mental Trigger:**
> 
> Whenever you see a service, don't immediately think **"Which exploit?"**
> 
> First think:
> 
> **🔎 Identify → 🔑 Authenticate → 🔐 Enumerate Permissions → ⚙️ Check Configuration → 📂 Find Sensitive Data → 💥 Determine Impact**
> 
> Many real-world compromises come from something that was simply **left enabled, left unchanged, or configured with too much access**.