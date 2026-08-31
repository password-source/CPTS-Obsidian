# 🌐 Catching Files over HTTP/S

> [!info]+ 📌 Why HTTP/S?  
> **HTTP/HTTPS** is one of the most common ways to transfer files because these protocols are frequently allowed through network firewalls.
> 
> Another major advantage is that **HTTPS encrypts data in transit**.
> 
> ```text
> 📄 Sensitive File
>       │
>       ▼
>    HTTP / HTTPS
>       │
>       ▼
> 🌐 Network
>       │
>       ▼
> 🖥️ Destination
> ```
> 
> > 🔐 **HTTPS** → encrypted in transit  
> > ⚠️ **HTTP** → plaintext in transit

> [!warning]+ 🚨 Why Encryption Matters  
> During a penetration test, transferring sensitive information over plaintext can trigger network security controls such as an IDS.
> 
> For example:
> 
> ```text
> Sensitive File
>      │
>      ▼
>    HTTP ❌
>      │
>      ▼
> 🛡️ Network IDS
>      │
>      ▼
> 🚨 Alert
> ```
> 
> Using encrypted transport such as **HTTPS** helps protect the contents while they are being transferred.

---

# 🧰 1️⃣ HTTP/S File Transfer Options

There are several ways to create an HTTP-based file-transfer server.

```text
             🌐 HTTP/S
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    🐍 Python  Apache   Nginx
    upload     HTTP      HTTP
    server     server    server
```

### Common Options

|Method|Purpose|
|---|---|
|🐍 Python `uploadserver`|Simple HTTP upload server|
|🪶 Apache|Full-featured web server|
|🟢 Nginx|Lightweight web server|
|🔐 HTTPS|Encrypted HTTP transport|

> [!tip]+ 💡 This Section  
> The focus here is creating an **Nginx-based HTTP upload server** using the HTTP `PUT` method.

---

# 🟢 2️⃣ Nginx — Enabling HTTP PUT

> [!info]+ 📌 Why Nginx?  
> Nginx is presented as an alternative to Apache for file-transfer operations because its configuration is relatively simple.
> 
> The material also notes that Apache's module system can make it easier to accidentally configure a server that executes uploaded files.

---

# ⚠️ 3️⃣ Important Upload Security Consideration

> [!danger]+ 🚨 Never Accidentally Create a Web Shell Upload Server  
> When accepting HTTP uploads, you must ensure that uploaded files **cannot be executed as web applications**.
> 
> This is particularly important with Apache + PHP configurations, where PHP files may be interpreted by the server.
> 
> ```text
> 📤 Upload
>    │
>    ▼
> malicious.php
>    │
>    ▼
> 🌐 Web Server
>    │
>    ├── ❌ Stored only
>    │
>    └── 🚨 Executed as PHP
> ```
> 
> For a file-transfer server, we generally want:
> 
> ```text
> 📤 Upload
>    │
>    ▼
> 💾 Store as a file
>    │
>    ✋ NOT execute
> ```

> [!success]+ 🛡️ Safer Design Principle  
> ==The upload directory should function as storage, not as an executable web application directory.==

---

# 📁 4️⃣ Create the Upload Directory

Create the directory that will hold uploaded files:

```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
```

### 🧠 Breakdown

|Part|Meaning|
|---|---|
|`sudo`|Execute with elevated privileges|
|`mkdir`|Create directory|
|`-p`|Create parent directories if necessary|
|`/var/www/uploads/...`|Upload destination|

### 📂 Result

```text
 /var/www/
    │
    └── uploads/
          │
          └── SecretUploadDirectory/
```

---

# 👤 5️⃣ Change Directory Ownership

Nginx commonly runs under the `www-data` account.

Set ownership:

```bash
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```

### 🔄 What This Does

```text
SecretUploadDirectory
          │
          ▼
      www-data
          │
          ├── 👤 User
          └── 👥 Group
```

> [!note]+ 🧠 Why?  
> The web server process needs appropriate permissions to write uploaded files into the directory.

---

# ⚙️ 6️⃣ Create the Nginx Configuration

Create:

```text
/etc/nginx/sites-available/upload.conf
```

Configuration:

```nginx
server {
    listen 9001;

    location /SecretUploadDirectory/ {
        root /var/www/uploads;
        dav_methods PUT;
    }
}
```

---

# 🧩 7️⃣ Understand the Nginx Configuration

```text
server
  │
  ├── listen 9001
  │       │
  │       └── 🌐 Listen on TCP port 9001
  │
  └── location /SecretUploadDirectory/
          │
          ├── root /var/www/uploads
          │
          └── dav_methods PUT
                    │
                    └── 📤 Allow HTTP PUT
```

### 📊 Configuration Breakdown

