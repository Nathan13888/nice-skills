---
name: feature-branch
description:
  Create a new git branch off trunk using the project's existing naming
  convention. Detects trunk (main/master/etc.) and the dominant prefix
  pattern (feat/, <username>/, etc.) from existing branches, slugifies
  the feature description, and runs git checkout -b. Use when user says
  "feature branch", "new branch", "create branch", "git branch", or
  "/feature-branch".
allowed-tools:
  - Bash
  - AskUserQuestion
---

# Feature Branch

One-shot helper that turns `/feature-branch <feature description>` into a properly named branch checked out from trunk. Detects the trunk branch and the local naming convention from a single `git branch -vv` call, slugifies the description, and runs `git checkout -b`.

`Read`, `Write`, `Edit`, `Glob`, and `Grep` are intentionally excluded from allowed-tools. This skill MUST NOT read or modify any files. The only mutating action is `git checkout -b`.

## Workflow

Follow all 4 steps sequentially.

### Step 1: Parse the argument

Take everything after `/feature-branch` as the raw feature description. Examples:

- `/feature-branch add rate limiting` -> description is "add rate limiting"
- `/feature-branch` (bare) -> description is empty

If the description is empty, print "What is the new branch for? Describe the feature in a short phrase." and **stop the turn**. Treat the user's next message as the description.

Do not slugify yet -- Step 3 needs the convention information from Step 2 first.

### Step 2: Single Bash call

Collect every piece of data needed in one shot. Do not split this into multiple Bash calls.

```bash
IS_GIT=$(git rev-parse --is-inside-work-tree 2>&1)
echo "=== IS_GIT ==="
echo "$IS_GIT"
echo "=== CURRENT ==="
git symbolic-ref --short HEAD 2>/dev/null || echo "DETACHED"
echo "=== BRANCHES ==="
git branch -vv --all 2>/dev/null
echo "=== TRUNK_REMOTE ==="
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null
echo "=== TRUNK_LOCAL ==="
for t in main master trunk develop; do
  if git rev-parse --verify "$t" &>/dev/null; then
    echo "$t"
    break
  fi
done
echo "=== USER ==="
git config user.name
echo "=== EMAIL ==="
git config user.email
echo "=== STATUS ==="
git status --porcelain 2>/dev/null | head -20
```

Route on the result:

- **Not a git repo** (`IS_GIT` is not `true`): report "Not a git repository." and **STOP**.
- **Detached HEAD** (`CURRENT == DETACHED`): report "HEAD is detached. Check out a branch before creating a new one." and **STOP**.
- Otherwise continue to Step 3.

Parse the rest of the output silently. Never echo raw git output back to the user.

### Step 3: Decide trunk and convention (silently)

**Trunk selection (in priority order):**

1. If `TRUNK_REMOTE` is non-empty (e.g. `origin/main`), strip the `origin/` prefix and use that as trunk.
2. Else, if `TRUNK_LOCAL` is non-empty, use it.
3. Else, use `AskUserQuestion` to ask which local branch is trunk. Offer up to 4 candidates from the local branches in `BRANCHES`.

**Convention detection from `BRANCHES`:**

1. Strip the leading `*` / whitespace and the trailing tracking info from each line. Keep only the branch name.
2. Drop trunk itself, any `remotes/origin/HEAD ->` line, and the current `HEAD` entry. Dedupe `remotes/origin/X` against local `X` (treat them as one branch).
3. From the remaining names, count two kinds of prefixes:
   - **Type prefixes:** `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`, `test/`, `perf/`, `style/`.
   - **Username prefix:** `<word>/...` where `<word>` matches the sanitized `git config user.name` (lowercased, non-alphanumeric -> `-`), OR any short (<=20 chars) lowercase word that appears as a prefix in 2+ branches and is not in the type-prefix list.
4. Decide the dominant convention:
   - If one pattern matches >= 60% of the sibling branches AND there are >= 3 sibling branches: that's the convention.
   - If < 3 sibling branches or the split is roughly even: fall back to `feat/`.
   - If both a type pattern and a username pattern are plausible: flag for confirmation in Step 4.

**Slugify the description:**

- Lowercase.
- Replace each run of non-alphanumeric characters with a single `-`.
- Strip leading/trailing `-`.
- Truncate at 50 characters; if the truncation lands mid-word, cut back to the last `-`.
- Keep meaningful short words (don't aggressively drop articles like "a"/"the" -- readability matters more than terseness).

**Compose:** `<prefix><slug>` (e.g. `feat/add-rate-limiting`, `nathan/add-rate-limiting`).

**Name collision check:** scan `BRANCHES` for an exact match against the proposed name. If found, prepare a `-2` suffix as an alternative for Step 4.

### Step 4: Confirm and create

Decide whether to ask or proceed directly:

| Situation | Action |
| --- | --- |
| Convention clear, no collision, working tree clean | Print preview + run `git checkout -b <name> <trunk>` directly. |
| Multiple plausible conventions | `AskUserQuestion`: list up to 4 candidates (e.g. `feat/<slug>`, `<username>/<slug>`, any other detected dominant pattern). |
| Name already exists locally | `AskUserQuestion`: offer `<name>-2`, switch to existing branch, or cancel. |
| Working tree has uncommitted changes | Add a one-line warning in the preview ("Note: uncommitted changes will carry over") but proceed unless the user cancels. |
| No trunk identified | `AskUserQuestion`: list detected local branches as trunk candidates. |

**Preview format (before creating):**

```markdown
**Proposed branch:** `feat/add-rate-limiting`
**From trunk:** `main`
**Convention:** `feat/` (5 of 7 sibling branches)
```

**Create the branch:**

```bash
git checkout -b "<name>" "<trunk>"
```

If `git checkout -b` fails, report the exact stderr and **STOP**. Do not retry with a different name without asking.

### Step 5: Report

On success, output a single markdown block:

```markdown
## Branch created

- **Name:** `feat/add-rate-limiting`
- **From:** `main` (abc1234)
- **Convention:** `feat/` (matched 5 of 7 sibling branches)
```

Use `git rev-parse --short <trunk>` to fill in the short SHA if it wasn't already captured.

## Guidelines

- **Always render the picker for enumerated options.** Whenever a step lists 2-4 selectable choices (e.g. trunk candidates, collision alternatives, multiple plausible conventions), you MUST invoke `AskUserQuestion` so the user sees a pre-filled picker. Never print numbered options in markdown and wait -- the user cannot select from inline text. For genuine free-text inputs, print the question and stop the turn instead of calling `AskUserQuestion` with invented options.
- Only one mutating command (`git checkout -b`). Never `git fetch`, `git pull`, `git push`, or `git branch -d/-D`.
- Branch is created from the **local** trunk ref. If the user wants the latest remote, they can pull first.
- No subagents. No file reads. No editing.
- Parse git output silently and present structured markdown. Never dump raw git output to the user.
- If anything fails, surface the real git error rather than retrying with guesses.
