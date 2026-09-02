
# 🔐 CPTS Password Attacks — Mark's Password Cheat Sheet

> [!important]  
> **Goal:** Build a **targeted wordlist** from Mark's OSINT → combine words to satisfy the 12-character policy → apply common password mutations → crack the hash with Hashcat.

---

# 🧠 1. Overall Methodology

```text
OSINT
  ↓
CeWL → Initial Wordlist
  ↓
Hashcat Combinator (-a 1)
  ↓
2-Word Combinations
  ↓
awk → Keep 12+ Character Candidates
  ↓
Custom Hashcat Rules
  ↓
Final Mutated Wordlist
  ↓
hashid → Identify Hash
  ↓
Hashcat → Crack
```

> [!tip]  
> **Mental model:**  
> **OSINT = WHAT words?**  
> **Combinator = WHICH words together?**  
> **Rules = HOW might they be modified?**  
> **Password policy = WHICH candidates are worth keeping?**

---

# 🌐 2. Step 1 — Create a Local Website for CeWL

Create `mark.html` containing **only useful personal OSINT**.

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

> [!warning]  
> Do **not** put password-policy terms such as `uppercase`, `symbol`, or `number` into the OSINT page. They are requirements, not likely password content.

Serve the directory containing `mark.html`:

```bash
python3 -m http.server 8000
```

The page becomes available at:

```text
http://<tun0>:8000/mark.html
```

---

# 🕷️ 3. Step 2 — Generate Initial Wordlist with CeWL

```bash
cewl -d 1 -m 2 --with-numbers -w mark_initial.txt http://<tun0>:8000/mark.html
```

### Options

|Option|Meaning|
|---|---|
|`-d 1`|Spider depth of 1|
|`-m 2`|Minimum word length = 2|
|`--with-numbers`|Include numeric values|
|`-w mark_initial.txt`|Save results to `mark_initial.txt`|

Check the size:

```bash
wc -l mark_initial.txt
```

Expected from the example:

```text
27
```

> [!important]  
> `--with-numbers` matters because information such as `1998` would otherwise be excluded.

---

# 🔗 4. Step 3 — Combine Words with Hashcat

Because the password policy requires a **minimum of 12 characters**, Mark may have combined multiple personal words.

Use Hashcat's **combinator attack mode**:

```bash
hashcat --stdout \
  -a 1 \
  mark_initial.txt \
  mark_initial.txt \
  > mark_pairs.txt
```

### 🧩 What `-a 1` Does

```text
Word A + Word B
      ↓
Combined Candidate
```

Using the same wordlist twice means every word can be combined with every other word, including itself.

Check the result:

```bash
wc -l mark_pairs.txt
```

Expected:

```text
729
```

> [!important]  
> **Hashcat attack mode `-a 1` = Combinator attack.**

---

# 📏 5. Step 4 — Filter for the 12-Character Policy

The company requires passwords to be **at least 12 characters**.

Filter the combinations:

```bash
awk 'length($0) >= 12' mark_pairs.txt > pairs_length12.txt
```

Check the result:

```bash
wc -l pairs_length12.txt
```

Expected:

```text
76
```

### 📉 Reduction

```text
729 candidates
      ↓
   length ≥ 12
      ↓
76 candidates
```

> [!tip]  
> Filtering **before** expensive mutation/cracking operations reduces unnecessary candidates.

---

# 🧩 6. Step 5 — Create Custom Hashcat Rules

Create:

```text
custom.rule
```

Contents:

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

### 🔑 Important Rules

|Rule|Meaning|
|---|---|
|`:`|Do nothing|
|`c`|Capitalize first letter|
|`so0`|Replace `o` → `0`|
|`sa@`|Replace `a` → `@`|
|`$!`|Append `!`|

The remaining rules combine these transformations.

---

# ⚙️ 7. Step 6 — Apply the Custom Rules

```bash
hashcat --stdout \
  -r custom.rule \
  pairs_length12.txt \
  > mut_mark_final.txt
```

Count the generated candidates:

```bash
cat mut_mark_final.txt | wc -l
```

Expected:

```text
1140
```

### 🔄 Full Wordlist Transformation

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

---

# 🧪 8. Step 7 — Identify the Hash

Target:

```text
97268a8ae45ac7d15c3cea4ce6ea550b
```

Use:

```bash
hashid -m 97268a8ae45ac7d15c3cea4ce6ea550b
```

Possible output includes:

