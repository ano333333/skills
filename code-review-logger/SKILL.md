---
name: code-review-logger
description: Automatically capture and log user corrections to agent-written code. Use this skill whenever the user provides feedback, corrections, or improvement requests for code that you (the agent) just wrote or modified. This includes cases where the user says things like "change this", "fix that", "this is wrong", "update this part", or provides any corrective guidance after you've written code. The skill logs the before/after code snippets and the user's feedback to a persistent markdown file for learning and reference. Always trigger this when the user is correcting your code work, even if they don't explicitly ask to "log" or "record" it.
---

# Code Review Logger

## Purpose

This skill helps Claude learn from user corrections by systematically logging feedback on agent-written code. Every time a user corrects, modifies, or provides feedback on code you've written, this skill captures:

- The original code (what you wrote)
- The user's correction or feedback
- The corrected code (what it should have been)

This creates a persistent learning log that helps identify patterns in mistakes and improvements over time.

## When to Use This Skill

Trigger this skill automatically whenever:

1. **The user corrects your code** - They point out something wrong or suboptimal in code you just wrote
2. **The user asks for modifications** - They request changes to code you recently created or edited
3. **The user provides critical feedback** - They explain why something you wrote isn't quite right
4. **The user rewrites your code** - They show you a better way to do something

Examples of triggering phrases:
- "Actually, change that to..."
- "No, that's not right. It should be..."
- "Fix the bug in that function"
- "Update this to use X instead of Y"
- "This part is wrong because..."

Do NOT trigger this skill when:
- The user is asking you to review THEIR code (not yours)
- You're making proactive improvements without user correction
- The user is asking general questions about code

## How It Works

### Step 1: Identify the Correction Context

When the user provides a correction, determine:

1. **What code did you write?** - Look back in the conversation to find the code you recently wrote or modified that the user is now correcting
2. **Which file(s) are affected?** - Identify the file path(s) being corrected
3. **What's the user's feedback?** - Extract their explanation, reasoning, or correction request

### Step 2: Capture the Before and After

Extract:
- **Before code**: The relevant code snippet you originally wrote
- **User feedback**: Their exact words explaining what's wrong or what to change
- **After code**: The corrected version (either what they specified or what you're about to write based on their feedback)

If you haven't written the correction yet, complete the correction first, then log it.

### Step 3: Write to the Log File

Append the correction to `.code-reviews.md` in the repository root using this exact format:

```markdown
## YYYY-MM-DD HH:MM - path/to/file.ext

### 修正前
\`\`\`language
[original code snippet]
\`\`\`

### 修正指示
[user's feedback/correction in their exact words]

### 修正後
\`\`\`language
[corrected code snippet]
\`\`\`

---

```

**Important formatting notes:**
- Use the current date and time in `YYYY-MM-DD HH:MM` format
- Include the full file path relative to the repository root
- Use the appropriate language identifier in code fences (e.g., `python`, `javascript`, `typescript`, `rust`)
- Keep code snippets focused on the relevant changed sections - don't include the entire file unless necessary
- Preserve the user's feedback verbatim - don't paraphrase or clean it up
- If the file doesn't exist, create it with a header first

### Step 4: Confirm the Logging

After writing to the log, briefly confirm to the user what was logged. Keep it concise:

"修正内容を`.code-reviews.md`に記録したのだ"

## File Location

The log file is always at the repository root: `.code-reviews.md`

If you're not in a git repository, politely inform the user that this skill requires a git repository to function, and ask where they'd like the log saved instead.

## Handling Edge Cases

**Multiple files changed**: If the correction affects multiple files, create separate entries for each file with the same timestamp.

**Unclear what changed**: If you're not sure what code the user is referring to, ask for clarification before logging: "どのコードの部分を指しているのだ?"

**Large code blocks**: If the changed code is very large (>50 lines), include context around the specific changes with `// ...` or `# ...` to indicate omitted code.

**No code written yet**: If the user is giving you requirements upfront (not correcting existing code), don't trigger this skill. Only trigger when correcting code you already wrote.

## Initial File Format

If `.code-reviews.md` doesn't exist, create it with this header:

```markdown
# Code Review Log

このファイルは、エージェントが書いたコードに対するユーザーの修正履歴を記録します。

---

```

Then append the first entry.

## Example Entry

```markdown
## 2026-03-06 14:32 - src/api/handler.ts

### 修正前
\`\`\`typescript
async function fetchData(url: string) {
  const response = await fetch(url);
  return response.json();
}
\`\`\`

### 修正指示
エラーハンドリングが無いのだ。fetchが失敗した時の処理を追加してほしいのだ。

### 修正後
\`\`\`typescript
async function fetchData(url: string) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json();
  } catch (error) {
    console.error('Failed to fetch data:', error);
    throw error;
  }
}
\`\`\`

---

```

## Why This Matters

This persistent log serves multiple purposes:

1. **Pattern recognition**: Over time, you can identify common types of corrections
2. **Learning resource**: Future agents can learn from past corrections
3. **Quality improvement**: Teams can analyze where AI code generation commonly needs human intervention
4. **Documentation**: Creates a record of how code evolved with human guidance

By systematically capturing corrections, we build a knowledge base that makes AI assistance progressively more valuable.
