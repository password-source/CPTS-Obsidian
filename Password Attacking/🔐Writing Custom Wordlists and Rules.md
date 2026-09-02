# 🔐  Targeted Wordlists & Custom Rules

> [!important]  
> **Core Goal:** Build a small, targeted password candidate list from **OSINT + password policy + human behavior**, then use **Hashcat rules** to generate likely password variations and crack the target hash.

---

# 🧠 1. Core Mental Model

```text
             TARGET
                │
                ▼
             🔎 OSINT
                │
                ▼
       Meaningful Base Words
                │
                ▼
       📏 Password Policy
                │
                ▼
       🧩 Custom Hashcat Rules
                │
                ▼
       Targeted Candidate List
                │
                ▼
          🎯 Hash Cracking
```

> [!tip]
> 
> ### Remember:
> 
> **OSINT → WHAT words?**  
> **Password Policy → WHAT must it contain?**  
> **Human Behavior → HOW might they modify it?**  
> **Hashcat Rules → Generate those modifications.**

The goal is **not** to create the biggest wordlist possible.

The goal is to create the **most probable candidates for the specific target**.

---

# 🔎 2. Why Targeted Password Attacks Work

Users often create passwords based on things that are meaningful to them:

- 👤 Names
    
- 🏢 Company names
    
- 🐶 Pets
    
- 👨‍👩‍👧 Family
    
- 📍 Locations
    
- ⚾ Sports
    
- 🎮 Hobbies
    
- 🎂 Birth dates
    
- 📅 Years/months
    
- 🏭 Industry terminology
    

Even strong password policies can still result in predictable passwords.

```text
Meaningful Word
      ↓
Capitalize
      ↓
Add Year
      ↓
Add Symbol
      ↓
Password satisfies policy
```

Example:

```text
bella
  ↓
Bella
  ↓
Bella1998
  ↓
Bella1998!
```

> [!warning]  
> **Password complexity ≠ password unpredictability.**
> 
> A password can contain uppercase letters, numbers, and symbols while still being highly predictable.

---

# 📏 3. Password Policy = Valuable Attack Information

If the target's policy is known, use it to reduce the search space.

Example:

```text
Minimum length: 12
Uppercase:      Required
Lowercase:      Required
Number:         Required
Symbol:         Required
```

Instead of generating random candidates:

```text
All Possible Passwords
        ↓
      HUGE
        ↓
     ❌ Slow
```

Use:

```text
OSINT
  +
Password Policy
  +
Human Patterns
  ↓
Relevant Candidates
```

> [!important]  
> Password requirements tell you **what the final password must look like**, while OSINT tells you **what content the user may have chosen**.

---

# 🧩 4. Common Human Password Patterns

Users commonly modify familiar words in predictable ways.

|Pattern|Example|
|---|---|
|Capitalize first letter|`Password`|
|Add numbers|`Password123`|
|Add year|`Password2022`|
|Add month|`Password02`|
|Add symbol|`Password2022!`|
|Character substitution|`P@ssw0rd2022!`|

Typical evolution:

```text
password
   ↓
Password
   ↓
Password2022
   ↓
Password2022!
   ↓
P@ssword2022!
```

Password expiration can produce similar behavior:

```text
Password2025!
      ↓
Password2026!
```

> [!warning]  
> Regular password changes do not necessarily create completely new passwords. Users may simply modify one predictable component.

---

# 🌐 5. OSINT → Build the Base Wordlist

The first step in a targeted attack is identifying information that could realistically appear in a password.

Example:

```text
Target: Mark White

Name:       Mark White
Company:    Nexura
Location:   San Francisco
Pet:        Bella
Wife:       Maria
Son:        Alex
Interest:   Baseball
DOB:        05/08/1998
```

Potential base words:

```text
mark
white
markwhite
nexura
bella
maria
alex
baseball
sanfrancisco
francisco
sf
```

Potential numeric components:

```text
1998
98
05
08
0508
0805
```

Mental model:

```text
Personal Information
        ↓
   Useful Words
        ↓
     Wordlist
        ↓
   Mutations
```

---

# 🕷️ 6. CeWL — Extract Words from Websites

**CeWL (Custom Word List generator)** spiders websites and extracts words that can be used to create a targeted wordlist.

```text
Company Website
      ↓
     CeWL
      ↓
Company-Specific Words
      ↓
Hashcat Rules
      ↓
Mutated Candidates
```

### Basic Example

```bash
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
```

### Important Options

|Option|Meaning|
|---|---|
|`-d 4`|Spider depth = 4|
|`-m 6`|Minimum word length = 6|
|`--lowercase`|Convert words to lowercase|
|`-w inlane.wordlist`|Save output to the specified file|