```text
[+] MD2
[+] MD5 [Hashcat Mode: 0]
[+] MD4 [Hashcat Mode: 900]
[+] Double MD5 [Hashcat Mode: 2600]
```

The target is a **32-character hexadecimal hash**, making standard MD5 a strong candidate.

### 🧪 Validate MD5 Format

```bash
echo -n "test" | md5sum
```

Example:

```text
098f6bcd4621d373cade4e832627b4f6
```

MD5 produces:

```text
32 hexadecimal characters
```

Therefore:

```text
MD5 → Hashcat Mode 0
```

> [!important]  
> Never rely solely on hash length for identification. Use tools such as `hashid` and the surrounding context to determine the most likely hash type.

---

# 💥 9. Step 8 — Crack the Hash

Use Hashcat:

```bash
hashcat -a 0 -m 0 97268a8ae45ac7d15c3cea4ce6ea550b mut_mark_final.txt
```

### Options

|Option|Meaning|
|---|---|
|`-a 0`|Straight / wordlist attack|
|`-m 0`|MD5|
|`<hash>`|Target hash|
|`mut_mark_final.txt`|Final candidate list|

```text
Hash
 ↓
MD5 Mode (-m 0)
 ↓
Straight Attack (-a 0)
 ↓
mut_mark_final.txt
 ↓
Password Found
```

---

# 🧠 10. The Important Numbers to Remember

|Stage|File|Result|
|---|---|--:|
|🌐 CeWL|`mark_initial.txt`|27|
|🔗 Combinator|`mark_pairs.txt`|729|
|📏 Length filter|`pairs_length12.txt`|76|
|🧩 Custom rules|`mut_mark_final.txt`|1140|

```text
27
 ↓ × combinations
729
 ↓ length >= 12
76
 ↓ × mutations
1140
 ↓
Crack MD5
```

---

# ⚡ 11. One-Shot Command Cheat Sheet

### 🌐 Start HTTP server

```bash
python3 -m http.server 8000
```

### 🕷️ CeWL

```bash
cewl -d 1 -m 2 --with-numbers -w mark_initial.txt http://<tun0>:8000/mark.html
```

### 🔢 Count

```bash
wc -l mark_initial.txt
```

### 🔗 Combinator

```bash
hashcat --stdout -a 1 mark_initial.txt mark_initial.txt > mark_pairs.txt
```

### 📏 Filter 12+ characters

```bash
awk 'length($0) >= 12' mark_pairs.txt > pairs_length12.txt
```

### 🧩 Apply custom rules

```bash
hashcat --stdout -r custom.rule pairs_length12.txt > mut_mark_final.txt
```

### 🔢 Count final candidates

```bash
wc -l mut_mark_final.txt
```

### 🔎 Identify hash

```bash
hashid -m 97268a8ae45ac7d15c3cea4ce6ea550b
```

### 💥 Crack MD5

```bash
hashcat -a 0 -m 0 97268a8ae45ac7d15c3cea4ce6ea550b mut_mark_final.txt
```

---

# 🎯 12. CPTS Exam Mental Model

> [!success]
> 
> ### Remember the sequence:
> 
> ```text
> 1️⃣ Gather OSINT
>        ↓
> 2️⃣ Extract meaningful words with CeWL
>        ↓
> 3️⃣ Combine words with -a 1
>        ↓
> 4️⃣ Filter using known password requirements
>        ↓
> 5️⃣ Apply mutation rules
>        ↓
> 6️⃣ Identify hash type
>        ↓
> 7️⃣ Crack with the correct Hashcat mode
> ```
> 
> **CeWL → Combinator → Filter → Rules → Hash ID → Crack**
> 
> That's the core methodology demonstrated by this exercise.






---

# 🔐 1. Writing Custom Wordlists and Rules

> [!info]  
> **Core Concept:** Users often create passwords based on **simplicity rather than security**. Even when password policies enforce uppercase letters, lowercase letters, numbers, symbols, and minimum lengths, users tend to satisfy these requirements using **predictable patterns**.

Many organizations enforce password policies such as:

- 🔠 At least one uppercase letter
    
- 🔡 At least one lowercase letter
    
- 🔢 At least one number
    
- 🔣 At least one special character
    
- 📏 A minimum password length, commonly 8 characters or more
    

However, password policies do not necessarily make passwords unpredictable. Users commonly start with something meaningful and then make small modifications to satisfy the policy.

