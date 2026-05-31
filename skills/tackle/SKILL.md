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

Turn arbitrary external feedback into a dispatched action plan against the current repo. Accepts any source -- GitHub/GitLab URL, pasted reviewer comments, an image, a file path, free text, or any combination. Decomposes the feedback into typed actions, assigns each to a human, the main agent, or a subagent, confirms with the user, then dispatches execution.

`/tackle` is execution-first. The deliverable is the diff, not a report.

## Workflow

Follow all 7 steps sequentially.

### Step 1: Parse Input + Preflight

1. Parse everything after `/tackle` as `RAW_ARG`.
   - `/tackle <text-and/or-urls>` -- normal flow.
   - `/tackle` (bare) -- use `AskUserQuestion` with a free-text field: "Paste the feedback or share a link / file path."
   - `/tackle --resume` or `/tackle resume [id]` -- jump to **Step 7 (Resume)**.

2. Run a single Bash call to gather context. Defer remote auth checks until you actually need them.

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
TACKLE_ID=$(date +%s)
echo "IS_GIT=$IS_GIT"
echo "REPO_ROOT=$REPO_ROOT"
echo "REMOTE_URL=$REMOTE_URL"
echo "HOST=$HOST"
echo "CURRENT=$CURRENT"
echo "GH_AVAILABLE=$GH_AVAILABLE"
echo "GLAB_AVAILABLE=$GLAB_AVAILABLE"
echo "TACKLE_ID=$TACKLE_ID"
```

3. Abort gates:

| Condition                                                           | Action                                                                                                                                         |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Not a git repo (`IS_GIT` not `true`)                                | Report "Not a git repository. /tackle needs a git repo to anchor changes." and **STOP**.                                                       |
| `RAW_ARG` empty after the bare-invocation prompt                    | Report "No feedback provided." and **STOP**.                                                                                                   |
| URL points at a self-hosted host (Gitea, Bitbucket, on-prem GitLab) | Continue, but treat that URL as paste-mode -- do not invoke `gh`/`glab` against it. Note in the corpus: "Self-hosted host -- URL not fetched." |

4. **Detect input shapes** in `RAW_ARG`. Multiple matches are allowed; all results are concatenated into a `FEEDBACK_CORPUS` with explicit source labels.

| Pattern                                                         | Ingest method                                                                                                                            |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `https://github.com/.../issues/N`                               | `gh issue view <url> --json title,body,comments,labels,url,author`                                                                       |
| `https://github.com/.../pull/N`                                 | `gh pr view <url> --json title,body,reviews,comments,url` plus `gh api repos/{owner}/{repo}/pulls/N/comments` for inline review comments |
| `https://gitlab.com/.../-/issues/N` or `.../-/merge_requests/N` | `glab issue view N -F json` / `glab mr view N -F json` (best effort -- field availability varies by `glab` version)                      |
| Bare `#N` or just `N` inside a git repo on a known host         | `gh issue view N` (or `glab issue view N`) against `origin`                                                                              |
| Path-like token, file exists                                    | `Read` the file (cap 200 lines, or a window around `:line` if specified) and treat the contents as feedback                              |
| Image attached to the chat                                      | Already auto-read by Claude -- transcribe what is visible into `FEEDBACK_CORPUS`                                                         |
| Image path on disk (`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`)   | `Read` it                                                                                                                                |
| Pasted multi-line block (>= 3 lines, fenced or bare)            | Verbatim into `FEEDBACK_CORPUS`                                                                                                          |
| Plain free text                                                 | Use as `RAW_FEEDBACK`                                                                                                                    |

5. Token-extraction step (mirrors `/issue` Step 2 verbatim):
   - **Path-like tokens** -- regex on extensions: `\.(ts|tsx|js|jsx|py|rs|go|md|yaml|yml|json|sh|java|c|h|cpp|hpp|kt|swift|rb|php|sql)(:\d+)?`
   - **Symbol-like tokens** -- CamelCase or snake_case identifiers >=4 chars, skipping a stoplist (`bug`, `error`, `issue`, `fix`, `test`, `function`, `method`, `class`).
   - **Quoted error strings** -- text in backticks or single/double quotes, especially patterns like `TypeError:` or `Error:`.

