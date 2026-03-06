---
name: wtf
description:
  Quick situational awareness for the current git branch. Summarizes what a
  feature branch is about by analyzing commits and changes against trunk. On
  trunk, highlights recent interesting activity. Use when user says "wtf",
  "what's going on", "what is this branch", "what changed", or "catch me up".
allowed-tools:
  - Bash
  - AskUserQuestion
---

# WTF

Single-pass situational awareness for the current git branch. Analyzes git state only — no source file reads.

`Write`, `Edit`, `Read`, `Glob`, and `Grep` are intentionally excluded from allowed-tools. This skill MUST NOT read or modify any files.

## Workflow

Follow all 3 steps sequentially.

### Step 1: Detect Context

1. Parse the optional focus argument from everything after `/wtf`.
   - `/wtf auth` -> focus is "auth"
   - `/wtf` (bare) -> no focus, full analysis

2. Run a single Bash call to detect the git context:

```bash
CURRENT=$(git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD)
DETACHED=$?
if git rev-parse --verify main &>/dev/null; then TRUNK="main"
elif git rev-parse --verify master &>/dev/null; then TRUNK="master"
else TRUNK=""; fi
echo "CURRENT=$CURRENT"
echo "TRUNK=$TRUNK"
echo "DETACHED=$DETACHED"
```

3. Route based on the result:
   - **Not a git repo** (commands fail) -> report "Not a git repository." and stop.
   - **Detached HEAD** -> report the hash, run `git log --oneline -10` around that point, summarize, and stop.
   - **No trunk found** (`TRUNK` is empty) -> use `AskUserQuestion` to ask the user for their trunk branch name, then continue.
   - **On trunk** (`CURRENT == TRUNK`) -> go to Step 2B.
   - **Feature branch** (anything else) -> go to Step 2A.

### Step 2A: Feature Branch

Run a single Bash call to collect all data at once:

```bash
MERGE_BASE=$(git merge-base HEAD $TRUNK)
echo "=== LOG ==="
git log --format="%h %s%n%w(0,4,4)%b" $MERGE_BASE..HEAD
echo "=== STAT ==="
git diff --stat $MERGE_BASE..HEAD
echo "=== SHORTSTAT ==="
git diff --shortstat $MERGE_BASE..HEAD
echo "=== MERGE_BASE ==="
echo $MERGE_BASE
```

If a focus argument was provided, add path filtering:

- Append `-- '*{focus}*'` to the `git log` and `git diff --stat` commands
- If no results match the focus, report "No changes matching '{focus}'" and stop.

**Edge case — no divergence:** If `git log` returns nothing (merge base == HEAD), report "Branch exists but hasn't diverged from {trunk} yet." and stop.

Parse the output silently. Do NOT echo the raw git output. Proceed to Step 3A.

### Step 2B: Trunk Branch

Run a single Bash call:

```bash
echo "=== RECENT ==="
git log --oneline --since="7 days ago" --format="%h %s (%an, %ar)"
echo "=== COUNT ==="
git rev-list --count --since="7 days ago" HEAD
```

Parse the output:

- Filter out routine commits (dependency bumps, CI config, docs-only, formatting, merge commits).
- Group remaining interesting commits by day.
- Count filtered-out commits separately.

**Edge case — no recent activity:** If count is 0, expand the window to 30 days and re-run. If still 0, report "Quiet repo -- no commits in the last 30 days." and stop.

Parse the output silently. Do NOT echo the raw git output. Proceed to Step 3B.

### Step 3A: Present (Feature Branch)

Output in this exact format:

```markdown
## WTF: {branch-name}

**Purpose:** {one-sentence description of what this branch does, inferred from commit messages}
**Base:** diverged from {trunk} at {merge-base-short} ({N} commits ahead)
**Scope:** {files changed} files | +{insertions} -{deletions}

### Changes

- {2-5 bullets summarizing logical changes, grouped by concern}

### Key Files

| File   | Change | What                   |
| ------ | ------ | ---------------------- |
| {path} | +n/-n  | {one-line description} |

---

**Next:**

1. Show the full diff
2. Deep-dive into a specific file
3. `/check` something specific
```

### Step 3B: Present (Trunk Branch)

Output in this exact format:

```markdown
## WTF: {trunk-branch}

{N} commits in the last 7 days.

### Highlights

**{Day}**

- {hash} {summary} ({author})

### Filtered Out

{N} routine commits omitted (deps, CI, docs).

---

**What do you want to look at?**

1. A specific commit
2. A different time range
3. Changes by a specific author
4. A specific area of the codebase
```

## Guidelines

- This skill is strictly read-only. Never suggest running a write operation.
- No subagents. This skill is too simple for Task delegation.
- Parse git output silently and present structured markdown. Never dump raw git output to the user.
- Keep the total output concise. This is a quick orientation, not a full audit.
- If focus argument is provided, scope everything to that filter. Report clearly when the filter has no matches.
- For the Key Files table in feature branch output, limit to the 8 most-changed files. If there are more, add a summary row like "... and {N} more files".