```text
Meaningful Word
      ↓
Capitalization
      ↓
Number / Year / Month
      ↓
Special Character
      ↓
Password satisfying the policy
```

> [!important]  
> **Password complexity does not necessarily mean password unpredictability.**
> 
> A password can contain uppercase letters, numbers, and special characters while still following a very predictable pattern.

---

# 🧠 2. Human Password-Creation Patterns

Users frequently base passwords on information that is personally meaningful or related to the organization they work for.

Common sources include:

- 🏢 Company names
    
- 🐶 Pet names
    
- 👨‍👩‍👧 Family members
    
- ⚽ Sports
    
- 🎮 Hobbies
    
- ❤️ Personal interests
    
- 🌎 Locations
    
- 🎂 Birth dates
    
- 📅 Years and months
    
- 🏭 Industry terminology
    

This is where **OSINT (Open Source Intelligence)** becomes useful. Information gathered about a target can provide likely words that may appear in their password.

For example:

```text
Target
 ├── Company
 ├── Name
 ├── Location
 ├── Pets
 ├── Family
 ├── Hobbies
 └── Important dates
          ↓
     Base Wordlist
          ↓
   Password Mutations
          ↓
   Targeted Candidates
```

> [!tip]  
> When performing targeted password guessing, think:
> 
> **"What information is meaningful to this user?"**
> 
> rather than simply trying random passwords.

---

# 🔢 3. Common Password Mutations

Users commonly modify familiar words in predictable ways to satisfy password policies.

## 📊 Common Password Patterns

|**Description**|**Password Syntax**|
|---|---|
|First letter is uppercase|`Password`|
|Adding numbers|`Password123`|
|Adding year|`Password2022`|
|Adding month|`Password02`|
|Last character is an exclamation mark|`Password2022!`|
|Adding special characters|`P@ssw0rd2022!`|

These patterns are important because we can automate them instead of manually generating every possible variation.

---

# 📏 4. Password Length and Predictability

Statistics referenced by the material indicate that most passwords are no longer than **10 characters**.

A common strategy is therefore to select familiar words that are at least five characters long and combine them with predictable additions.

For example:

```text
Current Month
     ↓
February
     ↓
February2022
     ↓
February2022!
```

Password expiration policies can also encourage predictable changes. Instead of creating an entirely new password, users may simply modify one component:

```text
January2025!
       ↓
February2025!
```

or:

```text
Password2025!
       ↓
Password2026!
```

> [!warning]  
> Regular password changes do not guarantee that users create completely new passwords. Users may simply modify an existing password in a predictable way.

---

# ⚙️ 5. Hashcat Rules

**Hashcat** can combine a list of potential words with mutation rules to automatically generate custom wordlists.

Hashcat uses a specific syntax to describe:

- Characters
    
- Words
    
- Transformations
    
- Character substitutions
    
- Added characters
    

The complete Hashcat rule syntax is documented in the official rule-based attack documentation.

> [!info]  
> **Mental Model:**
> 
> ```text
> Base Word + Rule = Candidate Password
> ```
> 
> Example:
> 
> ```text
> password + c      → Password
> password + so0    → passw0rd
> password + $!     → password!
> ```

---

# 🧩 6. Important Hashcat Rule Syntax

|**Function**|**Description**|
|---|---|
|`:`|Do nothing|
|`l`|Lowercase all letters|
|`u`|Uppercase all letters|
|`c`|Capitalize the first letter and lowercase others|
|`sXY`|Replace all instances of `X` with `Y`|
|`$!`|Add the exclamation character at the end|

### 🔍 Examples

```text
password
```

Using:

```text
c
```

produces:

```text
Password
```

Using:

```text
so0
```

produces:

```text
passw0rd
```

Using:

```text
sa@
```

produces:

```text
p@ssword
```

Using:

```text
$!
```

produces:

```text
password!
```

Rules can also be combined:

```text
c so0
```

produces:

```text
Passw0rd
```

---

# 📝 7. Creating a Custom Rule File

Each rule is written on a **new line**.

Example `custom.rule`:

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

Starting with:

```text
password
```

these rules generate multiple realistic password variations.

---

# 💻 8. Generating a Mutated Wordlist

Assume the base wordlist contains:

```text
password
```

We can apply the custom rules with:

