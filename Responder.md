---
tags: [cpts, tool, ldap, ldapsearch, windapsearch]
---

# 🧰 LDAP Enumeration — ldapsearch / windapsearch

## 📦 Install
See [[Tool-Install]].

## 🎯 Purpose
Anonymous or credentialed LDAP queries against AD for users, computers and password-policy attributes.

## 🧱 Core syntax — ldapsearch
```bash
ldapsearch -H ldap://$DC_IP -x -b "DC=$DOMAIN,DC=LOCAL" -s sub "$FILTER"
```
> Modern `ldapsearch` deprecates `-h` in favor of `-H ldap://...`.

## 🧱 Core syntax — windapsearch
```bash
./windapsearch.py --dc-ip $DC_IP -u "" -U
```

## 🚩 Useful flags
```text
-x                        # simple auth (not SASL)
-b "DC=...,DC=..."        # search base
-s sub                    # search subtree
"(&(objectclass=user))"   # user objects filter
--dc-ip                   # (windapsearch) target DC
-u ""                     # (windapsearch) blank = anonymous bind
-U                        # (windapsearch) enumerate users
```

## 🔧 Variants
```bash
# Clean username list
ldapsearch -H ldap://$DC_IP -x -b "DC=$DOMAIN,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "

# Password policy attributes
ldapsearch -H ldap://$DC_IP -x -b "DC=$DOMAIN,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

Key attributes: `minPwdLength`, `lockoutThreshold`, `lockoutDuration`, `lockOutObservationWindow`, `pwdProperties`, `pwdHistoryLength`. Entries ending `$` = computer accounts.

#ldap #ldapsearch #windapsearch #cpts
