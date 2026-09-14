# 🔐 1. Spraying, Stuffing, and Default Credentials

> [!info]  
> Not every credential attack requires trying thousands of passwords against one account.
> 
> Three important techniques are:
> 
> - 🎯 **Password Spraying** → One password → many accounts
>     
> - 🔄 **Credential Stuffing** → Many known username/password pairs → one or many services
>     
> - 🔑 **Default Credentials** → Known vendor/product credentials → target systems
>     
> 
> The key difference is **where the passwords come from and how they are applied**.

---

# 🧠 2. Password Spraying

**Password spraying** is a brute-force technique where an attacker attempts **one password against many different user accounts**.

Instead of:

```text
ONE USER
   ↓
password1
password2
password3
password4
...
```

we do:

```text
password = ChangeMe123!
        ↓
 ┌──────┼──────┬──────┬──────┐
 ↓      ↓      ↓      ↓      ↓
User1  User2  User3  User4  User5
```

> [!important]  
> The objective is to avoid aggressively testing many passwords against a single account while checking whether a commonly used password works for **multiple users**.

---

## 🎯 2.1 When Is Password Spraying Useful?

Password spraying is particularly useful when we suspect users were created with a **default or standardized password**.

For example, suppose an organization commonly initializes new accounts with:

```text
ChangeMe123!
```

Some users may never have changed that password.

We can therefore test:

```text
ChangeMe123!
       ↓
All discovered usernames
       ↓
Identify accounts still using it
```

This can be more effective than blindly trying thousands of passwords.

---

# 💻 3. Password Spraying with NetExec

