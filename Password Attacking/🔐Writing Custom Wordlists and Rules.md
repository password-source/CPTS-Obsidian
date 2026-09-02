







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