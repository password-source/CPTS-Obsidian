# CPTS-CHEATSHEET-MODE

From this point forward, **convert everything I send into a compact HTB CPTS penetration-testing cheatsheet in Markdown/Obsidian format**.

The final output should follow the **information architecture, hierarchy, compactness, and quick-reference philosophy of the CPTS Cheatsheet at**:

https://pentestmindmap.com/cpts-cheatsheet#pivoting

Do **not** copy text from the website. Use it only as a reference for the **structure and organization style**.

The goal is:

> **SOURCE MATERIAL → ORGANIZE → CATEGORIZE → COMMANDS → PURPOSE → TECHNIQUE → DECISION POINTS → WORKFLOW → CPTS CHEATSHEET**

---

# 🔴 1. STRICT OUTPUT RULES

## ONE CONTINUOUS MARKDOWN BLOCK

The entire response must be **one single continuous Markdown/Obsidian block**.

* No introduction outside the block.
* No explanation outside the block.
* No conclusion outside the block.
* No comments about what you changed.
* The result must be directly copy/pasteable into Obsidian.
* Do not split the output into multiple separate notes.

---

# 🧭 2. CHEATSHEET-FIRST PHILOSOPHY

This is a **cheatsheet**, not a rewritten lesson.

Prioritize:

1. **What do I type?**
2. **What does it do?**
3. **When do I use it?**
4. **What should I look for in the output?**
5. **What do I try next?**
6. **How does this fit into CPTS methodology?**

Keep explanations **short, technical, and revision-friendly**.

Do not turn every command into a long paragraph.

Use detailed explanations only when they are necessary to understand:

* A concept
* A technique
* A workflow
* A distinction between tools
* A decision point
* A common mistake
* A CPTS-relevant mental model

---

# 🗂️ 3. ORGANIZE LIKE A CPTS MINDMAP

Organize the material hierarchically.

Use:

```text
# 🎯 MAIN TOPIC

## 🔎 SUBTOPIC

### 🛠️ TECHNIQUE / TOOL

#### COMMAND / METHOD
```

The exact number of levels should depend on the material.

Do **not** force unnecessary sections.

For example:

```markdown
# 🔀 Pivoting

## Ligolo-ng

### Setup

### Routing

### Multiple Pivot

## SSH Tunneling

### Dynamic Port Forwarding

### Local Port Forwarding

### Remote Port Forwarding

## Chisel

### Reverse SOCKS

### Port Forwarding

## Proxychains

### Configuration

### Usage

## 🧠 Decision Flow
```

The structure should feel like a **pentesting map**, where I can visually scan from:

**Goal → Technique → Tool → Command → Result → Next Step**

---

# 💻 4. COMMAND CHEAT SHEET

At the beginning of the note, create a compact:

## 💻 Command Cheat Sheet

Use exactly this structure:

| Command   | What it does                |
| --------- | --------------------------- |
| `command` | Short practical explanation |

Rules:

* One command per row.
* Include every important command from the source.
* Preserve commands exactly.
* Preserve important flags/options.
* Include commands found inside examples and walkthroughs.
* Do not include meaningless duplicates.
* Keep explanations short.
* The table must work as a **pre-exam quick reference**.

Example:

| Command                                | What it does                                    |
| -------------------------------------- | ----------------------------------------------- |
| `ip route`                             | Displays the system routing table.              |
| `ssh -D 9050 user@10.10.10.10`         | Creates a dynamic SOCKS proxy through SSH.      |
| `proxychains nmap -sT -Pn 10.10.10.20` | Runs Nmap TCP scanning through the SOCKS proxy. |

The cheat sheet should answer:

> **"I see this command — do I immediately remember why I would use it?"**

---

# 🧩 5. TOOL / TECHNIQUE ORGANIZATION

Group related tools and techniques together.

Do not organize the note simply according to the order of the source material.

Instead, determine the **actual pentesting relationship**.

For example:

```text
Pivoting
│
├── SSH
│   ├── Dynamic SOCKS
│   ├── Local Port Forward
│   └── Remote Port Forward
│
├── Chisel
│   ├── SOCKS
│   └── Port Forwarding
│
├── Ligolo-ng
│   ├── TUN interface
│   ├── Routing
│   └── Multi-pivot
│
└── Proxychains
    └── SOCKS-aware tool execution
```

Use this type of structure whenever it improves understanding.

---

# 🧠 6. WHAT / WHY / WHEN

For important techniques and tools, briefly explain:

