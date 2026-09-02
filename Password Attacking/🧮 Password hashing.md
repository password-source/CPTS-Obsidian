# 🔐 1. Introduction to Password Cracking

> [!info] 🧠 Core Concept  
> Passwords are commonly stored as **hashes** rather than plaintext. Hashing transforms arbitrary input data into a typically fixed-size output.
> 
> During a penetration test, if password hashes are obtained, the goal is often to determine the original password through **password cracking**.

---

## 🧮 2. What Is Password Hashing?

A **hash function** takes input data and produces a deterministic hash value:

```text
Password
   │
   ▼
┌──────────────┐
│ Hash Function│
└──────────────┘
   │
   ▼
Fixed-size Hash
```

For example:

```text
Password: Soccer06!

MD5:
40291c1d19ee11a7df8495c4cccefdfa

SHA-256:
a025dc6fabb09c2b8bfe23b5944635f9b68433ebd9a1a09453dd4fee00766d93
```

These can be generated with:

```bash
echo -n Soccer06! | md5sum
echo -n Soccer06! | sha256sum
```

### 🔑 Important Properties

|Property|Meaning|
|---|---|
|🔄 Deterministic|Same input → same hash|
|➡️ One-way|Hash should not be practically reversible|
|📏 Fixed-size|Output is generally a fixed length for a given algorithm|
|⚡ Fast|Many traditional hash functions can be calculated extremely quickly|
|💥 Crackable|An attacker can guess passwords and compare their hashes|

> [!warning] ⚠️ Hashing ≠ Encryption  
> **Encryption** is designed to be reversible with the appropriate key.
> 
> **Hashing** is designed to be one-way.
> 
> Therefore, attackers generally do **not decrypt a hash**. They **guess passwords, hash those guesses, and compare the resulting hashes**.

---

# 🎯 3. What Is Password Cracking?

**Password cracking** is the process of attempting to determine the original password represented by a password hash.

The attacker generally follows this process:

```text
             Obtained Hash
                  │
                  ▼
        ┌───────────────────┐
        │ Generate Password │
        │      Guess        │
        └─────────┬─────────┘
                  │
                  ▼
            Hash the Guess
                  │
                  ▼
       Compare With Target Hash
             │           │
          Match       No Match
             │           │
             ▼           └──────► Try Again
        🎯 Password
```

### 🧠 Mental Model

> **You don't reverse the hash. You search for an input that produces the same hash.**

Common password-cracking approaches include:

1. 🌈 **Rainbow tables**
    
2. 📖 **Dictionary / wordlist attacks**
    
3. 💥 **Brute-force attacks**
    
4. 🎭 **Mask attacks** — often more practical than unrestricted brute force
    

---

# 🌈 4. Rainbow Tables

A **rainbow table** is a large precomputed mapping of likely inputs to their corresponding hashes.

Instead of calculating hashes during the attack, an attacker can search the precomputed database.

### Example

|Password|MD5 Hash|
|---|---|
|`123456`|`e10adc3949ba59abbe56e057f20f883e`|
|`12345`|`827ccb0eea8a706c4c34a16891f84e7b`|
|`password`|`5f4dcc3b5aa765d61d8327deb882cf99`|
|`iloveyou`|`f25a2fc72690b780b2a14e140ef6a9e0`|
|`princess`|`8afa847f50a716e64932d995c8e7435a`|
|`12345678`|`25d55ad283aa400af464c76d713c07ad`|

If the target hash is:

```text
5f4dcc3b5aa765d61d8327deb882cf99
```

The attacker can look it up and discover:

```text
Hash
 │
 ▼
5f4dcc3b5aa765d61d8327deb882cf99
 │
 ▼
password
```

> [!important] 🎯 Why Rainbow Tables Are Fast  
> The expensive hashing work was performed **before the attack**.
> 
> The attacker mainly performs lookups rather than calculating every candidate from scratch.

---

# 🧂 5. Salting Password Hashes

Rainbow tables become much less effective when passwords are **salted**.

A **salt** is a random sequence of bytes added to a password before hashing.