6. If `gh` / `glab` is needed but unavailable or unauthenticated, fall back to treating the URL as paste-mode and note the limitation in the corpus. Do not block on auth -- the user may already have the relevant text in the prompt.

Store `TACKLE_ID`, `HOST`, and a boolean `STEP2_USED_TASK_AGENT` (default `false`) for downstream steps.

### Step 2: Light Verification

Apply the verification routing table from `/issue` Step 2:

| Signal                                                                                               | Action                                                                                                                                                                                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0 refs extracted                                                                                     | Skip verification. Verdict is `not-verifiable`. Continue to Step 3.                                                                                                                                                                                                                                                              |
| 1-2 file paths that exist                                                                            | `Read` each (cap 200 lines, or a window around `:line`). Confirm the area matches the feedback.                                                                                                                                                                                                                                  |
| 1-2 symbols, no paths                                                                                | Single `Grep` per symbol. If a definition is found, `Read` 40 lines around it.                                                                                                                                                                                                                                                   |
| Referenced file does not exist                                                                       | Flag in the draft as "referenced file not found" and continue.                                                                                                                                                                                                                                                                   |
| >=3 distinct file paths, OR phrases like "across the codebase", "everywhere we", "find all", "audit" | Spawn one `Task` (`subagent_type: general-purpose`) with `FEEDBACK_CORPUS` + extracted refs. Prompt: "Verify the feedback against the codebase. Read referenced files, grep for symbols, identify whether each concern is present. Return verdict, up to 5 file:line anchors, contradictions." Set `STEP2_USED_TASK_AGENT=true`. |

Produce an `EVIDENCE` block:

- **Verdict** -- one of `confirmed | partial | cannot-confirm | not-verifiable`
- **Anchors** -- up to 5 `file:line` bullets

Never block on `cannot-confirm`. Flag it and let the user decide.

### Step 3: Decompose into Action Plan

1. Split `FEEDBACK_CORPUS` into atoms using `/doc-n-fix` Step 3 splitting rules: numbered lists (`1.`, `2.`), bullet points (`-`, `*`), blank-line-separated paragraphs, and thread reply boundaries. If the corpus is one unstructured blob, leave it as a single atom and let the user split it during Step 4.

2. For each atom, derive an `Action` with these fields:

| Field       | Description                                                                                   |
| ----------- | --------------------------------------------------------------------------------------------- |
| `#`         | Sequential id, no gaps                                                                        |
| Description | One-sentence imperative (e.g., "Validate JWT expiry in `src/auth.ts:88`")                     |
| Owner       | `human` / `main-agent` / `subagent` (rules below)                                             |
| Anchor      | `file:line` from Step 2 evidence, or empty                                                    |
| Priority    | P1 / P2 / P3 using `/doc-n-fix` signal words                                                  |
| Effort      | S / M / L using `/doc-n-fix` heuristics                                                       |
| Deps        | List of preceding item ids that must complete first                                           |
| Source      | Where this atom came from -- URL fragment, "pasted §N", "image #1", "file `docs/notes.md:42`" |

3. **Priority rules** (from `/doc-n-fix`):
   - **P1 -- Must Fix** -- correctness, bugs, security, blockers. Signals: "must", "broken", "wrong", "blocker", "security", "critical", "fix this".
   - **P2 -- Should Fix** -- substantive improvements, refactors, error handling, test gaps. Default.
   - **P3 -- Nice to Have** -- style, naming, optional suggestions. Signals: "nit", "consider", "might want to", "optional", "minor", "style", "suggestion".

4. **Owner-assignment rules** (apply in order; first match wins):

