---
name: issue
description: >-
  Capture a problem or change request, verify it lightly against the codebase,
  draft a structured issue report, then route to one of: upload to
  GitHub/GitLab, document in code, hand off for implementation, or a free-text
  next step. Use when user says "/issue", "report a problem", "file a bug",
  "raise an issue", "track this", or "open a ticket".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

# Issue

Turn a free-text problem report into a structured issue artifact, verify it lightly against the codebase, then route the result to a chosen destination: upload to GitHub/GitLab, document in code, hand off for an implementation, or a free-text custom action.

## Workflow

Follow all 6 steps sequentially.

### Step 1: Parse Input + Preflight

1. Parse everything after `/issue` as `RAW_DESC`.
   - `/issue <text>` -- `RAW_DESC=<text>`
   - `/issue` (bare) -- use `AskUserQuestion` with a free-text field titled "What's the issue?"

2. Run a single Bash call to gather context. Defer `gh`/`glab` auth checks until Step 6A -- the skill is useful for options 2/3/4 even without remote auth.

```bash
IS_GIT=$(git rev-parse --is-inside-work-tree 2>&1)
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || echo "")
REMOTE_URL=$(git remote get-url origin 2>/dev/null || echo "")
CURRENT=$(git symbolic-ref --short HEAD 2>/dev/null || echo "DETACHED")
case "$REMOTE_URL" in
  *github.com*) HOST="github" ;;
  *gitlab.com*|*gitlab.*) HOST="gitlab" ;;
  "") HOST="none" ;;
  *) HOST="unknown" ;;
esac
GH_AVAILABLE=$(command -v gh >/dev/null 2>&1 && echo "yes" || echo "no")
GLAB_AVAILABLE=$(command -v glab >/dev/null 2>&1 && echo "yes" || echo "no")
ISSUE_ID=$(date +%s)
echo "IS_GIT=$IS_GIT"
echo "REPO_ROOT=$REPO_ROOT"
echo "REMOTE_URL=$REMOTE_URL"
echo "HOST=$HOST"
echo "CURRENT=$CURRENT"
echo "GH_AVAILABLE=$GH_AVAILABLE"
echo "GLAB_AVAILABLE=$GLAB_AVAILABLE"
echo "ISSUE_ID=$ISSUE_ID"
```

3. Route based on the results:

| Condition                                                                      | Action                                                                                                                                                  |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Not a git repo (`IS_GIT` is not `true`)                                        | Report "Not a git repository. /issue needs a git repo to anchor the report." and **STOP**                                                                |
| `RAW_DESC` empty after the bare-invocation prompt                              | Report "No description provided." and **STOP**                                                                                                          |
| `RAW_DESC` shorter than 8 chars or only generic words ("bug", "broken", "fix") | `AskUserQuestion` with a free-text field: "That description is too vague. Add detail: what happened, where (file/symbol if known), what you expected." Re-validate. |
| `HOST=none`                                                                    | Continue. Note in Step 5 that option 1 (upload) will be unavailable.                                                                                    |
| `HOST=unknown`                                                                 | Continue. Note in Step 5 that option 1 is unavailable for non-GitHub/GitLab remotes.                                                                    |

Store all preflight vars for downstream steps.

### Step 2: Contextualize and Verify (Light)

Extract references from `RAW_DESC`:

- **Path-like tokens** -- regex match on extensions: `\.(ts|tsx|js|jsx|py|rs|go|md|yaml|yml|json|sh|java|c|h|cpp|hpp|kt|swift|rb|php|sql)(:\d+)?`
- **Symbol-like tokens** -- CamelCase or snake_case identifiers >=4 chars, skipping a stoplist (`bug`, `error`, `issue`, `fix`, `test`, `function`, `method`, `class`)
- **Quoted error strings** -- text in backticks or single/double quotes, especially patterns like `TypeError:` or `Error:`

Apply verification rules:

| Signal                                                                                                                | Action                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0 refs extracted                                                                                                      | Skip verification. Verdict is `not-verifiable`. Continue to Step 3.                                                                                                                                                                                                                                                 |
| 1-2 file paths that exist                                                                                             | `Read` each (cap at 200 lines, or a window around the `:line` if specified). Confirm the area matches the description.                                                                                                                                                                                              |
| 1-2 symbols, no paths                                                                                                 | Single `Grep` per symbol. If a definition is found, `Read` 40 lines around it.                                                                                                                                                                                                                                      |
| Referenced file does not exist                                                                                        | Flag in the draft as "referenced file not found" and continue.                                                                                                                                                                                                                                                      |
| >=3 distinct file paths, OR phrases like "across the codebase", "everywhere we", "find all", "audit"                  | Spawn one `Task` (`subagent_type: general-purpose`) with `RAW_DESC` + extracted refs. Prompt the subagent: "Verify the issue description against the codebase. Read referenced files, grep for symbols, identify whether the described problem is present. Return verdict, up to 5 file:line anchors, contradictions." |

