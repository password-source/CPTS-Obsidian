---
tags: [cpts, tool, hashcat, cracking]
---

# 🧰 Hashcat

## 📦 Install
See [[Tool-Install]].

## 🎯 Purpose
GPU-accelerated offline password/hash cracking.

## 🧱 Core syntax
```bash
hashcat -m $MODE $HASH_FILE $WORDLIST
```

## 🚩 Useful modes (from these notes)
| Mode | Hash type |
|---|---|
| `5600` | NetNTLMv2 (LLMNR/NBT-NS captures) |
| `13100` | Kerberos 5, etype 23, TGS-REP (RC4) — Kerberoasting |
| `19700` | Kerberos 5, etype 18, TGS-REP (AES256) — Kerberoasting |

## 🔧 Variants
```bash
hashcat -m 5600 $HASH_FILE /usr/share/wordlists/rockyou.txt      # NetNTLMv2
hashcat -m 13100 $HASH_FILE /usr/share/wordlists/rockyou.txt     # Kerberoast RC4
hashcat -m 19700 $HASH_FILE /usr/share/wordlists/rockyou.txt     # Kerberoast AES256
```

In-session status while running:
```text
s        # print status
```

Unknown hash format → check the Hashcat "example hashes" reference page to identify the mode.

#hashcat #cpts