Check the wordlist:

```bash
wc -l inlane.wordlist
```

Example:

```text
326
```

> [!tip]  
> CeWL is especially useful when company terminology, products, projects, departments, or other organization-specific words may influence passwords.

---

# 🌐 7. Creating a Local OSINT Page for CeWL

If the relevant OSINT is already known, create a simple local webpage containing the useful information.

Example `mark.html`:

```html
<!DOCTYPE html>
<html>
<head><title>Mark White Bio</title></head>
<body>
  <h1>Mark White</h1>
  <ul>
    <li>Born: August 5, 1998</li>
    <li>Works at Nexura, Ltd.</li>
    <li>Lives in San Francisco, CA, USA</li>
    <li>Pet: cat named Bella</li>
    <li>Wife: Maria</li>
    <li>Son: Alex</li>
    <li>Fan of baseball</li>
  </ul>
</body>
</html>
```

Start a local HTTP server:

```bash
python3 -m http.server 8000
```

Then CeWL can access:

```text
http://<tun0>:8000/mark.html
```

> [!warning]  
> Do **not** put password-policy requirements such as `uppercase`, `symbol`, or `number` into the OSINT page.
> 
> These are **requirements**, not likely password content.

---

# 🕷️ 8. Generate the Initial Wordlist

```bash
cewl -d 1 -m 2 --with-numbers -w mark_initial.txt http://<tun0>:8000/mark.html
```

### Options

|Option|Meaning|
|---|---|
|`-d 1`|Spider depth = 1|
|`-m 2`|Minimum word length = 2|
|`--with-numbers`|Include numeric values|
|`-w mark_initial.txt`|Save the wordlist|

Check the size:

```bash
wc -l mark_initial.txt
```

Example:

```text
27
```

> [!important]  
> `--with-numbers` is useful when dates or years such as `1998` could become password components.

---

# 🔗 9. Hashcat Combinator Attack — `-a 1`

A user may combine multiple meaningful words to satisfy a minimum password length.

Hashcat's **combinator attack** joins words together.

```bash
hashcat --stdout \  -a 1 \  mark_initial.txt \
  mark_initial.txt \
  > mark_pairs.txt
```

### What `-a 1` Means

```text
-a 1
  ↓
Combinator Attack
```

Conceptually:

```text
Word A + Word B
      ↓
Combined Candidate
```

Using the same wordlist twice means words can be combined with every other word, including themselves.

Check the result:

```bash
wc -l mark_pairs.txt
```

Example:

```text
729
```

> [!important]  
> **Hashcat `-a 1` = Combinator attack.**

---

# 📏 10. Filter Candidates Using Password Length

If the policy requires **12+ characters**, remove shorter candidates before performing further processing.

```bash
awk 'length($0) >= 12' mark_pairs.txt > pairs_length12.txt
```

Check:

```bash
wc -l pairs_length12.txt
```

Example:

```text
76
```

### Reduction

```text
729 candidates
      ↓
 length >= 12
      ↓
76 candidates
```

> [!tip]  
> Filter as early as possible when you already know requirements. This reduces the number of candidates passed to later, more expensive operations.

---

# 🧩 11. Hashcat Rules

Hashcat rules modify words automatically.

```text
Base Word + Rule
      ↓
Candidate Password
```

Example:

```text
password + c
      ↓
Password
```

```text
password + so0
      ↓
passw0rd
```

```text
password + $!
      ↓
password!
```

Rules allow us to automate common human password modifications instead of manually creating every variation.

---

# 🔑 12. Important Hashcat Rule Syntax

|Rule|Meaning|
|---|---|
|`:`|Do nothing|
|`l`|Lowercase all letters|
|`u`|Uppercase all letters|
|`c`|Capitalize first letter and lowercase the rest|
|`sXY`|Replace `X` with `Y`|
|`$!`|Append `!`|

### Examples

Starting word:

```text
password
```

#### Capitalize

```text
c
```

→

```text
Password
```

#### Replace `o` → `0`

```text
so0
```

→

```text
passw0rd
```

#### Replace `a` → `@`

```text
sa@
```

→

```text
p@ssword
```

#### Append `!`

```text
$!
```

→

```text
password!
```

#### Combine transformations

```text
c so0
```

→

```text
Passw0rd
```

> [!tip]  
> **Rule chaining = multiple transformations applied to the same candidate.**

---

# 📝 13. Create a Custom Rule File

Create:

```text
custom.rule
```

Example:

```text
:
c
so0
c so0
sa@
c sa@
c sa@ so0
$!
$! c
$! so0
$! sa@
$! c so0
$! c sa@
$! so0 sa@
$! c so0 sa@
```

Each line represents a separate transformation.

Conceptually:

