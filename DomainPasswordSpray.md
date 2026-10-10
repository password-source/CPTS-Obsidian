---
tags: [cpts, tool, windows-native, net, setspn]
---

# 🧰 Windows Native Commands (net / setspn)

## 📦 Install
Built into Windows — no install needed.

## 🎯 Purpose
No-tool-transfer enumeration: NULL sessions, password policy, and SPN discovery using only built-in binaries.

## 🧱 Core syntax
```cmd
net use \\$DC_HOSTNAME\ipc$ "" /u:""
net accounts
setspn.exe -Q */*
```

## 🚩 Useful variants
```cmd
net use \\$DC_HOSTNAME\ipc$ "" /u:""              REM NULL IPC$ session
net use \\$DC_HOSTNAME\ipc$ "" /u:$USER            REM test specific account, blank password
net use \\$DC_HOSTNAME\ipc$ "$PASS" /u:$USER       REM test specific credential

net accounts                                       REM local/domain password + lockout policy

setspn.exe -Q */*                                  REM all SPNs in domain
setspn.exe -T $DOMAIN -Q */*                        REM scope to a specific domain
```

## 🧠 `net use` error codes (account-state signal)
| Error | Meaning |
|---|---|
| `1331` | Account exists but is disabled |
| `1326` | Username or password incorrect |
| `1909` | Account currently locked out |

#windows-native #net #setspn #cpts
