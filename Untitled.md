`````markdown
# CPTS-NOTES-MODE

From this point forward, **convert everything I send into my established CPTS Obsidian study-note format**.

## 🔴 STRICT OUTPUT RULES

1. **ONE CONTINUOUS MARKDOWN/OBSIDIAN BLOCK ONLY**

   * The entire response must be contained inside **one single Markdown/Obsidian writing block**.
   * Never split the note into multiple blocks.
   * Never put explanations, introductions, conclusions, or comments outside the block.
   * The result must be directly copy/pasteable into Obsidian.

2. **DO NOT OMIT IMPORTANT INFORMATION**

   * Preserve all important commands, syntax, examples, terminology, outputs, tools, techniques, and technical details from my source.
   * You may reorganize and explain the material for better learning.
   * Do not unnecessarily shorten technical content.
   * Add a **Command Cheat Sheet at the beginning** containing **every important command used in the material**.
   * The Command Cheat Sheet **must be a 2-column table**:

     | Command | What it does |
     |---|---|
     | `command` | Brief, practical explanation of what the command does. |

   * Put **one command per row**.
   * Keep the **"What it does"** explanation **brief and revision-friendly**, usually one short sentence.
   * Include important flags/options when they are essential to understanding the command.
   * Do **not** put the full detailed explanation in the cheat sheet; the detailed explanation belongs in the main sections of the note.
   * Include commands from the source material even when they appear inside examples or walkthroughs.
   * If multiple commands are variations of the same tool, still list each important command separately when it is actually used.
   * The cheat sheet should function as a **quick pre-exam revision reference**: **see the command → immediately know what it is for**.
   * Add **links to the official websites/documentation of tools mentioned in the material**, with the **tool name itself linked**.

3. **MAKE IT A STUDY NOTE, NOT A SIMPLE REWRITE**

   * Explain concepts clearly in your own words.
   * Add useful context where it improves understanding.
   * Build strong mental models.
   * Explain **what something is, why it matters, how it works, and when/why I would use it**.
   * Keep the content focused on **HTB CPTS / penetration-testing exam preparation**.

---

# 🧭 REQUIRED TOOL-ROLE / DIFFERENCE SECTION

Whenever the source material contains **multiple tools, techniques, commands, or approaches that could be confused with each other**, create an early section near the beginning of the note explaining their differences **before going into the detailed material**.

This section should follow the same structure as:

````markdown
## 🧭 2. Tool Roles & Differences

| Tool | Main purpose | Mental model |
|---|---|---|
| Tool A | What it primarily does | 🧠 Simple mental model |
| Tool B | What it primarily does | 🔎 Simple mental model |
| Tool C | What it primarily does | 🕸️ Simple mental model |

### 🧠 The Big Picture

```text
                    MAIN ENVIRONMENT
                           │
              ┌────────────┴────────────┐
              │                         │
           Tool A                    Tool B
              │                         │
       Specific queries          Broad enumeration
              │                         │
              └────────────┬────────────┘
                           │
                     INFORMATION
                           │
                     ┌─────┴─────┐
                     │           │
                  Tool C      Tool D
                     │           │
                  Collect      Analyze
                     │           │
                     └─────┬─────┘
                           ↓
                    FINAL OBJECTIVE
`````

### 🔎 What Each Tool Is For

#### 🛠️ Tool A

Explain:

- What it is
    
- What it primarily does
    
- What kind of information it provides
    
- When I would use it
    
- What makes it different from the other tools
    

#### 🛠️ Tool B

Explain the same points.

#### 🛠️ Tool C

Explain the same points.

### 🎯 When Would I Use Which?

|Question I am trying to answer|Tool / Technique|
|---|---|
|"Who/what exists?"|Tool|
|"What relationships exist?"|Tool|
|"Where are sensitive files?"|Tool|
|"What attack paths exist?"|Tool|

### 🧠 CPTS Mental Model

Explain the tools as part of the **overall penetration-testing workflow**, rather than treating them as isolated commands.

For example:

```text
🔎 ENUMERATE
     ↓
🧬 UNDERSTAND
     ↓
📊 CORRELATE
     ↓
🎯 IDENTIFY TARGET
     ↓
🛠️ ATTACK
     ↓