```bash
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

### 🧩 Command Breakdown

|**Command**|**Purpose**|
|---|---|
|`hashcat`|Runs Hashcat|
|`--force`|Forces execution|
|`password.list`|Input wordlist|
|`-r custom.rule`|Applies the specified rule file|
|`--stdout`|Outputs generated candidates instead of performing cracking|
|`sort -u`|Sorts the output and removes duplicates|
|`>`|Redirects output into a file|
|`mut_password.list`|Generated wordlist|

The workflow is:

```text
password.list
     │
     ▼
custom.rule
     │
     ▼
Hashcat
     │
     ▼
Generated Candidates
     │
     ▼
sort -u
     │
     ▼
mut_password.list
```

---

# 📄 9. Resulting Mutated Wordlist

The single input word:

```text
password
```

produces fifteen mutated variants:

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
> A small base wordlist can therefore become a much larger and more useful candidate list by applying realistic password mutations.

---

# 🧰 10. Pre-Built Rule Sets

Both **Hashcat** and **John the Ripper (JtR)** provide pre-built rule lists that can be used for password generation and cracking.

One of the most effective and widely used Hashcat rule sets is:

```text
best64.rule
```

`best64.rule` applies common transformations that frequently result in successful password guesses.

> [!important]  
> Password cracking and custom wordlist generation are, in most cases, a **guessing game**.
> 
> The objective is to make the guesses as relevant as possible.

Instead of relying entirely on generic guesses, we can narrow the search using:

```text
Password Policy
      +
Company Information
      +
Geographical Region
      +
Industry
      +
Personal Information
      +
Hobbies / Interests
      ↓
Targeted Password Candidates
```

> [!warning]  
> An important exception is when passwords have already been leaked and obtained directly. In that case, guessing may not be necessary because the actual password may already be available.

---

# 🌐 11. Generating Wordlists Using CeWL

**CeWL (Custom Word List generator)** can spider a company's website and extract potential words into a wordlist.

This can be useful because company websites may contain words that employees could potentially use in passwords, such as:

```text
Company names
Product names
Locations
People
Projects
Industry terminology
Departments
Services
```

The resulting wordlist can then be combined with password mutation rules.

```text
Company Website
      ↓
     CeWL
      ↓
Company Wordlist
      ↓
Hashcat Rules
      ↓
Mutated Wordlist
      ↓
Password Cracking
```

---

# 🕷️ 12. Using CeWL

Example:

```bash
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
```

### 🧩 Command Breakdown

|**Option**|**Description**|
|---|---|
|`cewl`|Runs CeWL|
|`https://www.inlanefreight.com`|Target website|
|`-d 4`|Spider/crawl to a depth of 4|
|`-m 6`|Minimum word length of 6 characters|
|`--lowercase`|Stores discovered words in lowercase|
|`-w inlane.wordlist`|Writes the results to the specified wordlist|

We can check the number of generated entries with:

```bash
wc -l inlane.wordlist
```

Example output:

```text
326
```

This indicates that the generated wordlist contains **326 lines/entries**.

---

# 🎯 13. Targeted Password-Cracking Scenario

The exercise provides a compromised password hash belonging to a work email account for:

```text
Mark White
```

After performing OSINT, we have:

|**Category**|**Information**|
|---|---|
|👤 Name|Mark White|
|🎂 Date of birth|August 5, 1998|
|🏢 Company|Nexura, Ltd.|
|📏 Minimum password length|12 characters|
|🔠 Uppercase requirement|At least one uppercase letter|
|🔡 Lowercase requirement|At least one lowercase letter|
|🔣 Symbol requirement|At least one symbol|
|🔢 Number requirement|At least one number|
|📍 Location|San Francisco, CA, USA|
|🐱 Pet|Bella|
|👩 Wife|Maria|
|👦 Son|Alex|
|⚾ Interest|Baseball|

The compromised hash is:

```text
97268a8ae45ac7d15c3cea4ce6ea550b
```

---

# 🧠 14. Building a Target-Specific Wordlist

Rather than using only a generic wordlist, we can extract meaningful words from the information gathered about Mark.

Potential base words include:

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

Potential date-related components include:

```text
1998
98
05
08
0508
0805
```

These can then be transformed using common password patterns.

For example:

```text
bella
   ↓
Bella
   ↓
Bella1998
   ↓
Bella1998!
```

or:

```text
nexura
   ↓
Nexura
   ↓
Nexura1998
   ↓
Nexura1998!
```

The exact password should **not** be assumed from one pattern. The purpose of the custom rules is to generate many plausible variations efficiently.

---

# 📏 15. Using the Password Policy to Narrow the Search

Mark's company's password policy requires:

```text
Minimum 12 characters
At least 1 uppercase letter
At least 1 lowercase letter
At least 1 symbol
At least 1 number
```

This information is extremely valuable.

Instead of generating every possible mutation, prioritize candidates that satisfy the known requirements.

```text
                    Password
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Content      Structure     Policy
          │            │            │
          ▼            ▼            ▼
     Bella/Nexura   Capitalize   12+ chars
     Baseball/Alex  Add digits   Uppercase
     Maria/White    Add symbol   Lowercase
                                  Number
                                  Symbol
```

> [!important]  
> **OSINT tells you what the user may choose.**
> 
> **Password rules tell you how the user may modify it.**
> 
> **The password policy tells you what the final candidate must satisfy.**

---

# ⚔️ 16. Targeted Password-Cracking Methodology

The complete methodology can be viewed as:

```text
🔎 OSINT
   │
   ▼
📋 Identify meaningful words
   │
   ▼
📏 Identify password policy
   │
   ▼
🧠 Predict human password patterns
   │
   ▼
🧩 Create custom rules
   │
   ▼
⚙️ Generate candidate passwords
   │
   ▼
🎯 Crack the compromised hash
```

### Generic Approach

```text
Large Generic Wordlist
        ↓
Generic Rules
        ↓
Huge Candidate Set
```

### Targeted Approach

```text
OSINT
  ↓
Relevant Words
  ↓
Password Policy
  ↓
Custom Rules
  ↓
Targeted Candidate Set
```

> [!tip]  
> A **smaller but highly relevant** wordlist can be more useful than an enormous generic list when targeting a specific user.

---

# 📊 17. Generic vs Targeted Wordlists

|**Factor**|**Generic Wordlist**|**Targeted Wordlist**|
|---|---|---|
|🎯 Target knowledge|Little or none|Detailed OSINT|
|📚 Word sources|Common passwords|Target-specific information|
|🧠 Relevance|General|High|
|🧩 Rules|Generic mutations|Target-specific patterns|
|📈 Candidate quality|Lower against a specific target|Higher against a specific target|
|💾 Size|Often very large|Can be significantly smaller|
|⚔️ Best use|Unknown targets / broad attacks|Known users or organizations|

---

# 🧠 18. Key Mental Model

> [!success]
> 
> ### 🔑 Remember the Four-Part Model
> 
> ```text
> 🌐 OSINT
>    ↓
> WHAT WORDS WOULD THE USER CHOOSE?
> 
> 📏 Password Policy
>    ↓
> WHAT MUST THE PASSWORD CONTAIN?
> 
> 🧩 Hashcat/JtR Rules
>    ↓
> HOW WOULD THE USER MODIFY THE WORD?
> 
> 🎯 Hash Cracking
>    ↓
> DOES ONE OF THE GENERATED CANDIDATES MATCH?
> ```

The overall process is:

```text
OSINT
  +
Password Policy
  +
Human Behavior
  +
Mutation Rules
      ↓
Targeted Wordlist
      ↓
Password Cracking
```

> [!important]  
> **The key skill is not generating the largest possible number of passwords.**
> 
> The key skill is generating the **most probable passwords for the specific target**.

---

# 🎯 19. CPTS Key Takeaways

> [!success]
> 
> - 🔐 Users frequently choose passwords based on **personal information and familiar concepts**.
>     
> - 📏 Password policies can enforce complexity without guaranteeing unpredictability.
>     
> - 🧠 OSINT can provide valuable password candidates such as names, companies, locations, hobbies, and dates.
>     
> - ⚙️ Hashcat rules automate predictable password transformations.
>     
> - `:` → Do nothing.
>     
> - `l` → Lowercase all letters.
>     
> - `u` → Uppercase all letters.
>     
> - `c` → Capitalize the first letter and lowercase the others.
>     
> - `sXY` → Replace `X` with `Y`.
>     
> - `$!` → Append `!`.
>     
> - 💻 `hashcat --stdout` can generate mutated candidates without directly cracking a hash.
>     
> - 🧹 `sort -u` removes duplicate candidates.
>     
> - 🧰 `best64.rule` is a commonly used pre-built rule set.
>     
> - 🌐 CeWL can extract words from websites to create organization-specific wordlists.
>     
> - 🎯 Combining **CeWL + custom Hashcat rules + OSINT + password-policy information** creates highly targeted password candidates.
>     
> - 🧠 Always think about **how a human would construct the password**, not only what technically valid passwords exist.
>