```text
Password
   +
Random Salt
   │
   ▼
Combined Input
   │
   ▼
Hash Function
   │
   ▼
Password Hash
```

For example:

```text
Password:
Soccer06!

Salt:
Th1sIsTh3S@lt_

Combined:
Th1sIsTh3S@lt_Soccer06!
```

The resulting MD5 hash is:

```text
90a10ba83c04e7996bc53373170b5474
```

Command:

```bash
echo -n Th1sIsTh3S@lt_Soccer06! | md5sum
```

### 🧠 Why Does Salting Work?

Without a salt:

```text
Soccer06!
    │
    ▼
   MD5
    │
    ▼
40291c1d...
```

A rainbow table could already contain that mapping.

With a unique salt:

```text
Th1sIsTh3S@lt_ + Soccer06!
              │
              ▼
             MD5
              │
              ▼
90a10ba8...
```

The attacker now needs a mapping for **that specific salt + password combination**.

---

# 🧂 6. Salts Are NOT Secrets

A common misconception is that the salt must be hidden.

It doesn't.

The authentication system needs to know the salt so it can reproduce the hashing process when checking a password.

Therefore:

```text
Salt → Public / Stored
Hash → Stored
Password → Secret
```

> [!important] 🔑 Key Point  
> **A salt is not a password and does not need to be secret.**
> 
> Its purpose is to make precomputed attacks such as rainbow tables impractical.

---

# 🧂 7. Why Salts Destroy Rainbow Tables

Suppose an attacker has a rainbow table containing:

```text
15 billion password → hash mappings
```

If a salt can have **256 possible values** because it is only one byte:

```text
15 billion × 256
        │
        ▼
3.84 trillion mappings
```

So the attacker would need to account for every possible salt.

With properly generated longer random salts, precomputing every possible combination becomes infeasible.

### 📊 Unsalted vs Salted

|Feature|Unsalted|Salted|
|---|---|---|
|Rainbow tables|🔴 Highly effective|🟢 Largely defeated|
|Identical passwords|Produce identical hashes|Produce different hashes with unique salts|
|Precomputation|Practical|Extremely difficult|
|Salt secret?|N/A|❌ No|
|Salt should be unique?|N/A|✅ Yes|

> [!success] 🧠 Mental Model  
> **Rainbow tables attack the predictability of password → hash mappings.**
> 
> **Salting destroys that predictability by changing the input for each password.**

---

# 💥 8. Brute-Force Attacks

A **brute-force attack** attempts every possible combination of characters until the correct password is found.

For a character set containing:

```text
abcdefghijklmnopqrstuvwxyz
ABCDEFGHIJKLMNOPQRSTUVWXYZ
0123456789
!@#$%^&*...
```

the attacker systematically generates candidates:

```text
aaaaa
aaaab
aaaac
...
Sxejd
Sxeje
Sxejf
Sxejg
...
```

Each candidate is hashed and compared with the target:

```text
Candidate
    │
    ▼
Hash Function
    │
    ▼
Candidate Hash
    │
    ├─────── Match ──────► 🎯 Password Found
    │
    └──── No Match ──────► Next Candidate
```

---

# 📈 9. Brute-Force Complexity

The problem with brute force is the **size of the search space**.

If:

- `C` = number of possible characters
    
- `L` = password length
    

Then the number of possible passwords is approximately:

```text
C^L
```

For example, increasing password length dramatically increases the search space.

> [!warning] ⚠️ Why Password Length Matters  
> Brute force becomes exponentially more expensive as password length increases.
> 
> This is why unrestricted brute force is often inefficient against strong passwords.

---

# 🎯 10. Why Brute Force Is Technically 100% Effective

Brute force is considered **100% effective in theory** because, given:

- the correct character set,
    
- unlimited time,
    
- sufficient computational resources,
    

the attacker will eventually test the correct password.

```text
Every Possible Password
          │
          ▼
 ┌─────────────────┐
 │ Test Candidate  │
 └────────┬────────┘
          │
          ▼
      Correct?
       /    \
     No      Yes
     │        │
     ▼        ▼
 Continue    🎯
```