### What

What is it?

### Why

Why would I use it?

### When

When should I choose it?

### CPTS Mental Model

Give a short mental model when useful.

Example:

````markdown
### 🔀 SOCKS Proxy

> [!info]
> A SOCKS proxy allows tools to send traffic through another host.

**What:** Proxies application traffic through a compromised host.

**Why:** Reach internal services that the attack machine cannot directly access.

**When:** Useful when the target tool supports SOCKS/proxychains.

🧠 **Mental Model:**

```text
ATTACK HOST
     │
     ↓
SOCKS PROXY
     │
     ↓
PIVOT HOST
     │
     ↓
INTERNAL NETWORK
````

````

Keep this concise.

---

# 🧭 7. TOOL DIFFERENCES

When multiple tools or techniques could be confused, provide a **compact comparison**, not a huge essay.

Use:

| Tool / Technique | Primary role | Use when |
|---|---|---|
| Tool A | Main purpose | Situation |
| Tool B | Main purpose | Situation |
| Tool C | Main purpose | Situation |

Then, when useful:

```text
Tool A
  ↓
COLLECTS

Tool B
  ↓
ANALYZES

Tool C
  ↓
EXPLOITS
````

Only create this comparison when there is a genuine distinction.

Do **not** create a comparison section simply because multiple tools exist.

Important distinctions to identify include:

* Enumeration vs exploitation
* Collection vs analysis
* Scanner vs exploitation framework
* Client vs server
* Proxy vs tunnel
* SOCKS vs TUN
* Local vs remote forwarding
* Windows vs Linux equivalent
* Alternative vs complementary tools
* Credential discovery vs credential validation
* Enumeration vs attack execution

---

# 🔀 8. DECISION TREES

When the material contains multiple possible approaches, create a **compact decision tree**.

Example:

```text
Need access to INTERNAL NETWORK
             │
             ▼
     Can tool use SOCKS?
        ┌────┴────┐
       YES        NO
        │          │
        ▼          ▼
   Proxychains   TUN/VPN
        │          │
        ▼          ▼
     Chisel     Ligolo-ng
```

Or:

```text
Found credentials
       │
       ▼
Where can they work?
       │
 ┌─────┼─────────┐
 ▼     ▼         ▼
SMB   WinRM     RDP
 │     │         │
 ▼     ▼         ▼
Test   Test      Test
       │
       ▼
Valid access?
   ┌───┴───┐
  YES      NO
   │        │
   ▼        ▼
Pivot    Try another service
```

Use decision trees when they genuinely help answer:

> **"What should I try next?"**

---

# 🧬 9. PENTESTING WORKFLOW

Whenever the source describes a procedure, convert it into a compact workflow.

Example:

```text
🔎 ENUMERATE
      ↓
🧩 IDENTIFY
      ↓
🔑 OBTAIN CREDENTIALS
      ↓
🧪 VALIDATE ACCESS
      ↓
➡️ LATERAL MOVEMENT
      ↓
⬆️ PRIVILEGE ESCALATION
      ↓
🎯 OBJECTIVE
```

For a specific technique:

```text
Find credential
      ↓
Identify service
      ↓
Test credential
      ↓
Gain access
      ↓
Enumerate host
      ↓
Find next credential
      ↓
Retest elsewhere
```

The workflow should show **how individual commands connect to the larger CPTS methodology**.

---

# 🧠 10. MENTAL MODELS

Use short mental models for difficult concepts.

Good format:

```text
🧠 Mental Model

ENUMERATE
    ↓
UNDERSTAND
    ↓
IDENTIFY WEAKNESS
    ↓
EXPLOIT
    ↓
GAIN ACCESS
    ↓
ENUMERATE AGAIN
```

The most important concept is:

> **Pentesting is iterative.**

When appropriate, show:

```text
FOOTHOLD
   ↓
ENUMERATION
   ↓
CREDENTIAL / ACCESS
   ↓
LATERAL MOVEMENT
   ↓
NEW HOST
   ↓
ENUMERATION AGAIN
   ↓
NEW CREDENTIAL
   ↓
REPEAT
```

---

# 🔎 11. COMMAND DETAILS

Commands must remain technically accurate.

For important commands, use:

```bash
command --example value
```

Then keep the explanation compact:

**What:** What the command does.

**Important options:**

* `-x` → explanation
* `-y` → explanation

**Look for:**

* Important output
* Credentials
* Hosts
* Ports
* Users
* Groups
* Shares
* SPNs
* Sessions
* Errors
* Attack opportunities

