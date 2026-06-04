---
name: tackle
description: >-
  Ingest arbitrary feedback (GitHub/GitLab URL, pasted review, image, file
  path, free text) about the current repo, decompose it into a prioritized
  action plan with per-item owners (human / main-agent / subagent), confirm
  with the user, then dispatch execution. Use when user says "/tackle",
  "address this feedback", "act on this review", "work through this
  feedback", or "what should I do about this".
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

# Tackle

Turn arbitrary external feedback into a dispatched action plan against the current repo. Accepts any source -- GitHub/GitLab URL, pasted reviewer comments, an image, a file path, free text, or any combination. Decomposes feedback into typed actions, assigns each to a human, the main agent, or a subagent, confirms with the user, then dispatches execution.

`/tackle` is execution-first. The deliverable is the diff, not a report.

## Workflow

Follow all 6 steps sequentially.

### Step 1: Parse Input + Preflight

1. Parse everything after `/tackle` as `RAW_ARG`.
   - `/tackle <text-and/or-urls>` -- normal flow.
   - `/tackle` (bare) -- print "Paste the feedback or share a link / file path." and **stop the turn**. Treat the user's next message as `RAW_ARG`.
   - `/tackle --resume` or `/tackle resume [id]` -- jump to **Step 6 (Resume)**.

2. Run a single Bash call to gather context. No upfront auth check.

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
TACKLE_ID=$(date +%s)-$$
echo "IS_GIT=$IS_GIT REPO_ROOT=$REPO_ROOT HOST=$HOST CURRENT=$CURRENT GH=$GH_AVAILABLE GLAB=$GLAB_AVAILABLE TACKLE_ID=$TACKLE_ID"
```

3. Abort gates:

| Condition                                            | Action                                                                                                                                                          |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Not a git repo (`IS_GIT` not `true`)                 | Report "Not a git repository. /tackle needs a git repo to anchor changes." and **STOP**.                                                                        |
| `RAW_ARG` empty after the bare-invocation prompt     | Report "No feedback provided." and **STOP**.                                                                                                                    |
| URL host is not `github.com` or `gitlab.*`           | Continue, but treat that URL as paste-mode -- do not invoke `gh`/`glab`. Note in the corpus: "Unknown host -- URL not fetched, paste only."                     |

4. **Detect input shapes** in `RAW_ARG`. Multiple matches allowed; results concatenate into `FEEDBACK_CORPUS` with explicit source labels.

| Pattern                                                         | Ingest method                                                                                                                            |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `https://github.com/.../issues/N`                               | `gh issue view <url> --json title,body,comments,labels,url,author`                                                                       |
| `https://github.com/.../pull/N`                                 | `gh pr view <url> --json title,body,reviews,comments,url` plus `gh api repos/{owner}/{repo}/pulls/N/comments` for inline review comments |
| `https://gitlab.com/.../-/issues/N` or `.../-/merge_requests/N` | `glab issue view N -F json` / `glab mr view N -F json` (field availability varies by `glab` version)                                     |
| Bare `#N` inside a git repo on a known host                     | `gh issue view N` (or `glab issue view N`) against `origin`. Plain integers (no `#`) are ignored.                                        |
| Path-like token, file exists                                    | `Read` the file (cap 200 lines, or window around `:line`) and treat contents as feedback                                                 |
| Image attached to the chat                                      | Already auto-read -- transcribe what's visible into `FEEDBACK_CORPUS`, but keep the attached image in scope for visual nuance            |
| Image path on disk (`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`)   | `Read` it                                                                                                                                |
| Pasted multi-line block (>= 3 lines)                            | Verbatim into `FEEDBACK_CORPUS`                                                                                                          |
| Plain free text                                                 | Use as `RAW_FEEDBACK`                                                                                                                    |

5. Token-extraction (mirrors `/issue` Step 2):
   - **Paths** -- regex `\.(ts|tsx|js|jsx|py|rs|go|md|yaml|yml|json|sh|java|c|h|cpp|hpp|kt|swift|rb|php|sql)(:\d+)?`
   - **Symbols** -- CamelCase or snake_case identifiers >=4 chars, skipping (`bug`, `error`, `issue`, `fix`, `test`, `function`, `method`, `class`).
   - **Quoted error strings** -- text in backticks or quotes, especially `TypeError:`-style patterns.

6. On any `gh`/`glab` non-zero exit (CLI missing, not authenticated, network, 404), treat that URL as paste-mode for the rest of the run and note the failure in the corpus. Do not retry.