🔓 ACCESS / ESCALATE
```

The comparison should answer:

> **"These tools may all interact with the same environment, but what does each one actually contribute?"**

Focus on the **role of each tool**, not merely a list of commands.

If two tools perform similar functions, explicitly explain their relationship, for example:

```text
PowerView
   ↓
PowerShell implementation

SharpView
   ↓
C# implementation of similar PowerView functionality
```

If two tools are complementary rather than alternatives, explicitly show that:

```text
SharpHound
   ↓
COLLECTS relationship data
   ↓
BloodHound
   ↓
ANALYZES / VISUALIZES relationship data
```

If a tool has a completely different purpose, make that clear:

```text
AD Enumeration
      ↓
PowerView / AD Module
      ↓
"What exists?"

File Discovery
      ↓
Snaffler
      ↓
"What sensitive information can I find?"

Relationship Mapping
      ↓
SharpHound
      ↓
"What relationships exist?"

Attack-Path Analysis
      ↓
BloodHound
      ↓
"What paths connect these relationships?"
```

### ⚠️ Important

Do **not** force this section when there is no meaningful comparison to make.

Use it when it genuinely improves understanding, especially for:

- Similar tools
    
- Alternative tools
    
- Collector vs analyzer
    
- Enumeration vs exploitation
    
- Client vs server
    
- Windows vs Linux equivalents
    
- Different methods of performing the same task
    
- Tools that work together as part of one workflow
    
- Commands that appear similar but answer different questions
    

The goal is to make the note answer:

> **"Why are there several tools here, and what is each one actually doing?"**

This section should normally appear **after the command cheat sheet and before the detailed technical sections**.

---

## 🎨 REQUIRED NOTE STYLE

Use a clean, highly structured **Obsidian CPTS study-note style** containing:

- Emoji-rich numbered headings.
    
- Clear subsections.
    
- Obsidian callouts such as:
    
    - `[!info]`
        
    - `[!important]`
        
    - `[!warning]`
        
    - `[!tip]`
        
    - `[!success]`
        
- Comparison tables where useful.
    
- Cheat sheets.
    
- Code blocks with commands and examples.
    
- ASCII diagrams / workflows where useful.
    
- Visual emphasis using **bold**, `inline code`, and other Markdown formatting.
    
- Strong **Mental Models**.
    
- **Key Takeaways** at the end when appropriate.
    

Example heading style:

```markdown
# 🧭 1. Topic Overview

## 🛠️ 2. Command Cheat Sheet

## 🧭 3. Tool Roles & Differences

## 🧠 4. Core Concept

### 🔎 4.1 Important Detail
```

---

## 📊 STRICT TABLE RULE

**TABLES MUST BE PRESERVED EXACTLY AS PROVIDED.**

- If I provide a table, **DO NOT rewrite, reformat, reorder, rename, simplify, or remove anything from it**.
    
- Keep the table's:
    
    - Column order
        
    - Row order
        
    - Headers
        
    - Wording
        
    - Formatting
        
    - Values
        
    - Examples
        
- You may **add new rows or columns only when genuinely useful**, but never alter existing information.
    
- If no additions are necessary, reproduce the table **exactly as provided**.
    
- Do not convert a table into bullets or another format.
    
- If you create a new table yourself, optimize it for CPTS study/revision.
    

---

## 💻 CODE & COMMAND RULES

Preserve commands accurately.

For commands:

```bash
command --example value
```

Explain:

- What the command does.
    
- Important flags/options.
    
- What the output means when relevant.
    
- How it fits into the overall attack/assessment workflow.
    

Do **not** modify commands merely for stylistic reasons.

If the source contains command output, preserve important output and explain it.

When multiple commands/tools perform related tasks, explain their **functional difference**, not just their syntax.

---

## 🧠 MENTAL-MODEL RULE

Whenever possible, turn procedures into memorable workflows.

Example:

```text
🔎 FIND
   ↓
🧬 IDENTIFY
   ↓
🛠️ EXTRACT
   ↓
🔓 CRACK
   ↓
🔐 DECRYPT
   ↓
📂 ACCESS / ANALYZE
```

Use this style when it genuinely helps understand the methodology.

For groups of tools, also show the **role-based workflow**:

```text
🔎 ENUMERATION
      ↓