```text
password
   │
   ├── :
   ├── c
   ├── so0
   ├── c so0
   ├── sa@
   ├── c sa@
   ├── $!
   └── ...
```

---

# ⚙️ 14. Generate a Mutated Wordlist

Apply the rules without cracking a hash:

```bash
hashcat --stdout \
  -r custom.rule \
  pairs_length12.txt \
  > mut_mark_final.txt
```

Count the candidates:

```bash
wc -l mut_mark_final.txt
```

Example:

```text
1140
```

### Full Transformation

```text
mark_initial.txt
       │
       │ -a 1
       ▼
mark_pairs.txt
       │
       │ length >= 12
       ▼
pairs_length12.txt
       │
       │ custom.rule
       ▼
mut_mark_final.txt
       │
       ▼
1140 candidates
```

> [!important]  
> `hashcat --stdout` is useful for **generating candidate passwords** without actually performing the cracking operation.

---

# 🧹 15. Remove Duplicate Candidates

When generating wordlists, different transformations can sometimes produce the same candidate.

Use:

```bash
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

### Command Breakdown

|Command|Purpose|
|---|---|
|`hashcat`|Run Hashcat|
|`--force`|Force execution|
|`password.list`|Input wordlist|
|`-r custom.rule`|Apply custom rules|
|`--stdout`|Generate candidates only|
|`sort -u`|Sort + remove duplicates|
|`>`|Redirect output|
|`mut_password.list`|Output wordlist|

Workflow:

```text
password.list
      ↓
custom.rule
      ↓
Hashcat --stdout
      ↓
Generated Candidates
      ↓
sort -u
      ↓
mut_password.list
```

---

# 📄 16. Example Rule Output

Input:

```text
password
```

Possible generated candidates:

```text
password
Password
passw0rd
Passw0rd
p@ssword
P@ssword
P@ssw0rd
password!
Password!
passw0rd!
p@ssword!
Passw0rd!
P@ssword!
p@ssw0rd!
P@ssw0rd!
```

> [!success]  
> A small base wordlist can become a much more useful candidate set through realistic mutations.

---

# 🧰 17. Pre-Built Rules — `best64.rule`

Hashcat and JtR provide pre-built rule sets.

One commonly used Hashcat ruleset is:

```text
best64.rule
```

Instead of manually creating every transformation:

```text
Wordlist
   ↓
best64.rule
   ↓
Many common mutations
```

If you need to locate it:

```bash
locate best64.rule
```

or:

```bash
find /usr/share -name best64.rule 2>/dev/null
```

> [!important]  
> Use custom rules when you have **target-specific patterns**.
> 
> Use pre-built rules such as `best64.rule` when you want a broad set of common mutations.

---

# 🎯 18. Generic vs Targeted Wordlists

|Factor|Generic Wordlist|Targeted Wordlist|
|---|---|---|
|🎯 Target knowledge|Little/none|Detailed OSINT|
|📚 Word sources|Common passwords|Target-specific information|
|🧠 Relevance|General|High|
|🧩 Rules|Generic mutations|Target-specific patterns|
|📈 Candidate quality|Lower for a specific target|Higher for a specific target|
|💾 Size|Often very large|Can be much smaller|
|⚔️ Best use|Broad/unknown targets|Known users/organizations|

> [!tip]  
> **Smaller + relevant** can beat **huge + generic** when attacking a known target.

---

# 🧪 19. Identify the Hash

Target:

```text
97268a8ae45ac7d15c3cea4ce6ea550b
```

Use:

```bash
hashid -m 97268a8ae45ac7d15c3cea4ce6ea550b
```

Possible output:

```text
[+] MD2
[+] MD5 [Hashcat Mode: 0]
[+] MD4 [Hashcat Mode: 900]
[+] Double MD5 [Hashcat Mode: 2600]
```

The hash is:

```text
32 hexadecimal characters
```

MD5 is therefore a strong candidate in this scenario.

Validate the MD5 format:

```bash
echo -n "test" | md5sum
```

Example:

```text
098f6bcd4621d373cade4e832627b4f6
```

MD5 output:

```text
32 hexadecimal characters
```

Therefore:

```text
MD5
 ↓
Hashcat Mode 0
```

> [!warning]  
> **Do not identify a hash from length alone.**
> 
> Use `hashid`, hash format, context, and other available evidence to determine the most likely algorithm.

---

# 💥 20. Crack the Hash

Use:

```bash
hashcat -a 0 -m 0 \
  97268a8ae45ac7d15c3cea4ce6ea550b \
  mut_mark_final.txt
```

### Important Options

|Option|Meaning|
|---|---|
|`-a 0`|Straight / wordlist attack|
|`-m 0`|MD5|
|`<hash>`|Target hash|
|`mut_mark_final.txt`|Candidate wordlist|

Mental model:

```text
Target Hash
     ↓