Store `TACKLE_ID`, `HOST`, and `STEP2_USED_TASK_AGENT` (default `false`).

### Step 2: Light Verification

| Signal                                                                                               | Action                                                                                                                                                                                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0 refs extracted                                                                                     | Verdict `not-verifiable`. Continue.                                                                                                                                                                                                                                                                                              |
| 1-2 file paths that exist                                                                            | `Read` each (cap 200 lines, or window around `:line`). Confirm the area matches the feedback.                                                                                                                                                                                                                                    |
| 1-2 symbols, no paths                                                                                | Single `Grep` per symbol. If a definition is found, `Read` 40 lines around it.                                                                                                                                                                                                                                                   |
| Referenced file does not exist                                                                       | Flag as "referenced file not found" and continue.                                                                                                                                                                                                                                                                                |
| >=3 distinct file paths, OR "across the codebase", "everywhere", "find all", "audit"                 | Spawn one `Task` (`subagent_type: general-purpose`) with `FEEDBACK_CORPUS` + extracted refs. Prompt: "Verify the feedback against the codebase. Read referenced files, grep for symbols, identify whether each concern is present. Return verdict, up to 5 file:line anchors, contradictions." Set `STEP2_USED_TASK_AGENT=true`. |

Produce an `EVIDENCE` block: **Verdict** (`confirmed | partial | cannot-confirm | not-verifiable`) and up to 5 `file:line` **Anchors**. Never block on `cannot-confirm`.

### Step 3: Decompose into Action Plan

1. **Split** `FEEDBACK_CORPUS` into atoms by numbered lists, bullets, blank-line paragraphs, or thread reply boundaries. If unstructured, leave as one atom; let the user split in Step 4.

2. For each atom derive an `Action` with these fields:

| Field       | Description                                                                              |
| ----------- | ---------------------------------------------------------------------------------------- |
| `#`         | Sequential id, no gaps                                                                   |
| Description | One-sentence imperative ("Validate JWT expiry in `src/auth.ts:88`")                      |
| Owner       | `human` / `main-agent` / `subagent` (rule below)                                         |
| Anchor      | `file:line` from Step 2, or `--`                                                         |
| Pri         | P1 / P2 / P3                                                                             |
| Source      | URL fragment, "pasted §N", "image #1", or `file:line`                                    |

Only add a `Deps` column if the model spontaneously notices ordering ("A introduces helper, B uses it"). Otherwise omit.

3. **Priority signals**: P1 -- "must", "broken", "wrong", "blocker", "security", "critical", "fix this". P3 -- "nit", "consider", "optional", "minor", "style", "suggestion". Everything else P2.

4. **Owner**: route to `human` if the atom needs deployment, secrets, prod data, external services, or a taste/strategy decision. Route to `subagent` if the atom uses an explicit "investigate / audit / find all / map" verb, OR if it touches >=2 files AND `STEP2_USED_TASK_AGENT=true`. Everything else is `main-agent`.

### Step 4: Confirmation Loop

Present the plan inline as a markdown table:

```markdown
## Action Plan ({total} items)

**Verification:** {verdict} | **Sources:** {comma-separated}

| #   | Description           | Owner              | Anchor           | Pri | Source    |
| --- | --------------------- | ------------------ | ---------------- | --- | --------- |
| 1   | Validate JWT expiry   | main-agent         | `src/auth.ts:88` | P1  | issue #42 |
| 2   | Audit validateJwt use | subagent (research) | --              | P2  | pasted §1 |
| 3   | Migrate to `jose`?    | human              | --               | P2  | image #1  |
```

`AskUserQuestion`, single-select:

1. **Looks good** -- proceed.
2. **Edit** -- after the picker resolves to this option, print "What would you like to change? (re-prioritize, reassign, split, merge, remove, add)" and **stop the turn**. Apply the user's next message exactly, do not re-derive. Re-present.
3. **Regenerate from scratch** -- discard and re-run Step 3. After one regeneration, replace this option with "Already regenerated -- use this one anyway."
4. **Cancel** -- "Cancelled. No plan dispatched." **STOP**.

After option 1, `Write` `/tmp/tackle-${TACKLE_ID}.md`: a header line `**Started:** {YYYY-MM-DD HH:MM} | **Progress:** 0/{total} | **Sources:** {sources} | **Verification:** {verdict}`, then the same table from above with `- [ ] **#{n}**` checkbox prefixes on each row body. No additional templating -- mirror what the user just approved.

