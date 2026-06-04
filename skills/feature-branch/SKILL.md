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

## Workflow

Follow all 5 steps sequentially. `Read`, `Write`, `Edit`, `Glob`, and `Grep` are intentionally excluded from allowed-tools -- this skill must not read or modify any files, and `git checkout -b` is the only mutating command.

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
echo "=== TRUNK_SHA ==="
for t in main master trunk develop; do
  if git rev-parse --verify "$t" &>/dev/null; then
    git rev-parse --short "$t"
    break
  fi
done
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
- Do not drop words selectively -- the slug is mechanically derived from the description.

**Compose:** `<prefix><slug>` (e.g. `feat/add-rate-limiting`, `nathan/add-rate-limiting`).

**Name collision check:** scan `BRANCHES` for an exact match against the proposed name. If found, prepare a `-2` suffix as an alternative for Step 4.

### Step 4: Confirm and create

Apply the following rules in order; the first matching rule fires. Multiple rules may contribute (e.g. a dirty tree note plus a convention picker), but only one `AskUserQuestion` call should run per turn.

| Order | Situation | Action |
| --- | --- | --- |
| 1 | Step 3 flagged multiple plausible conventions | `AskUserQuestion` listing exactly the patterns Step 3 flagged, with `feat/<slug>` as a fallback when fewer than 3 are flagged. |
| 2 | Proposed name already exists locally | `AskUserQuestion` with options `<name>-2` / switch to existing branch / cancel. |
| 3 | Convention clear, no collision | Print preview and run `git checkout -b <name> <trunk>` directly. |

If the working tree is dirty, prepend a one-line note to the preview ("Note: uncommitted changes will carry over") in any of the rows above. Do not ask -- the user has already seen the preview and can interrupt.

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

On success, output a single markdown block using `TRUNK_SHA` captured in Step 2:

```markdown
## Branch created

- **Name:** `feat/add-rate-limiting`
- **From:** `main` (abc1234)
- **Convention:** `feat/` (matched 5 of 7 sibling branches)
```

## Guidelines

- **Always render the picker for enumerated options.** Whenever a step lists 2-4 selectable choices (e.g. trunk candidates, collision alternatives, multiple plausible conventions), you MUST invoke `AskUserQuestion` so the user sees a pre-filled picker. Never print numbered options in markdown and wait -- the user cannot select from inline text. For genuine free-text inputs, print the question and stop the turn instead of calling `AskUserQuestion` with invented options.
- Only one mutating command (`git checkout -b`). Never `git fetch`, `git pull`, `git push`, or `git branch -d/-D`.
- Branch is created from the **local** trunk ref. If the user wants the latest remote, they can pull first.
- Parse git output silently and present structured markdown. Never dump raw git output to the user.
- If anything fails, surface the real git error rather than retrying with guesses.