-m 0 → MD5
     ↓
-a 0 → Wordlist Attack
     ↓
mut_mark_final.txt
     ↓
Password Match
```

> [!important]  
> **Hashcat `-a 0` = Straight/wordlist attack.**  
> **Hashcat `-m 0` = MD5.**

---

# 🎯 21. Complete Mark Methodology

```text
                    🔎 OSINT
                       │
                       ▼
              Meaningful Information
                       │
                       ▼
                🕷️ CeWL
                       │
                       ▼
             mark_initial.txt
                       │
                       │ -a 1
                       ▼
               mark_pairs.txt
                       │
                       │ length >= 12
                       ▼
            pairs_length12.txt
                       │
                       │ custom.rule
                       ▼
             mut_mark_final.txt
                       │
                       ▼
                  🔎 hashid
                       │
                       ▼
                 MD5 → -m 0
                       │
                       ▼
                 -a 0 Attack
                       │
                       ▼
                🎯 Password
```

---

# 📊 22. Important Numbers

|Stage|File|Example Count|
|---|---|--:|
|🌐 CeWL|`mark_initial.txt`|**27**|
|🔗 Combinator|`mark_pairs.txt`|**729**|
|📏 Length filter|`pairs_length12.txt`|**76**|
|🧩 Custom rules|`mut_mark_final.txt`|**1140**|

```text
27
 ↓ combinations
729
 ↓ length >= 12
76
 ↓ mutations
1140
 ↓
Crack
```

---

# ⚡ 23. CPTS Command Cheat Sheet

### 🌐 Start Local Server

```bash
python3 -m http.server 8000
```

### 🕷️ Generate Wordlist

```bash
cewl -d 1 -m 2 --with-numbers \
-w mark_initial.txt \
http://<tun0>:8000/mark.html
```

### 🔢 Count Words

```bash
wc -l mark_initial.txt
```

### 🔗 Combine Words

```bash
hashcat --stdout -a 1 \
mark_initial.txt mark_initial.txt \
> mark_pairs.txt
```

### 📏 Keep 12+ Characters

```bash
awk 'length($0) >= 12' \
mark_pairs.txt > pairs_length12.txt
```

### 🧩 Apply Custom Rules

```bash
hashcat --stdout \
-r custom.rule \
pairs_length12.txt \
> mut_mark_final.txt
```

### 🔢 Count Final Candidates

```bash
wc -l mut_mark_final.txt
```

### 🔎 Identify Hash

```bash
hashid -m 97268a8ae45ac7d15c3cea4ce6ea550b
```

### 💥 Crack MD5

```bash
hashcat -a 0 -m 0 \
97268a8ae45ac7d15c3cea4ce6ea550b \
mut_mark_final.txt
```

---

# 🧠 24. CPTS Exam Cheat Sheet

> [!success]
> 
> ### 🔑 The Sequence to Remember

```text
1️⃣ OSINT
   ↓
2️⃣ Extract useful words
   ↓
3️⃣ CeWL → Build wordlist
   ↓
4️⃣ -a 1 → Combine words
   ↓
5️⃣ Filter using password policy
   ↓
6️⃣ Create/apply Hashcat rules
   ↓
7️⃣ Generate targeted candidates
   ↓
8️⃣ hashid → Identify hash
   ↓
9️⃣ -a 0 + correct -m → Crack
```

### 🔥 Hashcat Modes to Remember

```text
-a 0 → Straight / Wordlist
-a 1 → Combinator
```

### 🔥 Important Rule Syntax

```text
:     → Do nothing
l     → Lowercase
u     → Uppercase
c     → Capitalize
sXY   → Substitute X with Y
$!    → Append !
```

### 🔥 Important Tools

```text
CeWL
  → Build targeted wordlists

Hashcat
  → Combine / mutate / crack

hashid
  → Identify likely hash type

awk
  → Filter candidates

sort -u
  → Remove duplicates
```

---

# 🎯 25. Final Mental Model

```text
          🔎 OSINT
             │
             ▼
     WHAT words are relevant?
             │
             ▼
     📏 Password Policy
             │
             ▼
     WHAT must it contain?
             │
             ▼
     🧠 Human Behavior
             │
             ▼
     HOW might the user modify it?
             │
             ▼
     🧩 Hashcat Rules
             │
             ▼
     Targeted Candidates
             │
             ▼
     🎯 Hash Cracking
```

> [!important]  
> **The key CPTS skill is not generating the largest possible wordlist.**
> 
> It is using **OSINT + password policy + human behavior + mutation rules** to generate the **most probable candidates for the specific target**.