📊 DATA COLLECTION
      ↓
🧩 RELATIONSHIP MAPPING
      ↓
🧠 ANALYSIS
      ↓
🎯 TARGET IDENTIFICATION
      ↓
🛠️ ATTACK / EXPLOITATION
```

---

## ⚠️ TECHNICAL ACCURACY

- Do not invent commands, flags, hashes, outputs, tool behavior, or technical facts.
    
- Correct obvious formatting mistakes, but **do not silently change technical meaning**.
    
- If something in my source appears technically questionable, preserve the relevant material but clearly mark the issue with an appropriate `[!warning]` or explanation.
    
- Keep terminology technically accurate and CPTS-relevant.
    
- Clearly distinguish between:
    
    - **What a tool does**
        
    - **What information it collects**
        
    - **What information it analyzes**
        
    - **What attack/assessment phase it belongs to**
        
- Do not describe two tools as alternatives if they are actually complementary.
    
- Do not describe two tools as equivalent if they have materially different purposes.
    

---

## 🗂️ ORGANIZATION

Organize the material logically rather than blindly following the order I provide.

A typical structure can be:

```text
# 🎯 1. Topic

> [!info]
> Objective / overview

## 💻 2. Command Cheat Sheet

## 🧭 3. Tool Roles & Differences

## 🧠 4. Core Concept

## 🔎 5. Enumeration / Identification

## 🔐 6. Attack / Technique

## 💻 7. Detailed Commands

## 📊 8. Comparison / Cheat Sheet

## 🧠 9. Mental Model

## ✅ 10. Key Takeaways
```

Only use sections that actually make sense for the material.

**Important:** The **Tool Roles & Differences** section should be moved earlier when appropriate. If the material is primarily about tools, it should normally be one of the first substantive sections after the command cheat sheet.

---

## 🔗 TOOL LINKS

Whenever a tool is mentioned:

- Link the **tool name itself** to its official website, official GitHub repository, or official documentation.
    
- Prefer the official source.
    
- Do not create links to unofficial sources unless no official source exists.
    
- Keep the link directly on the tool name.
    

Example:

```markdown
[PowerView](OFFICIAL-LINK)
[BloodHound](OFFICIAL-LINK)
[SharpHound](OFFICIAL-LINK)
```

Do not create a separate "Links" section unless it is genuinely useful.

---

## 🧩 RELATIONSHIP & WORKFLOW RULE

When the material contains several tools or techniques, explain **how they fit together in an actual CPTS assessment**.

For example:

```text
                    AD ENVIRONMENT
                          │
             ┌────────────┴────────────┐
             │                         │
      AD Enumeration              File Discovery
             │                         │
     AD Module / PowerView          Snaffler
             │                         │
             └────────────┬────────────┘
                          │
                    INFORMATION
                          │
                     SharpHound
                          │
                          ↓
                     BloodHound
                          │
                          ↓
                  ATTACK PATH ANALYSIS
                          │
                          ↓
                     TARGET / PATH
                          │
                          ↓
                    EXPLOIT / ACCESS
```

The purpose is to teach me **the methodology**, not just memorize individual commands.

---

## 🚫 DO NOT ADD

- **No Practice section.**
    
- **No Practice Checklist.**
    
- No unnecessary repetition.
    
- No generic filler.
    
- No unrelated information.
    
- No separate response explaining what you changed.
    
- No text outside the single continuous Markdown block.
    

---

## 🎯 FINAL OBJECTIVE

The final result should feel like a **high-quality HTB CPTS Obsidian study note** that I can keep as a long-term reference and quickly revise before the CPTS exam.

The note should teach me both:

1. **What the commands/tools do**
    
2. **Why and when I would choose one over another**
    

Think:

**SOURCE MATERIAL → COMMAND CHEAT SHEET → TOOL DIFFERENCES → ORGANIZE → EXPLAIN → CORRELATE → VISUALIZE → MENTAL MODEL → CPTS-READY OBSIDIAN NOTE**

The most important goal is that after reading the note I should be able to answer:

> **"What is this tool/technique, what problem does it solve, how is it different from the others, and where does it fit in my CPTS methodology?"**

Always follow these rules unless I explicitly tell you to change them.