|Directive|Purpose|
|---|---|
|`server {}`|Defines a server configuration|
|`listen 9001`|Nginx listens on port `9001`|
|`location`|Defines how a URL path is handled|
|`root`|Defines the filesystem root|
|`dav_methods PUT`|Enables HTTP `PUT` requests|

---

# 🔗 8️⃣ Enable the Nginx Site

Create a symbolic link from `sites-available` to `sites-enabled`:

```bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
```

### 🧠 Nginx Configuration Structure

```text
/etc/nginx/
    │
    ├── sites-available/
    │      │
    │      └── upload.conf
    │
    └── sites-enabled/
           │
           └── upload.conf → 🔗 symlink
```

> [!tip]+ 💡 Remember
> 
> ```text
> sites-available = configurations available
> 
> sites-enabled = configurations Nginx actually uses
> ```

---

# 🔄 9️⃣ Restart Nginx

Apply the configuration:

```bash
sudo systemctl restart nginx.service
```

### 🧪 If Something Goes Wrong

Check:

```text
/var/log/nginx/error.log
```

A useful command:

```bash
tail -2 /var/log/nginx/error.log
```

---

# 🚨 🔟 Troubleshooting Port 80 Conflicts

> [!warning]+ ⚠️ Port Already in Use  
> On the HTB PwnBox environment, the example may produce an error because **port 80 is already occupied**.
> 
> Example:
> 
> ```text
> bind() to 0.0.0.0:80 failed
> (98: Address already in use)
> ```
> 
> This means another process is already listening on port `80`.

---

# 🔎 1️⃣1️⃣ Find What Is Using Port 80

Use:

```bash
ss -lnpt | grep 80
```

Example:

```text
LISTEN 0 100 0.0.0.0:80 0.0.0.0:*
users:(("python",pid=2811,...))
```

### 🧠 What We Learned

```text
Port 80
  │
  ▼
Already occupied
  │
  ▼
Find process
  │
  ▼
ss -lnpt
```

---

# 🔍 1️⃣2️⃣ Identify the Process

Once you have the PID, inspect it:

```bash
ps -ef | grep 2811
```

Example:

```text
user65  2811 ... python -m websockify 80 localhost:5901 -D
```

### 🔄 Troubleshooting Flow

```text
❌ Nginx won't start
        │
        ▼
Check error.log
        │
        ▼
"Address already in use"
        │
        ▼
Check port 80
        │
        ▼
ss -lnpt | grep 80
        │
        ▼
Find PID
        │
        ▼
ps -ef | grep <PID>
```

---

# 🧹 1️⃣3️⃣ Remove Default Nginx Configuration

The default Nginx configuration may itself bind to port `80`.

The material removes it with:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Then restart Nginx:

```bash
sudo systemctl restart nginx.service
```

> [!note]+ 🧠 Why?  
> Removing the default site prevents that default configuration from binding to port `80`, allowing the custom configuration to be used without the same conflict.

---

# 📤 1️⃣4️⃣ Upload Using cURL + HTTP PUT

Once Nginx is configured, `curl` can send a file using an HTTP `PUT` request.

Example:

```bash
curl -T /etc/passwd \
http://localhost:9001/SecretUploadDirectory/users.txt
```

### 🧠 `curl -T`

```text
curl
 │
 ├── -T
 │    │
 │    └── Upload file
 │
 └── URL
      │
      └── Destination
```

### 🔄 Transfer Flow

```text
🐧 Local Host
/etc/passwd
     │
     │ curl -T
     │ HTTP PUT
     ▼
Nginx :9001
     │
     ▼
/var/www/uploads/
SecretUploadDirectory/
     │
     ▼
users.txt
```

---

# ✅ 1️⃣5️⃣ Verify the Uploaded File

Check the file on the server:

```bash
sudo tail -1 /var/www/uploads/SecretUploadDirectory/users.txt
```

Example output:

```text
user65:x:1000:1000:,,,:/home/user65:/bin/bash
```

> [!success]+ ✅ Successful Upload  
> If the expected contents appear in the destination file, the HTTP `PUT` upload worked successfully.

---

# 👀 1️⃣6️⃣ Directory Listing — Why It Matters

> [!warning]+ 🚨 Don't Accidentally Expose Uploaded Files  
> A directory containing sensitive uploaded files should not automatically expose a listing of its contents.
> 
> The material suggests testing:
> 
> ```text
> http://localhost/SecretUploadDirectory
> ```
> 
> to ensure directory listing is not enabled.

### ❌ Bad Scenario

```text
/SecretUploadDirectory/
        │
        ▼
📋 Directory Listing
        │
        ├── users.txt
        ├── credentials.txt
        └── scan-results.txt
```

Anyone able to access that directory could potentially discover the transferred files.

### ✅ Desired Behavior

```text
/SecretUploadDirectory/
        │
        ▼
🚫 No Directory Listing
        │
        ▼
Files remain less exposed
```