[NetExec GitHub repository](https://github.com/Pennyw0rth/NetExec?utm_source=chatgpt.com)

In an Active Directory/SMB environment, NetExec can perform password spraying.

```bash
netexec smb 10.100.38.0/24 -u <usernames.list> -p 'ChangeMe123!'
```

### 🔎 Command Breakdown

```text
netexec
   ↓
smb
   ↓
10.100.38.0/24
   ↓
Target subnet
   ↓
-u <usernames.list>
   ↓
Multiple usernames
   ↓
-p 'ChangeMe123!'
   ↓
ONE password
```

> [!important]  
> The defining characteristic is:
> 
> **One password is tested against many usernames.**

---

# 🌐 4. Password Spraying Tools

The appropriate tool depends on the target.

|Target Environment|Example Tools|
|---|---|
|Web applications|`Burp Suite`|
|Active Directory / SMB|`NetExec`, `Kerbrute`|

[Burp Suite](https://portswigger.net/burp?utm_source=chatgpt.com)

[Kerbrute GitHub repository](https://github.com/ropnop/kerbrute?utm_source=chatgpt.com)

> [!tip]  
> **CPTS mindset:** First identify the authentication surface, then select the tool that understands that protocol/application.

---

# 🔄 5. Credential Stuffing

**Credential stuffing** is different from password spraying.

Instead of taking one password and testing it against many accounts, credential stuffing uses **previously obtained username/password combinations**.

Typical source:

```text
💥 Database Leak
      ↓
username:password
username:password
username:password
username:password
      ↓
Try the SAME credentials
against another service
```

The attack works because users frequently **reuse passwords across multiple services**.

For example:

```text
Email
   ↓
Social Media
   ↓
Enterprise Account
   ↓
VPN
   ↓
SSH
```

If the same credentials are reused, compromise of one service can lead to access to another.

> [!important]  
> **Credential stuffing does not mean guessing passwords.**
> 
> The credentials are already known or obtained from another source. The attack tests whether those credentials were **reused elsewhere**.

---

# 🧠 6. Password Spraying vs Credential Stuffing

|Technique|Password Source|Authentication Pattern|Example|
|---|---|---|---|
|**Password Spraying**|Common/guessed password|One password → many users|`ChangeMe123!` against many accounts|
|**Credential Stuffing**|Previously stolen credentials|Username/password pair → another service|`user:password` from a breach against SSH|

### 🧩 Mental Model

```text
🎯 PASSWORD SPRAYING

ONE PASSWORD
     ↓
┌────┼────┬────┬────┐
User1 User2 User3 User4
```

```text
🔄 CREDENTIAL STUFFING

Credential 1 ──→ Service
Credential 2 ──→ Service
Credential 3 ──→ Service
Credential 4 ──→ Service
```

---

# 💻 7. Credential Stuffing with Hydra

Suppose we have a file containing stolen credentials formatted as:

```text
username:password
```

For example:

```text
user1:password1
user2:password2
user3:password3
```

Hydra can use this format with the `-C` option.

```bash
hydra -C user_pass.list ssh://10.100.38.23
```

### 🔎 Important Option

```text
-C user_pass.list
```

means Hydra uses a **combined credential file** containing username/password pairs.

> [!tip]  
> Remember:
> 
> ```text
> -L → username list
> -P → password list
> -C → combined username:password credentials
> ```

---

# 🔑 8. Default Credentials

Many products are shipped with **default credentials**.

Common examples include:

- 🌐 Routers
    
- 🔥 Firewalls
    
- 🗄️ Databases
    
- 📡 Network devices
    
- 🖥️ Applications
    
- ⚙️ Management interfaces
    

The intended secure process is:

```text
Install Product
      ↓
Default Credentials
      ↓
Administrator Changes Them
      ↓
Unique Credentials
```

But sometimes:

```text
Install Product
      ↓
Default Credentials
      ↓
❌ Never Changed
      ↓
🎯 Potential Attack Path
```

> [!warning]  
> Default credentials can create a serious security weakness because the username/password may be publicly documented and therefore easily discoverable.

---

# 🔎 9. Finding Default Credentials

[Default Credentials Cheat Sheet](https://github.com/ihebski/DefaultCreds-cheat-sheet?utm_source=chatgpt.com)

Default credentials can be found through:

1. 📚 Vendor/product documentation
    
2. 🌐 Public default-credential databases
    
3. 🛠️ Dedicated tools
    
4. 🔎 Security research
    
5. 📋 Existing credential lists
    

Product documentation is particularly useful because installation/setup instructions often explain the initial authentication configuration.

Some products force the administrator to choose a password during installation, while others may retain a predefined default password.

---

# 🛠️ 10. DefaultCreds Cheat Sheet

The **Default Credentials Cheat Sheet** can be installed with `pip3`.

```bash
pip3 install defaultcreds-cheat-sheet
```

After installation, the `creds` command can search for known credentials associated with a product/vendor.

---

## 💻 10.1 Searching for Product Credentials

```bash
creds search linksys
```

Example output:

```text
+---------------+---------------+------------+
| Product       |    username   |  password  |
+---------------+---------------+------------+
| linksys       |    <blank>    |  <blank>   |
| linksys       |    <blank>    |   admin    |
| linksys       |    <blank>    | epicrouter |
| linksys       | Administrator |   admin    |
| linksys       |     admin     |  <blank>   |
| linksys       |     admin     |   admin    |
| linksys       |    comcast    |    1234    |
| linksys       |      root     |  orion99   |
| linksys       |      user     |  tivonpw   |
| linksys (ssh) |     admin     |   admin    |
| linksys (ssh) |     admin     |  password  |
| linksys (ssh) |    linksys    |  <blank>   |
| linksys (ssh) |      root     |   admin    |
+---------------+---------------+------------+
```

> [!important]  
> `<blank>` means the credential field is empty.
> 
> For example:
> 
> ```text
> admin : <blank>
> ```
> 
> means the username is `admin` and the password is blank.

---

# 🧠 11. Building a Custom Default-Credential List

During an assessment, suppose enumeration identifies several products:

```text
Product A
Product B
Product C
Product D
```

Research their known default credentials and combine the relevant results into a credential file.

Recommended format:

```text
username:password
```

Conceptually:

```text
Product A → admin:admin
Product B → root:password
Product C → user:1234
Product D → administrator:admin
                  ↓
          Combined Credential List
                  ↓
          Credential Stuffing
```

The resulting list can then be used with tools such as Hydra when the target protocol supports it.

> [!tip]  
> This demonstrates an important CPTS principle:
> 
> **Enumeration creates intelligence → intelligence creates a targeted credential list → the list is tested against relevant services.**

---

# 🌐 12. Router Default Credentials

Routers are another common source of default credentials.

A public list of router defaults can be used to identify likely:

- Default IP addresses
    
- Default usernames
    
- Default passwords
    

Although routers are critical infrastructure and are therefore less likely to retain their defaults, mistakes and oversights still occur.

Internal testing environments may be particularly prone to this because devices are sometimes deployed quickly with default configurations.

> [!warning]  
> Never assume that a default credential is active merely because it is documented. **Test it against the authorized target** and verify the product/version and authentication interface.

---

# 📊 13. Router Default Credentials

|**Router BrandDefault IP AddressDefault UsernameDefault Password**||||
|---|---|---|---|
|3Com|[http://192.168.1.1](http://192.168.1.1/)|admin|Admin|
|Belkin|[http://192.168.2.1](http://192.168.2.1/)|admin|admin|
|BenQ|[http://192.168.1.1](http://192.168.1.1/)|admin|Admin|
|D-Link|[http://192.168.0.1](http://192.168.0.1/)|admin|Admin|
|Digicom|[http://192.168.1.254](http://192.168.1.254/)|admin|Michelangelo|
|Linksys|[http://192.168.1.1](http://192.168.1.1/)|admin|Admin|
|Netgear|[http://192.168.0.1](http://192.168.0.1/)|admin|password|

---

# 🧠 14. CPTS Credential-Attack Mental Model

```text
                 🔎 ENUMERATE TARGET
                         ↓
                Identify users/products
                         ↓
             ┌───────────┼───────────┐
             ↓           ↓           ↓
       Common Password  Leaked     Defaults
                       Credentials
             ↓           ↓           ↓
        🎯 SPRAY       🔄 STUFF     🔑 DEFAULT
             │           │           │
             └───────────┼───────────┘
                         ↓
                  🔐 AUTHENTICATE
                         ↓
                📊 DETERMINE ACCESS
                         ↓
              💻 / 📂 / 🌐 / 🗄️
                         ↓
                 🔎 ENUMERATE AGAIN
```

---

# 📊 15. Quick Revision Cheat Sheet

|Attack|Core Idea|Typical Input|Example|
|---|---|---|---|
|🎯 **Password Spraying**|One password against many accounts|Username list + one/common password|`netexec smb <subnet> -u users.list -p 'ChangeMe123!'`|
|🔄 **Credential Stuffing**|Reuse stolen credentials against another service|`username:password` pairs|`hydra -C user_pass.list ssh://<target>`|
|🔑 **Default Credentials**|Test known vendor/product defaults|Product → known default credentials|`creds search <product>`|

---

# 🧠 16. The Three Techniques — One-Line Memory Aid

```text
🎯 SPRAY
"ONE password → MANY users"

🔄 STUFF
"MANY stolen pairs → ANOTHER service"

🔑 DEFAULT
"KNOWN vendor credentials → TARGET product"
```

> [!success]  
> **CPTS Key Takeaways**
> 
> - **Password spraying** tests one password across many accounts.
>     
> - **Credential stuffing** reuses previously obtained `username:password` pairs against another service.
>     
> - **Default credentials** exploit passwords that administrators failed to change after installation.
>     
> - Use **NetExec** for credential spraying against protocols such as SMB.
>     
> - Use **Hydra `-C`** when working with a combined `username:password` credential list.
>     
> - Use **DefaultCreds Cheat Sheet** to research product/vendor defaults.
>     
> - Always identify the **target product, version, protocol, and authentication interface** before blindly testing credentials.
>     
> - A successful authentication is only the beginning — always determine **what access the compromised account provides**.
>