---
name: check
description:
  Verifies the agent's current work against a specific question by analyzing
  unstaged changes, staged changes, recent commits, and codebase context. Answers
  succinctly for a senior audience. Use when user says "/check", "verify that",
  "confirm that", "check if", "is X done?", or asks about current session changes.
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

# Check

Read-only verification of the agent's current work against a specific question. Produces a verdict-first answer backed by evidence from git state and codebase context.

`Write` and `Edit` are intentionally excluded from allowed-tools. This skill MUST NOT modify any files.

## Workflow

Follow all 4 steps sequentially.

### Step 1: Parse the Question

1. Extract the question text from everything after `/check`.
   - `/check does the new endpoint handle errors?` -> question is "does the new endpoint handle errors?"
   - `/check` (bare, no argument) -> must clarify

2. Classify the question:
   - **Verification** — expects YES / NO / PARTIAL (e.g., "is X done?", "does Y handle Z?", "are tests passing?")
   - **Explanation** — expects a short description (e.g., "what changed in auth?", "how does the new caching work?")
   - **General** — anything else; answer directly

3. **Clarification gate** — only ask using `AskUserQuestion` if:
   - Bare `/check` with no argument at all
   - Question references something outside the codebase that you cannot inspect (deployment environment, external system state, production data)

   Do NOT ask for clarification when the question is broad but interpretable. Instead, state your interpretation in the output and proceed.

### Step 2: Gather Evidence

Run these evidence-gathering steps. Use parallel tool calls where possible.

#### Always run (git state):

```bash
git status --short
```

```bash
git diff
```

```bash
git diff --staged
```

```bash
git log --oneline -10
```

**Edge case — no git repo:** If git commands fail, skip them. Use file reads and Grep only. This is not an error; state "Not a git repository — checked files directly." in the output.

**Edge case — no changes:** If all git commands return empty, that is a valid finding. The answer may be "nothing has changed."

#### Conditionally run (based on the question):

| Question type                                | Action                                                        |
| -------------------------------------------- | ------------------------------------------------------------- |
| Names specific files or paths                | Read those files                                              |
| Asks "how does X work" or "what does X do"   | Grep for X, read relevant files                               |
| Asks "do tests pass" or "are tests green"    | Run the test suite via Bash (read-only — do NOT fix failures) |
| Asks about a function, class, or symbol      | Grep for its definition and usages                            |
| Asks about recent changes to a specific area | `git log --oneline -20 -- {path}` and `git diff -- {path}`    |

### Step 3: Synthesize (conditional delegation)

Evaluate the complexity of the evidence gathered:

- **Simple** (4 or fewer files to consider, single code path): Answer directly from the evidence. Skip to Step 4.
- **Complex** (5+ files, multi-hop code paths, cross-module question): Spawn a single `Task` agent (`subagent_type: general-purpose`) with the following prompt structure:

```
You are a senior engineer reviewing recent work in a codebase.

## Question
{the user's question}

## Evidence
{paste the git diff output, file contents, and any other gathered evidence}

## Task
Analyze the evidence and answer the question. Be specific:
- Reference exact file paths and line numbers
- For verification questions, state YES / NO / PARTIAL with justification
- For explanation questions, describe what changed and why it matters
- Note any gaps, risks, or incomplete work you observe

Keep your answer concise — bullets over paragraphs.
```

### Step 4: Deliver the Answer

Output in this exact format:

```markdown
## Check: {question, truncated to ~60 chars if needed}

**Verdict:** YES | NO | PARTIAL | {direct answer for non-verification questions}

{One sentence expanding the verdict — what makes it yes/no/partial, or the key takeaway.}

**Evidence:**

- `{file:line}` — {specific supporting detail}
- `{file:line}` — {specific supporting detail}
- {up to 4 bullets total}

**Could elaborate on:** {1-2 follow-up angles the user might care about} _(optional — omit if the answer is complete)_
**Suggestions:** {1-2 actionable items if gaps were found} _(optional — omit if verdict is a clean YES)_
```

### Format rules

- **Verdict line is the most important output.** A senior manager skimming should get the answer from that line alone.
- No preamble — do not start with "Let me check..." or "I'll look into...". Go straight to the output.
- Use specific file paths and line numbers over vague descriptions like "the auth module."
- Be direct about gaps. "PARTIAL — error handling added for network errors but not for timeouts" is better than "mostly done."
- If the question was interpreted (broad but not ambiguous), state the interpretation: `**Interpreted as:** "Did the recent changes add input validation to the /users endpoint?"`
- Omit the optional sections rather than padding them with filler.

## Guidelines

- This skill is strictly read-only. Never suggest running a write operation and never attempt to fix issues found during checking.
- Prefer evidence from `git diff` (what actually changed) over file reads (current state) when both are available.
- For "do tests pass" questions, run the test command and report the result. Do not attempt to fix failing tests.
- If the evidence is ambiguous, say so. "PARTIAL — cannot confirm without running the integration suite" is a valid verdict.
- Keep the total output under 30 lines. This is a spot-check, not a full audit.