> [!tip]+ 💡 Nginx Advantage  
> The material notes that Apache may list directory contents when no index file exists, while Nginx does not enable this behavior by default in the described setup.

---

# 🔐 1️⃣7️⃣ HTTP vs HTTPS

> [!important]+ 🌐 Don't Confuse HTTP and HTTPS
> 
> **HTTP**
> 
> ```text
> 📄 File
>   │
>   ▼
> HTTP
>   │
>   ▼
> 🌐 Plaintext Network
> ```
> 
> **HTTPS**
> 
> ```text
> 📄 File
>   │
>   ▼
> 🔐 HTTPS
>   │
>   ▼
> 🌐 Encrypted Network
> ```
> 
> HTTP is useful for reachability, but **HTTPS is preferred when confidentiality during transit is required**.

---

# 🧠 1️⃣8️⃣ Complete Nginx Upload Setup

```text
1️⃣ Create directory
        │
        ▼
mkdir /var/www/uploads/SecretUploadDirectory
        │
        ▼
2️⃣ Set ownership
        │
        ▼
chown www-data:www-data
        │
        ▼
3️⃣ Create upload.conf
        │
        ▼
listen 9001
        │
        ▼
dav_methods PUT
        │
        ▼
4️⃣ Enable configuration
        │
        ▼
sites-enabled/upload.conf
        │
        ▼
5️⃣ Restart Nginx
        │
        ▼
systemctl restart nginx
        │
        ▼
6️⃣ Test with curl
        │
        ▼
curl -T file http://server:9001/...
        │
        ▼
7️⃣ Verify uploaded file
        │
        ▼
📦 File received
```

---

# ⚡ 1️⃣9️⃣ Quick Command Reference

## 📁 Create Upload Directory

```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
```

## 👤 Change Ownership

```bash
sudo chown -R www-data:www-data \
/var/www/uploads/SecretUploadDirectory
```

## 🔗 Enable Site

```bash
sudo ln -s \
/etc/nginx/sites-available/upload.conf \
/etc/nginx/sites-enabled/
```

## 🔄 Restart Nginx

```bash
sudo systemctl restart nginx.service
```

## 🔎 Check Errors

```bash
tail -2 /var/log/nginx/error.log
```

## 🔍 Check Port 80

```bash
ss -lnpt | grep 80
```

## 🔎 Identify Process

```bash
ps -ef | grep <PID>
```

## 🧹 Remove Default Site

```bash
sudo rm /etc/nginx/sites-enabled/default
```

## 📤 Upload with PUT

```bash
curl -T <LOCAL_FILE> \
http://<SERVER>:9001/SecretUploadDirectory/<REMOTE_NAME>
```

## 📖 Verify File

```bash
sudo tail -1 \
/var/www/uploads/SecretUploadDirectory/<REMOTE_NAME>
```

---

# 🧩 2️⃣0️⃣ The Important Concepts

> [!success]+ 🧠 Remember These

> **🌐 HTTP/S**
> 
> HTTP/HTTPS is commonly permitted through firewalls, making web-based transfer useful in many environments.

> **📤 HTTP PUT**
> 
> `PUT` can be used to send a file to an HTTP server.

> **🟢 Nginx**
> 
> Nginx can be configured as a simple file-upload server using `dav_methods PUT`.

> **👤 Permissions**
> 
> The upload directory needs appropriate ownership/permissions for the web server.

> **🔎 Troubleshooting**
> 
> If Nginx fails to start, inspect:
> 
> ```text
> /var/log/nginx/error.log
> ```
> 
> and check whether another process already occupies the required port.

> **📋 Directory Listing**
> 
> Sensitive upload directories should not unnecessarily expose directory contents.

> **🔐 HTTPS**
> 
> When confidentiality is required, prefer encrypted transport rather than sending sensitive data over plaintext HTTP.

---

# 🏁 Final Takeaway

> [!quote]+ 💡 The Main Idea  
> ==HTTP/S is a powerful and commonly available file-transfer channel, but it must be configured carefully.==
> 
> ```text
>              📦 FILE TRANSFER
>                    │
>                    ▼
>                 🌐 HTTP/S
>                    │
>          ┌─────────┴─────────┐
>          ▼                   ▼
>       📥 Download          📤 Upload
>                              │
>                              ▼
>                           HTTP PUT
>                              │
>                              ▼
>                           🟢 Nginx
>                              │
>                              ▼
>                    /SecretUploadDirectory/
> ```
> 
> The important concepts are:
> 
> **HTTP/S → `PUT` → Nginx → upload directory → permissions → verify transfer**
> 
> And from a security perspective:
> 
> **🔐 Prefer HTTPS + 🚫 don't execute uploaded files + 👀 don't expose directory listings.**
> 
> The next major concept introduced by this section is **Living off the Land (LotL)** — using utilities already present on Windows or Linux systems to perform file-transfer operations.