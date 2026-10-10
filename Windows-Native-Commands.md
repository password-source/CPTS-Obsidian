---
tags: [cpts, tool, crackmapexec, smb]
---

# 🧰 CrackMapExec (CME)

> [!warning] Status
> Upstream project archived Dec 2023. Still widely used in labs; consider **NetExec (nxc)** as the maintained fork — same syntax in most cases.

## 📦 Install
See [[Tool-Install]].

## 🎯 Purpose
SMB/Windows network enumeration, credential validation, and spraying framework.

## 🧱 Core syntax
```bash
crackmapexec smb $TARGET -u $USER -p $PASS
crackmapexec smb $TARGET -u $USER -H $NTHASH
```

## 🚩 Useful flags
```bash
--pass-pol                       # dump password policy
--users                          # enumerate domain users (+ badpwdcount/baddpwdtime)
--local-auth                     # authenticate as LOCAL account, not domain (critical for local-admin hash reuse)
-u $USERLIST -p $PASS            # password spray
-u $USER -H $NTHASH              # pass-the-hash
```

## 🔧 Variants
```bash
# Password policy
crackmapexec smb $DC_IP -u $USER -p $PASS --pass-pol

# Domain user enumeration
crackmapexec smb $DC_IP -u $USER -p $PASS --users

# Spray username file against one password, show hits only
sudo crackmapexec smb $DC_IP -u $USERLIST -p $PASS | grep +

# Validate a single credential
sudo crackmapexec smb $DC_IP -u $USER -p $PASS

# Local admin hash reuse across a subnet
sudo crackmapexec smb --local-auth $SUBNET/24 -u administrator -H $NTHASH | grep +
```

#crackmapexec #cpts