### Step 5: Dispatch + Progress

Walk items in dependency order, P1 > P2 > P3 within a level. Dispatch by owner. After each item resolves: `Edit` the plan file to flip `- [ ]` to `- [x]` (or `- [~]` for skips with `-- skipped: {reason}`), bump the `**Progress:**` numerator, and print one line: `Tackle #{n} complete: {description} -- Progress: {x}/{total}`. Continue automatically -- never ask "continue?" between items.

#### 5A. `human` items

Emit as `- [ ]` under a **Needs you** heading in the running summary. Do not execute. Move on.

#### 5B. `main-agent` items

Emit one inline-handoff block and **stop the skill** -- the parent turn implements the item from "Starting on item N.":

```markdown
## Now tackling: #{n} -- {Description}

Plan saved at /tmp/tackle-${TACKLE_ID}.md.

### Plan

1. {first concrete action}
2. {second}
3. {...}

### Files involved

- `path/to/file.ts` ({why})

Starting on item {n}.
```

If multiple `main-agent` items exist, only the first dispatches. The rest queue in the plan file for `/tackle --resume`.

#### 5C. `subagent` items

Dispatch one `Task` (`subagent_type: general-purpose`) per item -- never bundle. **"Archetype" is a prompt-shaping label, not a `subagent_type` value.**

| Verb / intent                                | Archetype    | What the subagent does                                              |
| -------------------------------------------- | ------------ | ------------------------------------------------------------------- |
| "find / list / audit / map / verify / check" | **research** | Read-only. `Grep` + `Read`. Returns anchors + a 5-bullet summary.   |
| "change / refactor / extract / fix / rename" | **execute**  | Minimal `Edit`s, no commit, summary of files touched.               |

Subagent prompt skeleton:

```
You are tackling item #{n} from a feedback-driven plan.

## Feedback excerpt
{relevant slice of FEEDBACK_CORPUS}

## Anchors from prior verification
{file:line bullets from Step 2}

## Your task
{Description}

## Constraints
- Archetype: {research | execute}
- Minimal change. Do not touch unrelated code.
- Do not commit.
- Report each file edited (or read, for research) with a one-line summary.
- If you cannot complete the task, say so -- do not invent a result.
```

Capture the returned summary verbatim into the progress line.

### Step 6: Summary + Resume

#### Summary (end of a normal run)

```markdown
## Tackle Complete

**Plan:** /tmp/tackle-${TACKLE_ID}.md
**Done:** {x}/{total} | **Skipped:** {s} | **Deferred to human:** {h}

### Completed
- [x] #1 {description}

### Needs you
- [ ] #3 {description} -- {source}

### Skipped
- [~] #4 {description} -- {reason}

## What's next
1. Resume queued `main-agent` items via `/tackle --resume ${TACKLE_ID}`
2. Run `/check` to verify, then `/commit`
```

#### Resume (`/tackle --resume` or `/tackle resume [id]`)

1. If no `id`, `Glob /tmp/tackle-*.md`. If one match, auto-pick; else `AskUserQuestion`.
2. `Read` the plan, find the first `- [ ]` in dependency order, and jump to **Step 5** from there. Skip Steps 1-4 -- the plan is already confirmed.

Resume is idempotent: re-toggling `[x]` is a no-op; a fully-done plan exits cleanly.

## Guidelines

- **Always render the picker for enumerated options.** Whenever a step lists 2-4 selectable choices (Step 4 confirmation, resume-id selection), you MUST invoke `AskUserQuestion` so the user sees a pre-filled picker. Never print numbered options in markdown and wait -- the user cannot select from inline text. For genuine free-text inputs, print the question and stop the turn instead of calling `AskUserQuestion` with invented options.
- **Combined plan by design.** Multi-source input merges into one plan with per-item `source:` labels.
- **Subagent dispatch is an escalation, not a default.** Default is `main-agent` inline handoff. Escalate only when the owner rule fires.
- **Don't fabricate evidence.** If reproduction steps are absent, write "Not provided". `cannot-confirm` carries through Step 4.
- **`Write` tool for the plan file.** Never `echo`, `cat <<EOF`, or shell redirects.
- **Preserve `/tmp/tackle-*.md` on partial failure** so resume always works.
- **GitLab CLI is `glab`, not `gl`.** Correct the user in copy if they typed `gl`.
- **One regeneration cap in Step 4** to prevent loops.
- **Inline handoff stops the skill.** After emitting the Step 5B block, return control. Resume picks up the rest.