However:

> [!important] ⏱️ Theoretical ≠ Practical  
> A technique can be guaranteed to work while still being completely impractical because of the amount of time and computational power required.

Short passwords can be viable targets, while long and complex passwords can make unrestricted brute force prohibitively expensive.

---

# 🎭 11. Mask Attacks

Because unrestricted brute force wastes time testing unrealistic combinations, attackers often use **mask attacks**.

A mask attack applies knowledge about the expected password structure.

For example, if you know a password is likely to follow:

```text
4 lowercase letters + 2 digits + !
```

you can restrict the search to that structure instead of testing every possible password.

Conceptually:

```text
Unrestricted Brute Force
        ↓
[Everything]

Mask Attack
        ↓
[Known Pattern]
        ↓
aaaa00!
aaaa01!
aaaa02!
...
zzzz99!
```

> [!tip] 🧠 CPTS Perspective  
> When you have information about password construction, **use that information to reduce the search space**.
> 
> This is the fundamental idea behind masks and other optimized cracking strategies.

---

# ⚡ 12. Cracking Speed Depends on the Hash

Different hashing algorithms have dramatically different cracking speeds.

Example given in the material:

|Hash Type|Approx. Rate on Typical Company Laptop|
|---|--:|
|MD5|> **5 million guesses/sec**|
|DCC2|~ **10,000 guesses/sec**|

The exact speed depends on:

- 🖥️ Hardware
    
- 🔐 Hashing algorithm
    
- ⚙️ Hash configuration
    
- 🧮 Number of computational operations required
    
- 🛠️ Cracking tool
    

### 🧠 Critical Mental Model

```text
Fast Hash
   │
   ▼
More guesses / second
   │
   ▼
Password cracking is easier

Slow / expensive Hash
   │
   ▼
Fewer guesses / second
   │
   ▼
Password cracking is harder
```

> [!important] 🔥 Hash Algorithm Matters  
> You cannot estimate password-cracking difficulty from password complexity alone.
> 
> **The hashing algorithm and its configuration have a major impact on how quickly guesses can be tested.**

---

# 📖 13. Dictionary / Wordlist Attacks

A **dictionary attack**, also called a **wordlist attack**, uses a list of likely passwords instead of generating every possible combination.

Instead of:

```text
aaaaaa
aaaaab
aaaaac
...
```

the attacker tests statistically likely passwords:

```text
123456
password
qwerty
iloveyou
monkey
...
```

### 🔄 Attack Flow

```text
             Wordlist
                │
                ▼
       ┌─────────────────┐
       │ Read Candidate  │
       └────────┬────────┘
                │
                ▼
          Hash Candidate
                │
                ▼
       Compare Target Hash
           /          \
       Match        No Match
         │              │
         ▼              ▼
      🎯 Found      Next Word
```

---

# 📚 14. Why Dictionary Attacks Are So Effective

Humans are not random password generators.

Users commonly choose:

- Common words
    
- Names
    
- Dates
    
- Sports teams
    
- Places
    
- Keyboard patterns
    
- Common passwords
    
- Passwords from previous breaches
    

Therefore, a well-built wordlist can eliminate enormous amounts of unnecessary search space.

> [!success] 🎯 Pentesting Mental Model  
> During a real engagement, time is limited.
> 
> **Don't immediately start with the largest possible search space.**
> 
> Start with the highest-probability candidates and progressively increase complexity if necessary.

---

# 📂 15. Common Password Wordlists

Two commonly encountered sources are:

### 🪨 rockyou.txt

`rockyou.txt` is one of the most famous password wordlists used in penetration testing.

Example:

```bash
head --lines=20 /usr/share/wordlists/rockyou.txt
```

Example contents:

```text
123456
12345
123456789
password
iloveyou
princess
1234567
rockyou
12345678
abc123
nicole
daniel
babygirl
monkey
lovely
jessica
654321
michael
ashley
qwerty
```

`rockyou.txt` contains **over 14 million real passwords** originating from a 2009 RockYou breach.