Produce an `EVIDENCE` block:

- **Verdict** -- one of `confirmed | partial | cannot-confirm | not-verifiable`
- **Anchors** -- up to 5 `file:line` bullets

Never block on `cannot-confirm` -- flag it in the draft and let the user decide.

### Step 3: Draft the Report

Infer issue type from `RAW_DESC` heuristics:

| Type          | Trigger phrases                                                |
| ------------- | -------------------------------------------------------------- |
| `bug`         | "broken", "crash", "error", "doesn't work", "fails", "regression" |
| `enhancement` | "should also", "add support for", "would be nice", "missing"   |
| `proposal`    | "we should change", "refactor", "redesign", "consider migrating" |
| `question`    | "why does", "is it intentional", "should this"                 |

Default to `enhancement` if no signals match. Show the inferred type in the draft -- the user can correct it in Step 4.

Draft the report using this structure:

```markdown
## Issue Draft

**Title:** {<=70 chars, capitalized, action-oriented, no conventional prefix}
**Type:** {bug | enhancement | proposal | question}
**Verification:** {confirmed | partial | cannot-confirm | not-verifiable}
**Suggested labels:** {comma-separated, <=4: one type label + optional area + optional priority}

---

### Summary

{1-2 sentences. What is the problem or change in one breath.}

### Context

- `path/to/file.ts:42` -- {what the code does today}
- `path/to/file.ts:88` -- {related call site}

### Reproduction _(bug only -- omit otherwise)_

1. {steps if known; else "Not provided -- needs reporter input"}

### Expected vs Actual _(bug only)_

- **Expected:** {...}
- **Actual:** {...}

### Suggested resolution

{1 paragraph or 2-4 bullets, framed as a proposal not a prescription. For `cannot-confirm`, lead with "Investigate whether..."}

### Open questions

- {ambiguities the reporter or maintainer needs to resolve}
```

**Drafting rules:**

- Derive content from `RAW_DESC` + Step 2 evidence only. **Do not fabricate** reproduction steps, expected behavior, or evidence the user did not supply -- write "Not provided" instead.
- Total body under 60 lines.
- For `enhancement` / `proposal` / `question`, omit the Reproduction and Expected/Actual sections entirely.
- Labels: at most 4. One type label (`bug`/`enhancement`/`proposal`/`question`), one area label inferred from the top-most file path (`area:auth`, `area:api`), and at most one priority (`p1`/`p2`/`p3`) if signals like "crash", "data loss", or "blocker" appear in `RAW_DESC`.

### Step 4: Confirmation Loop

Present the draft inline. Then `AskUserQuestion` with single-select options:

1. **Looks good** -- proceed to Step 5.
2. **Edit** -- user types changes (title, body section, labels). Apply edits exactly as specified, do not re-derive. Re-present the updated draft and re-ask Step 4.
3. **Regenerate from scratch** -- discard the draft and re-run Steps 2-3. After one regeneration, replace option 3 with "Already regenerated -- use this one anyway" to prevent infinite loops.
4. **Cancel** -- discard and **STOP** with "Cancelled. No issue created."

Loop until the user picks option 1 or 4.

### Step 5: Choose Next Step

`AskUserQuestion`, **single-select**, 4 options. If `HOST=none` or `HOST=unknown`, hide option 1 entirely -- do not present it as selectable.

1. **Upload to issues** -- push the report to GitHub or GitLab (the detected host) via `gh` / `glab`.
2. **Document in code** -- add a `TODO(issue):` comment near the relevant code, append to `ISSUES.md`, or write to a custom path.
3. **Implement a fix now** -- hand off the report as context so the main agent (or a Task subagent) starts work.
4. **Something else** -- free-text field for edits, multi-action sequences, custom paths.

### Step 6: Execute the Chosen Action

Before any sub-flow, use the `Write` tool to create `/tmp/issue-${ISSUE_ID}.md` with the final body. **Do not** use `echo`, `cat <<EOF`, or shell redirects.

#### 6A. Upload to issues

1. **Just-in-time auth check** (deferred from Step 1):

   ```bash
   # GitHub
   gh auth status 2>&1; echo "EXIT=$?"
   # GitLab
   glab auth status 2>&1; echo "EXIT=$?"
   ```

   If `EXIT != 0`, report:

   > "GitHub CLI not authenticated. Run `gh auth login`, then re-run /issue. Your draft is saved at /tmp/issue-${ISSUE_ID}.md."

   (Substitute `glab auth login` for GitLab.) **Preserve** the temp file in this case and **STOP**.

