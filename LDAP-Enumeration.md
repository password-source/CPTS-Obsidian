---
tags: [cpts, tool, kerbrute, kerberos]
---

# 🧰 Kerbrute

## 📦 Install
See [[Tool-Install]]. Also commonly staged at `C:\Tools\kerbrute.exe` on provided Windows attack hosts.

## 🎯 Purpose
Kerberos pre-auth based username enumeration and password spraying — doesn't trigger normal failed-logon events the way SMB auth does, since enumeration alone needs no password.

## 🧱 Core syntax
```bash
kerbrute userenum -d $DOMAIN --dc $DC_IP $USERLIST
kerbrute passwordspray -d $DOMAIN --dc $DC_IP $USERLIST $PASS
```

## 🚩 Useful flags
```text
userenum          # validate usernames via Kerberos pre-auth response
passwordspray     # spray one password against a user list
-d $DOMAIN        # target domain
--dc $DC_IP       # KDC / domain controller
```

## 🔧 Variants
```bash
# Username enumeration against a large candidate list
kerbrute userenum -d $DOMAIN --dc $DC_IP /opt/jsmith.txt

# Spray
kerbrute passwordspray -d $DOMAIN --dc $DC_IP $USERLIST $PASS
```

> [!warning] Enumeration ≠ spraying for lockout purposes
> `userenum` doesn't submit passwords (no lockout risk). `passwordspray` does — normal lockout rules apply.

#kerbrute #cpts
