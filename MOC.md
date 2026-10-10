---
tags: [cpts, tool, enum4linux, smb]
---

# 🧰 enum4linux / enum4linux-ng

## 📦 Install
See [[Tool-Install]].

## 🎯 Purpose
Wrapper around Samba utilities (`smbclient`, `rpcclient`, `net`, `nmblookup`) for SMB/NetBIOS enumeration (users, groups, password policy, shares). `-ng` is the maintained Python rewrite with structured output.

## 🧱 Core syntax
```bash
enum4linux -U $DC_IP           # users
enum4linux -P $DC_IP           # password policy
enum4linux-ng -P $DC_IP -oA $OUTFILE
```

## 🚩 Useful flags
```text
-U     # enumerate users
-P     # enumerate password policy
-oA    # (ng only) output JSON + YAML + readable, base filename
```

## 🔧 Variants
```bash
# Clean username list
enum4linux -U $DC_IP | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"

# Password policy, structured output
enum4linux-ng -P $DC_IP -oA $OUTFILE
cat $OUTFILE.json
```

#enum4linux #cpts
