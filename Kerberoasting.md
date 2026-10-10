---
tags: [cpts, setup, install]
---

# 🛠️ Tool Install Reference

> [!tip] Kali
> Most of these ship by default on Kali. Entries below still list apt/brew/choco/source for non-Kali boxes, reinstalling, or deploying to a bare Debian/macOS/Windows attack host.

## rpcclient / net / smbclient (Samba suite)
- **Kali default:** ✅ yes
```bash
sudo apt install samba-common-bin smbclient   # Debian/Kali
brew install samba                             # macOS
```
Check: `rpcclient --version`

---

## enum4linux
- **Kali default:** ✅ yes
```bash
sudo apt install enum4linux                    # Debian/Kali
git clone https://github.com/portcullislabs/enum4linux && cd enum4linux && tar -xzf ... # if needed from source
```
Check: `enum4linux -h | head -1`

## enum4linux-ng
- **Kali default:** ✅ yes (modern Kali)
```bash
sudo apt install enum4linux-ng
pipx install enum4linux-ng
git clone https://github.com/cddmp/enum4linux-ng && cd enum4linux-ng && pip install -r requirements.txt --break-system-packages
```
Check: `enum4linux-ng -h | head -1`

---

## Kerbrute
- **Kali default:** ❌ no (manual install)
```bash
go install github.com/ropnop/kerbrute@latest                     # go install
# or GitHub release:
curl -sSL -o kerbrute https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64
chmod +x kerbrute && sudo mv kerbrute /usr/local/bin/
```
```powershell
# Windows release
Invoke-WebRequest -Uri https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_windows_amd64.exe -OutFile kerbrute.exe
```
Check: `kerbrute --version`

---

## CrackMapExec (CME) / NetExec
- **Kali default:** ✅ yes (CME on older Kali; NetExec on current Kali)
```bash
sudo apt install crackmapexec                   # Debian/Kali (legacy)
pipx install git+https://github.com/Pennyw0rth/NetExec   # maintained fork
```
Check: `crackmapexec --version` / `nxc --version`

---

## ldapsearch (OpenLDAP client)
- **Kali default:** ✅ yes
```bash
sudo apt install ldap-utils                      # Debian/Kali
brew install openldap                             # macOS
```
Check: `ldapsearch -VV`

## windapsearch
- **Kali default:** ❌ no
```bash
git clone https://github.com/ropnop/windapsearch && cd windapsearch
pip install -r requirements.txt --break-system-packages
```
Check: `./windapsearch.py --help | head -1`

---

## Impacket (GetUserSPNs.py and friends)
- **Kali default:** ✅ yes
```bash
sudo apt install python3-impacket                 # Debian/Kali
pipx install impacket
git clone https://github.com/fortra/impacket && cd impacket && sudo python3 -m pip install .
```
Check: `GetUserSPNs.py -h | head -1`

---

## Hashcat
- **Kali default:** ✅ yes
```bash
sudo apt install hashcat                           # Debian/Kali
brew install hashcat                                 # macOS
choco install hashcat                                # Windows
```
Check: `hashcat --version`

---

## Responder
- **Kali default:** ✅ yes
```bash
sudo apt install responder                          # Debian/Kali
git clone https://github.com/lgandx/Responder
```
Check: `responder -h | head -1`

---

## Inveigh / InveighZero
- **Kali default:** ❌ no (Windows-side tool; attack hosts often stage it under `C:\Tools`)
```powershell
# Build from source (requires .NET / dotnet SDK on the build machine)
git clone https://github.com/Kevin-Robertson/InveighZero
# Or download compiled release from the repo's Releases page
```
Check: `.\Inveigh.exe` (banner on launch)

---

## PowerView (PowerSploit)
- **Kali default:** ❌ no (staged on Windows attack host)
```powershell
git clone https://github.com/PowerShellMafia/PowerSploit
Import-Module .\PowerView.ps1
```
Check: `Get-Command Get-DomainUser`

---

## Rubeus
- **Kali default:** ❌ no (compiled .NET, Windows-side)
```powershell
git clone https://github.com/GhostPack/Rubeus
# build with msbuild/Visual Studio, or grab a prebuilt release/binary from your toolkit
```
Check: `.\Rubeus.exe` (prints usage)

---

## Mimikatz
- **Kali default:** ❌ no (Windows-side; often flagged by AV/EDR — use from an authorized lab/engagement toolkit)
```powershell
git clone https://github.com/gentilkiwi/mimikatz
# or grab a release build
```
Check: `.\mimikatz.exe` (prints banner)

---

## DomainPasswordSpray
- **Kali default:** ❌ no
```powershell
git clone https://github.com/dafthack/DomainPasswordSpray
Import-Module .\DomainPasswordSpray.ps1
```
Check: `Get-Command Invoke-DomainPasswordSpray`

---

## kirbi2john.py
- **Kali default:** ❌ no (single script, part of the `kerberoast` repo)
```bash
git clone https://github.com/nidem/kerberoast
```
Check: `python2.7 kirbi2john.py --help` (requires Python 2.7)

---

## linkedin2username (optional, external OSINT)
- **Kali default:** ❌ no
```bash
git clone https://github.com/initstring/linkedin2username && cd linkedin2username
pip install -r requirements.txt --break-system-packages
```
Check: `python3 linkedin2username.py --help | head -1`

#tool-install #cpts