> [!warning] 🔐 Historical Security Lesson  
> The RockYou breach became especially notable because the company stored users' passwords **without proper encryption/hashing protections**, exposing plaintext passwords.

### 🛡️ SecLists

**SecLists** is another widely used collection containing security-related wordlists, including password lists.

Common location:

```text
/usr/share/seclists/
```

---

# ⚔️ 16. Comparing Password-Cracking Techniques

|Technique|Main Idea|Speed|Storage|Best Use|
|---|---|--:|--:|---|
|🌈 Rainbow Table|Precomputed password → hash mappings|⚡ Very fast lookup|🔴 Very large|Unsalted hashes|
|📖 Dictionary|Test likely passwords|⚡ Fast|🟢 Wordlist required|Common passwords|
|💥 Brute Force|Test every combination|🐢 Potentially very slow|🟢 Low|Short/limited search spaces|
|🎭 Mask|Test combinations matching known patterns|⚡ More efficient than unrestricted brute force|🟢 Low|Known password structure|

---

# 🧠 17. CPTS Password-Cracking Methodology

A useful way to think about password cracking during an engagement is:

```text
              🔐 Obtain Hash
                    │
                    ▼
             Identify Hash Type
                    │
                    ▼
             Is it Salted?
              /           \
            Yes            No
             │              │
             ▼              ▼
      Rainbow tables      Consider
       less useful       precomputation
             │
             └───────┬───────────┘
                     ▼
              Try Wordlists
                     │
                     ▼
          Apply Known Information
                     │
                     ▼
               Mask Attack
                     │
                     ▼
           Brute Force if Needed
```

### 🎯 Practical Escalation

```text
1️⃣ Identify the hash
        ↓
2️⃣ Determine whether it is salted
        ↓
3️⃣ Use likely passwords / wordlists
        ↓
4️⃣ Apply transformations / patterns
        ↓
5️⃣ Use masks when password structure is known
        ↓
6️⃣ Resort to broader brute force when justified
```

> [!tip] 🧠 Think Like a Penetration Tester  
> The objective isn't simply:
> 
> **"Can I eventually crack this password?"**
> 
> The better question is:
> 
> **"What is the most efficient way to reduce the search space using everything I already know?"**

---

# 🔑 18. Key Takeaways

> [!important] 🧠 CPTS Exam Memory

- 🔐 **Hashing** transforms input into a fixed-size representation and is intended to be **one-way**.
    
- 🔓 Password cracking does **not** reverse the hash; it searches for an input that produces the same hash.
    
- 🌈 **Rainbow tables** use precomputed password/hash mappings.
    
- 🧂 **Salts** are random values added to passwords before hashing.
    
- 🔓 Salts **do not need to be secret**.
    
- 🧂 A unique salt prevents identical passwords from producing identical salted hashes.
    
- 🌈 Salting makes precomputed rainbow tables impractical because mappings would need to account for each salt.
    
- 💥 **Brute force** tests every possible combination and is theoretically guaranteed to find the password given enough time.
    
- 🎭 **Mask attacks** reduce brute-force search space by using known password structure.
    
- 📖 **Dictionary attacks** use statistically likely passwords and are often much more efficient during penetration tests.
    
- 📚 `rockyou.txt` contains millions of real-world leaked passwords and is a commonly used wordlist.
    
- ⚡ Cracking speed depends heavily on the **hashing algorithm and hardware**.
    
- 🐢 Slow password-hashing schemes dramatically reduce the number of guesses an attacker can perform per second.
    

> [!success] 🧠 Final Mental Model
> 
> ```text
> Password Hash
>      │
>      ▼
> Identify Algorithm
>      │
>      ▼
> Understand Salt
>      │
>      ▼
> Reduce Search Space
>      │
>      ├── 📖 Wordlist
>      ├── 🎭 Mask
>      ├── 🌈 Precomputed Data (when applicable)
>      └── 💥 Brute Force
>               │
>               ▼
>          Hash Each Guess
>               │
>               ▼
>        Compare With Target
>               │
>          ┌────┴────┐
>        Match     No Match
>          │           │
>          ▼           └──► Continue
>       🎯 Password
> ```