2. **CLI missing** (`GH_AVAILABLE=no` or `GLAB_AVAILABLE=no`): report `brew install gh` / `brew install glab`, preserve the temp file, and **STOP**. If the user mentioned `gl`, explicitly note: "GitLab CLI is `glab`, not `gl`."

3. **Duplicate detection (GitHub only, v1)**:

   ```bash
   gh issue list --search "{first 6 significant title words} in:title" --state open --json number,title,url --limit 5
   ```

   If >=1 result, `AskUserQuestion` with options:
   - **Create anyway** -- proceed
   - **View [#N]** -- print URL and **STOP**
   - **Comment on existing [#N]** -- `gh issue comment N --body-file /tmp/issue-${ISSUE_ID}.md`, then clean up temp file
   - **Cancel** -- discard and **STOP**

   GitLab duplicate detection is deferred -- see Guidelines.

4. **Label filtering**: before passing `--label`, run once:

   ```bash
   gh label list --json name --jq '.[].name'
   ```

   Intersect with suggested labels, drop unknowns. Report which were dropped in the success block. (Unknown labels make `gh issue create` fail with an opaque error.) For GitLab, run `glab label list` instead.

5. **Create the issue**:

   For GitHub:
   ```bash
   gh issue create \
     --title "{title}" \
     --body-file /tmp/issue-${ISSUE_ID}.md \
     --label "{filtered}"
   ```

   For GitLab:
   ```bash
   glab issue create \
     --title "{title}" \
     --description-file /tmp/issue-${ISSUE_ID}.md \
     --label "{filtered}"
   ```

6. Capture the URL from stdout. Report:

   ```markdown
   ## Issue Filed

   **URL:** {url}
   **Title:** {title}
   **Host:** {github | gitlab}
   **Labels applied:** {final labels}
   **Labels dropped:** {dropped labels, if any}
   ```

7. Clean up: `rm -f /tmp/issue-${ISSUE_ID}.md`.

#### 6B. Document in code

`AskUserQuestion` with single-select placement options:

1. **Inline at `{file:line}` from Step 2 evidence** -- only show when Step 2 produced a `confirmed` verdict with a single dominant `file:line` anchor.
2. **Append to `ISSUES.md` at repo root** -- create if missing. Recommended default.
3. **Custom path** -- free-text field. Default to `docs/issues/{slug-of-title}.md` if the path doesn't yet exist.

**Inline comment** (language-aware by file extension):

| Extensions                                                | Comment prefix |
| --------------------------------------------------------- | -------------- |
| `.ts`, `.tsx`, `.js`, `.jsx`, `.go`, `.rs`, `.java`, `.c`, `.h`, `.cpp`, `.hpp`, `.kt`, `.swift`, `.php` | `//`           |
| `.py`, `.sh`, `.yml`, `.yaml`, `.rb`                       | `#`            |
| `.sql`, `.hs`                                              | `--`           |
| `.md`, `.html`                                             | `<!--` / `-->` |

Template:

```
{prefix} TODO(issue): {Title}
{prefix} {1-line suggested resolution}
```

For HTML/Markdown, wrap as `<!-- TODO(issue): {Title}\n{1-line suggested resolution} -->`.

Insert via `Edit`: `old_string` = the original line, `new_string` = `{comment block}\n{original line}`. If `Edit` fails on non-uniqueness, expand `old_string` with more surrounding context and retry once. On second failure, fall back to `ISSUES.md`.

**`ISSUES.md` (idempotent)**: `Read` the file first if it exists. Append under `## Open` (create the section if missing). Section template:

```markdown
### {Title}

**Filed:** {YYYY-MM-DD} | **Type:** {type} | **Labels:** {labels}

{Summary section from the draft}

**Evidence:**

- `file:line` -- {detail}

**Suggested resolution:** {one paragraph from draft}

---
```

If the file does not exist, create it with this header:

```markdown
# Issues

Locally tracked issues. Promote to GitHub/GitLab when ready.

## Open

{first entry}
```

**Custom path**: use `Write` to create the file with the full draft as the body.

After success, clean up `/tmp/issue-${ISSUE_ID}.md` and report:

```markdown
## Documented

**Location:** {file:line | ISSUES.md | custom path}
**Next:** Commit when ready.
```

#### 6C. Implement a fix (handoff)

This skill does **not** implement the fix. Default to **inline handoff** so the parent agent continues with full conversation context.

**Escalate to a `Task` subagent only when** both: (a) Step 2 used a Task agent (>=3 files referenced), and (b) the suggested resolution touches >=2 of those files. Otherwise use inline handoff.

**Inline handoff output:**

```markdown
## Now implementing: {Title}

Report saved at /tmp/issue-${ISSUE_ID}.md.

### Plan

1. {first action item from suggested resolution}
2. {second action item}
3. {...}

### Files involved

- `path/to/file1.ts` ({why})
- `path/to/file2.ts` ({why})

Starting on item 1.
```

The next turn (from the parent agent) picks up from "Starting on item 1." Skill stops after emitting the handoff. **Never auto-commit** -- the user runs `/commit` after reviewing.

**Task subagent path** (complex only):

Use `Task` (`subagent_type: general-purpose`) with this prompt structure:

```
Implement the fix described in /tmp/issue-${ISSUE_ID}.md.

## Files involved
{list}

## Plan
{numbered list from the resolution}

## Constraints
- Make the minimal change needed.
- Do not modify unrelated code.
- Do not commit. The user will review and commit themselves.
- Report each file you edit with a one-line summary.

When done, summarize what changed and any tests you would recommend running.
```

Report the subagent's summary back to the user. Do not loop into another round of fixes -- the user takes over.

#### 6D. Something else (free text)

`AskUserQuestion` with a free-text field: "What would you like to do? Examples: 'edit the report', 'upload and also document', 'save to ~/notes/issue.md', 'cancel'."

Interpret the response and route:

| Pattern                                            | Action                                                                                                                            |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| "edit", "revise", "change the {section}"           | Re-enter Step 4 with the user's edits applied                                                                                     |
| "upload AND document", "do both", "1 and 2", "all" | Run 6A then 6B then 6C in the order implied. Between each sub-flow, `AskUserQuestion`: "Continue to {next}?"                       |
| "save to {path}"                                   | Treat as 6B option 3 with the specified path                                                                                      |
| "cancel", "abandon", "never mind"                  | Discard `/tmp/issue-${ISSUE_ID}.md` and **STOP**                                                                                  |
| Other interpretable intent                         | State the interpretation back, ask one clarification via `AskUserQuestion`, then execute                                          |
| Uninterpretable                                    | `AskUserQuestion`: "I couldn't interpret that. Pick: (a) explain differently, (b) go back to the 4 options, (c) cancel."           |

After any success path, clean up `/tmp/issue-${ISSUE_ID}.md`.

## Guidelines

- **GitLab CLI is `glab`, not `gl`.** Every user-facing message must use `glab`. If the user mentioned `gl` anywhere, gently correct them in the error/info copy.
- **`Write` tool for all temp files** -- always create `/tmp/issue-${ISSUE_ID}.md` via the `Write` tool. **DO NOT** use `echo`, `cat <<EOF`, or shell redirects. Pass it via `--body-file` (gh) or `--description-file` (glab).
- **Preserve the temp file on 6A failure** -- on auth or CLI-missing errors, do not delete the file. Tell the user the path so they can re-run after fixing the issue.
- **Defer auth checks** -- never run `gh auth status` / `glab auth status` until the user actually chooses option 1. The skill must work offline for options 2/3/4.
- **Never fabricate evidence** -- if the user did not supply reproduction steps, write "Not provided". If verification cannot confirm the issue, the verdict is `cannot-confirm`, not `confirmed`.
- **Step 4 regenerate cap** -- at most one regeneration. After that, option 3 becomes "use this anyway" to prevent loops.
- **Never auto-commit** -- after any sub-flow, the user runs `/commit` themselves. Do not stage or commit on their behalf.
- **No `--dry-run` flag** -- Step 4 confirmation and Step 5 picker are already the safe abort points. `/tmp/issue-${ISSUE_ID}.md` is the dry-run artifact.
- **Sub-agents are an escalation, not a default** -- Step 2 verification stays inline unless >=3 files are involved. Step 6C handoff stays inline unless Step 2 already used a Task agent AND >=2 files need to change.
- **Self-hosted hosts** (Gitea, Bitbucket, self-hosted GitLab on custom domains) classify as `unknown` and the user routes through option 4 with a free-text intent. No env-var override in v1.
- **GitLab duplicate detection is deferred** -- `glab issue list --search` semantics vary by version. v1 only checks duplicates on GitHub.
- **Label filtering before create** -- always intersect suggested labels with the repo's existing labels (`gh label list` / `glab label list`). Report dropped labels in the success block.
- **`ISSUES.md` is idempotent** -- always `Read` the file first and append under `## Open`. Never overwrite existing entries.
- **Title quoting** -- when passing the title to `gh issue create` / `glab issue create`, ensure it is properly quoted to handle special characters.