| Atom looks like...                                                             | Owner                                                    |
| ------------------------------------------------------------------------------ | -------------------------------------------------------- |
| Needs deployment, prod data, secrets, env vars, external services              | `human`                                                  |
| Taste/strategy decision ("should we migrate to X?", "is this approach right?") | `human`                                                  |
| Cross-repo / upstream library / external service work                          | `human`                                                  |
| Explicit "investigate", "audit", "find all", "map", "verify" verb              | `subagent` (explore-context or validate-issue archetype) |
| >=2 files affected AND `STEP2_USED_TASK_AGENT=true`                            | `subagent` (execute-changes archetype)                   |
| 1-2 files, clear intent, <= M effort                                           | `main-agent` (inline handoff)                            |
| Everything else                                                                | `main-agent`                                             |

5. **Dependency inference** -- if atom B references work atom A introduces (e.g., A says "extract `validate()` to a helper" and B says "use the new helper in `users.ts`"), add A to B's `deps`. When in doubt, leave `deps` empty -- the user can edit in Step 4.

### Step 4: Confirmation Loop

Present the plan as a markdown table:

```markdown
## Action Plan ({total} items)

**Verification:** {confirmed | partial | cannot-confirm | not-verifiable}
**Sources:** {comma-separated list, e.g., "issue #42, pasted block, image #1"}

| #   | Description           | Owner              | Anchor           | Pri | Effort | Deps | Source    |
| --- | --------------------- | ------------------ | ---------------- | --- | ------ | ---- | --------- |
| 1   | Validate JWT expiry   | main-agent         | `src/auth.ts:88` | P1  | S      | --   | issue #42 |
| 2   | Audit validateJwt use | subagent (explore) | --               | P2  | M      | 1    | pasted §1 |
| 3   | Migrate to `jose`?    | human              | --               | P2  | L      | --   | image #1  |
```

`AskUserQuestion`, single-select:

1. **Looks good** -- proceed to Step 5.
2. **Edit** -- user types changes (re-prioritize, reassign owner, split, merge, remove, add). Apply edits exactly as specified, do not re-derive. Re-present and re-ask Step 4.
3. **Regenerate from scratch** -- discard the plan and re-run Step 3. After one regeneration, replace option 3 with "Already regenerated -- use this one anyway."
4. **Cancel** -- "Cancelled. No plan dispatched." **STOP**.

After the user picks option 1, use the `Write` tool to persist `/tmp/tackle-${TACKLE_ID}.md` with this body:

```markdown
# Tackle ${TACKLE_ID}

**Started:** {YYYY-MM-DD HH:MM} | **Progress:** 0/{total}
**Sources:** {sources}
**Verification:** {verdict}

## Plan

- [ ] **#1** {description} -- owner: {owner}, anchor: {anchor or "--"}, pri: {Pri}, effort: {Effort}, deps: {deps or "--"}, source: {source}
- [ ] **#2** ...
```

### Step 5: Dispatch

Walk items in dependency-topological order, priority-first within a level (P1 > P2 > P3). For each item, dispatch by owner.

#### 5A. `human` items

Emit as a `- [ ]` line under a **Needs you** heading in the running summary. Do NOT execute. Move on.

#### 5B. `main-agent` items

Emit one inline-handoff block (template lifted from `/issue` 6C). The parent turn picks up from "Starting on item N.":

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

If multiple `main-agent` items exist, dispatch only the first. The rest stay queued in `/tmp/tackle-${TACKLE_ID}.md` as `- [ ]` and are picked up via `/tackle --resume` after the parent turn completes the current item.

#### 5C. `subagent` items

Dispatch one `Task` (`subagent_type: general-purpose`) per item. Pick the archetype by the leading verb of the atom:

| Verb / intent                                | Archetype           | What the subagent does                                                       |
| -------------------------------------------- | ------------------- | ---------------------------------------------------------------------------- |
| "find / list / audit / map"                  | **explore-context** | Read-only. `Grep` + `Read` to surface anchors + a 5-bullet summary.          |
| "verify / check / confirm"                   | **validate-issue**  | Targeted reads/greps; may run a test if relevant. Returns verdict + anchors. |
| "change / refactor / extract / fix / rename" | **execute-changes** | Minimal `Edit`s, no commit, summary of files touched.                        |

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
- Archetype: {explore-context | validate-issue | execute-changes}
- Make the minimal change needed. Do not touch unrelated code.
- Do not commit. The user reviews and commits themselves.
- Report each file you edited (or read, for explore/validate) with a one-line summary.
- If you cannot complete the task, say so and explain why -- do not invent a result.
```

Capture the subagent's summary verbatim into the progress update in Step 6.

### Step 6: Progress Updates

After each completed dispatch (subagent returns; or `main-agent` items resolved on resume):

1. **Toggle the item** -- `Edit` `/tmp/tackle-${TACKLE_ID}.md`, change `- [ ] **#{n}**` to `- [x] **#{n}**`.
2. **Update the counter** -- bump the `**Progress:**` numerator.
3. **Print one line**:

```
Tackle #{n} complete: {description} -- Progress: {x}/{total}
```

If a subagent reports failure or a `main-agent` item is skipped during a resume, record the item as skipped with the reason inline in the plan file:

```markdown
- [~] **#{n}** {description} -- skipped: {reason}
```

Proceed to the next item automatically. Do NOT ask "continue?" between items.

### Step 7: Summary + Resume

#### Summary (end of a normal run)

```markdown
## Tackle Complete

**Plan:** /tmp/tackle-${TACKLE_ID}.md
**Done:** {completed}/{total} | **Skipped:** {skipped} | **Deferred to human:** {human-count}

### Completed

- [x] #1 {description}
- [x] #2 {description}

### Needs you

- [ ] #3 {description} -- {source}

### Skipped

- [~] #4 {description} -- {reason}

## What's next

1. Pick up the queued `main-agent` items via `/tackle --resume ${TACKLE_ID}`
2. Run `/check` to verify the changes
3. Run `/commit` when ready
```

#### Resume (`/tackle --resume` or `/tackle resume [id]`)

1. If no `id` was given, `Glob /tmp/tackle-*.md` and `AskUserQuestion` to pick one. If only one match exists, pick it automatically.
2. `Read` the plan file. Recompute the next pending item (first `- [ ]` in dependency order).
3. If no pending items, emit a "Plan already complete" summary and **STOP**.
4. Otherwise, jump straight to **Step 5** from the first pending item. Skip Steps 1-4 -- the plan is already confirmed.

Resume is idempotent: toggling an already-`[x]` item is a no-op; re-running on a fully-done plan exits cleanly.

## Guidelines

- **Combined plan by design.** Multi-source input merges into one plan with per-item `source:` labels. Never produce two separate plans for one invocation.
- **Subagent dispatch is an escalation, not a default.** Default is `main-agent` inline handoff. Match `/issue` 6C: only escalate to `subagent` when >=2 files are affected AND Step 2 already used a Task agent, OR the atom uses an explicit explore/validate verb.
- **No auto-commit.** Hand off to the user; they run `/commit`.
- **No env-var or secret asks.** Any atom that needs them auto-routes to `human`.
- **Don't fabricate evidence.** If reproduction steps are absent, the action description says "Not provided" -- do not invent them. If Step 2 says `cannot-confirm`, the plan carries `cannot-confirm` through Step 4.
- **`Write` tool for all temp files.** Always create `/tmp/tackle-${TACKLE_ID}.md` via `Write`. **DO NOT** use `echo`, `cat <<EOF`, or shell redirects.
- **Preserve `/tmp/tackle-*.md` on partial failure** so `/tackle --resume` always works. Only the final summary in Step 7 may suggest the user clean it up.
- **GitLab CLI is `glab`, not `gl`.** If the user typed `gl`, correct them in the user-facing copy.
- **Self-hosted hosts route through paste mode.** The skill never invokes `gh`/`glab` against `unknown` hosts.
- **One regeneration cap in Step 4** to prevent infinite loops.
- **Resume is single-session-safe.** Toggling an already-toggled item is a no-op; re-reading `/tmp/tackle-*.md` is idempotent.
- **Step 5C subagent prompts are scoped per item.** Never bundle two atoms into one subagent call -- that defeats per-item progress tracking.
- **Inline handoff stops the skill.** After emitting the Step 5B block for a `main-agent` item, return control to the parent agent. The parent turn implements that item; resume picks up the rest.