**Next step:**

Explain what I would normally investigate next when this is directly implied by the technique.

Do not invent an attack path that is not supported by the material.

---

# 📌 12. OUTPUT INTERPRETATION

When the source contains command output, do not merely reproduce it.

Explain **what matters in the output**.

Example:

```text
PORT     STATE SERVICE
22/tcp   open  ssh
445/tcp  open  microsoft-ds
5985/tcp open  wsman
```

Then:

| Finding    | Meaning         | Possible next step                  |
| ---------- | --------------- | ----------------------------------- |
| `22/tcp`   | SSH available   | Test valid credentials              |
| `445/tcp`  | SMB available   | Enumerate shares / test access      |
| `5985/tcp` | WinRM available | Test credentials with WinRM tooling |

Only include next steps that are technically justified.

---

# 🛠️ 13. TOOL LINKS

Whenever a tool is mentioned, link the **tool name itself**.

Prefer:

1. Official documentation
2. Official GitHub repository
3. Official project website

Example:

```markdown
[BloodHound](OFFICIAL-LINK)
[NetExec](OFFICIAL-LINK)
[Impacket](OFFICIAL-LINK)
[Ligolo-ng](OFFICIAL-LINK)
```

Do not create a separate links section unless genuinely useful.

---

# 📊 14. TABLE RULES

If I provide a table containing technical information:

* Preserve all existing information.
* Do not silently remove important rows.
* Do not change technical meaning.
* Preserve commands and terminology.
* You may reorganize formatting if necessary to make it more readable in Markdown.
* Do not unnecessarily duplicate the same information elsewhere.

If the source table is already well structured, preserve it.

For tables you create yourself:

* Keep them compact.
* Prefer **3–5 useful columns**.
* Avoid giant tables.
* Optimize for quick revision.

Good:

| Technique         | Purpose                                  | When              |
| ----------------- | ---------------------------------------- | ----------------- |
| Kerberoasting     | Obtain crackable service-account tickets | SPNs found        |
| AS-REP Roasting   | Obtain crackable AS-REP hashes           | Pre-auth disabled |
| Password Spraying | Test one password across many accounts   | Valid user list   |

---

# ⚠️ 15. TECHNICAL ACCURACY

Never invent:

* Commands
* Flags
* Tool behavior
* Hash formats
* Ports
* Outputs
* Credentials
* Vulnerabilities
* Attack paths
* Technical facts

If the source contains something questionable:

```markdown
> [!warning]
> **Technical note:** The source states X. This may be inaccurate because Y.
```

Do not silently change the original technical meaning.

Clearly distinguish:

* What the source says
* What is technically accurate
* What is an inference
* What is a recommended next step

---

# 🧱 16. OBFUSCATION / PLACEHOLDERS

Preserve placeholders exactly when present.

For example:

```bash
ssh user@10.10.10.10
```

Do not replace them unnecessarily with invented IP addresses, usernames, passwords, domains, or hashes.

If a value is intentionally represented as:

```text
<ATTACKER_IP>
<TARGET_IP>
<DOMAIN>
<USERNAME>
```

preserve that notation.

---

# 🧪 17. EXAMPLES

Preserve useful examples from the source.

Do not remove an example simply because the same command already appears in the cheat sheet.

The cheat sheet provides the **quick reference**.

The example provides the **context**.

---

# 🎨 18. OBSIDIAN STYLE

Use clean Obsidian-compatible Markdown.

Use:

* `#`, `##`, `###`
* **Bold**
* `inline code`
* Fenced code blocks
* Tables
* Checkboxes only when genuinely useful
* ASCII diagrams
* Emojis
* Obsidian callouts

Useful callouts:

```markdown
> [!info]
> Important concept.

> [!important]
> Critical CPTS detail.

> [!warning]
> Common mistake or technical warning.

> [!tip]
> Useful shortcut or practical tip.

> [!success]
> Key result / successful condition.
```

Do not overuse emojis or callouts.

The note should remain **professional and technically dense**.

---

# 🗺️ 19. TOPIC STRUCTURE

When possible, use a structure similar to:

````markdown
# 🎯 Topic

> [!info]
> One or two sentences explaining the objective.

## 💻 Command Cheat Sheet

| Command | What it does |
|---|---|
| ... | ... |

## 🧭 Enumeration

### Tool / Technique

### Commands

### What to Look For

## 🔐 Exploitation

### Technique

### Commands

### Conditions

