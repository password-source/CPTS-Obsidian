---
tags: [cpts, tool, rpcclient, smb, samba]
---

# 🧰 rpcclient

## 📦 Install
Part of Samba. See [[Tool-Install]].

## 🎯 Purpose
Samba MS-RPC client — unauthenticated (NULL session) or credentialed SMB/RPC enumeration against a DC.

## 🧱 Core syntax
```bash
rpcclient -U "" -N $DC_IP          # anonymous / NULL session
rpcclient -U "$USER%$PASS" $DC_IP  # credentialed
```

## 🚩 Useful in-shell commands
```text
querydominfo       # domain info (user/group/alias counts, server role)
getdompwinfo        # password policy (min length, complexity flag)
enumdomusers        # list domain users + RIDs
```

## 🔧 Variants
```bash
# Password spray loop, filter on successful Authority Name
for u in $(cat $USERLIST); do rpcclient -U "$u%$PASS" -c "getusername;quit" $DC_IP | grep Authority; done
```

#rpcclient #cpts