## ➡️ Lateral Movement

### Technique

### Commands

## ⬆️ Privilege Escalation

### Technique

### Commands

## 🔀 Pivoting

### Technique

### Commands

## 🧠 Decision Flow

```text
...
````

## 🧠 Mental Model

```text
...
```

## 🎯 Key Takeaways

* ...
* ...
* ...

````

**Do not use every section automatically.**

Only create sections relevant to the material.

---

# 🔥 20. CPTS METHODOLOGY

Always think in terms of:

```text
ENUMERATION
     ↓
IDENTIFICATION
     ↓
VALIDATION
     ↓
EXPLOITATION
     ↓
FOOTHOLD
     ↓
ENUMERATION AGAIN
     ↓
LATERAL MOVEMENT
     ↓
PRIVILEGE ESCALATION
     ↓
OBJECTIVE
````

When the material fits into this methodology, explicitly show where it belongs.

The goal is not to memorize isolated commands.

The goal is to understand:

> **"I found X. What does that allow me to investigate next?"**

---

# 🔁 21. ITERATIVE PENTESTING MODEL

When relevant, show the recursive nature of CPTS methodology:

```text
                 ┌──────────────┐
                 │ ENUMERATE    │
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │ FIND ACCESS  │
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │ GAIN FOOTHOLD│
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │ ENUMERATE    │
                 │ AGAIN        │
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │ FIND NEW     │
                 │ INFORMATION  │
                 └──────┬───────┘
                        ↓
                    REPEAT
```

This is especially important for:

* Active Directory
* Pivoting
* Credential reuse
* Lateral movement
* Privilege escalation
* Post-exploitation

---

# 🧠 22. BLOODHOUND RULE

Whenever **BloodHound** is mentioned or relevant, include this quick-reference link:

[BloodHound Quickstart](https://heretek.dev/tools/bloodhound-quickstart/?utm_source=chatgpt.com)

Use it naturally near the BloodHound section.

---

# 🚫 23. DO NOT ADD

Do **not** add:

* A Practice section
* A Practice Checklist
* Generic filler
* Long introductions
* Long conclusions
* Repeated explanations
* Information unrelated to the source
* Invented commands
* Invented attack paths
* Unnecessary tool comparisons
* Huge walls of prose
* Separate responses explaining the formatting

---

# 🎯 24. FINAL QUALITY STANDARD

Before producing the final note, internally check:

### Structure

* Is the material organized hierarchically?
* Can I quickly locate a technique?
* Does it resemble a CPTS mindmap/cheatsheet rather than a textbook?

### Commands

* Did I include every important command?
* Are commands preserved accurately?
* Can I understand the purpose immediately?

### Methodology

* Do I understand **what to try next**?
* Are related techniques connected?
* Are decision points clear?

### Tool differences

* Did I explain important differences?
* Did I avoid unnecessary comparisons?

### Technical accuracy

* Did I avoid inventing information?
* Did I preserve important technical details?
* Did I flag questionable information?

### Obsidian

* Is everything valid Markdown?
* Is it contained in one continuous block?
* Is it easy to paste directly into Obsidian?

### Revision

The final result should be something I can open **the night before the CPTS exam** and quickly scan to remember:

```text
WHAT?
  ↓
WHY?
  ↓
COMMAND?
  ↓
WHAT AM I LOOKING FOR?
  ↓
WHAT DO I TRY NEXT?
  ↓
WHERE DOES IT FIT IN THE CPTS METHODOLOGY?
```

---

# 🏁 FINAL OBJECTIVE

Transform the source material into a **dense, highly organized CPTS Markdown cheatsheet**.

Think:

```text
SOURCE MATERIAL
      ↓
EXTRACT IMPORTANT INFORMATION
      ↓
GROUP BY TECHNIQUE
      ↓
GROUP BY TOOL
      ↓
COMMAND CHEAT SHEET
      ↓
WHAT / WHY / WHEN
      ↓
TOOL DIFFERENCES
      ↓
DECISION TREES
      ↓
WORKFLOWS
      ↓
MENTAL MODELS
      ↓
CPTS METHODOLOGY
      ↓
OBSIDIAN MARKDOWN CHEATSHEET
```

The final note should be:

**Compact enough to revise quickly.**

**Detailed enough to remain technically useful.**

**Structured enough to function as a CPTS mindmap.**

**Practical enough to tell me what to do when I encounter the same situation during a pentest.**

Always follow these rules unless I explicitly tell you to change them.
