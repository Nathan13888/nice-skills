---
name: send-it
description:
  Push feature branch and create a GitHub PR with structured title and
  description. Auto-detects branch context, drafts a structured PR body,
  gets user confirmation, then pushes and creates the PR. GitHub-only,
  uses gh CLI. Use when user says "send it", "push and PR", "create PR",
  "open a PR", "ship it", or "send this branch".
allowed-tools:
  - Bash
  - Write
  - Read
  - Glob
  - Grep
  - AskUserQuestion
---

# Send It

Push the current feature branch and create a structured GitHub PR in one shot. Gathers context from git history, drafts a PR with a clear title and body, confirms with the user, then pushes and opens the PR.

## Workflow

Follow all 5 steps sequentially.

### Step 1: Preflight Checks

Run a single Bash call to detect the full context:

```bash
IS_GIT=$(git rev-parse --is-inside-work-tree 2>&1)
CURRENT=$(git symbolic-ref --short HEAD 2>/dev/null || echo "DETACHED")
if git rev-parse --verify main &>/dev/null; then TRUNK="main"
elif git rev-parse --verify master &>/dev/null; then TRUNK="master"
else TRUNK=""; fi
GH_AUTH=$(gh auth status 2>&1)
GH_AUTH_EXIT=$?
if [ -n "$TRUNK" ] && [ "$CURRENT" != "$TRUNK" ]; then
  AHEAD=$(git rev-list --count ${TRUNK}..HEAD 2>/dev/null || echo "0")
else
  AHEAD="0"
fi
EXISTING_PR=$(gh pr view --json number,url,state 2>&1)
EXISTING_PR_EXIT=$?
echo "IS_GIT=$IS_GIT"
echo "CURRENT=$CURRENT"
echo "TRUNK=$TRUNK"
echo "GH_AUTH_EXIT=$GH_AUTH_EXIT"
echo "GH_AUTH=$GH_AUTH"
echo "AHEAD=$AHEAD"
echo "EXISTING_PR_EXIT=$EXISTING_PR_EXIT"
echo "EXISTING_PR=$EXISTING_PR"
```

Route based on results:

| Condition                                    | Action                                                                                                                                                                            |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Not a git repo (`IS_GIT` is not `true`)      | Report "Not a git repository." and **STOP**                                                                                                                                       |
| `gh` not authenticated (`GH_AUTH_EXIT` != 0) | Report "GitHub CLI not authenticated. Run `gh auth login`." and **STOP**                                                                                                          |
| On trunk (`CURRENT == TRUNK`)                | Report "You're on {trunk} -- switch to a feature branch first." and **STOP**                                                                                                       |
| Detached HEAD (`CURRENT == "DETACHED"`)      | Report "Detached HEAD -- checkout a branch first." and **STOP**                                                                                                                    |
| No trunk found (`TRUNK` is empty)            | Use `AskUserQuestion` to ask the user for their trunk branch name, then continue                                                                                                  |
| No commits ahead (`AHEAD == 0`)              | Report "No commits ahead of {trunk} -- nothing to send." and **STOP**                                                                                                              |
| PR already exists (`EXISTING_PR_EXIT == 0`)  | Use `AskUserQuestion`: "PR already exists at {url} ({state}). Options: (1) Update existing PR description, (2) Continue and create a new PR anyway, (3) Stop." Route accordingly. |

If all checks pass, proceed to Step 2.

### Step 2: Gather Context

Compute the merge base, then run parallel Bash calls to collect data:

```bash
MERGE_BASE=$(git merge-base HEAD $TRUNK) && echo "MERGE_BASE=$MERGE_BASE"
```

Then in parallel:

```bash
git log --format="%h %s%n%w(0,4,4)%b" ${MERGE_BASE}..HEAD
```

```bash
git diff --stat ${MERGE_BASE}..HEAD
```

```bash
git diff ${MERGE_BASE}..HEAD
```

```bash
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>&1
echo "EXIT=$?"
```

Parse all output silently. Do NOT echo raw git output to the user. Extract:

- List of commits (hash + subject + body)
- Files changed with line counts
- Full diff for understanding changes
- Whether upstream tracking exists

### Step 3: Draft PR

Analyze the commits, diff stat, and full diff to draft the PR.

#### Title

- Capitalized, human-like one-sentence summary of what this branch does
- No conventional commit prefixes (no `feat:`, `fix:`, etc.)
- Under 70 characters
- Examples: "Add user authentication with JWT tokens", "Fix race condition in queue processor", "Replace legacy search with Elasticsearch"

#### Body

Structure the body with exactly these 4 sections:

```markdown
{One-liner: a single sentence describing the overall change. No heading.}

## Summary

- {Bullet points describing what changed and why}
- {Up to 2 levels of nesting for sub-details}
  - {Sub-detail}

## Implementation

1. **{Bold item}**: {Explanation of how this was implemented}
2. **{Bold item}**: {Explanation}

## Followup

- {Tests to add or run}
- {Migrations or deployment steps}
- {Deferred features or known limitations}
- {Other remarks}
```

**Drafting rules:**

- Derive content from commits and diff -- do not fabricate changes
- Summary focuses on _what_ and _why_; Implementation focuses on _how_
- Followup should only include items that are genuinely relevant -- omit the section if there's nothing to note
- Keep the total body under 60 lines

Present the draft to the user:

```markdown
## PR Draft

**Title:** {title}

**Body:**

{full body}

---

Ready to send?
```

### Step 4: User Confirmation

Use `AskUserQuestion` with these options:

1. **Send as draft PR** -- push and create a draft PR
2. **Send as ready PR** -- push and create a ready-for-review PR
3. **Edit** -- user provides changes to title or body

If the user chooses "Edit", apply their changes to the draft and present the updated version. Repeat Step 4 until the user confirms with option 1 or 2.

### Step 5: Push and Create PR

#### Push the branch

Check whether upstream tracking exists (from Step 2):

- **No upstream**: `git push -u origin {branch}`
- **Has upstream**: `git push`

If push fails, report the error and **STOP**. Do not force-push.

#### Create the PR

1. Write the PR body to a temp file using the `Write` tool:

```
Write /tmp/send-it-pr-body.md with the final PR body content
```

2. Create the PR:

For a **draft** PR:

```bash
gh pr create --title "{title}" --body-file /tmp/send-it-pr-body.md --draft
```

For a **ready** PR:

```bash
gh pr create --title "{title}" --body-file /tmp/send-it-pr-body.md
```

3. Capture the PR URL from the command output.

4. Clean up the temp file:

```bash
rm -f /tmp/send-it-pr-body.md
```

#### Final output

```markdown
## Sent!

**Title:** {title}
**URL:** {pr_url}
**Status:** {Draft | Ready for review}
**Branch:** {branch} -> {trunk}
**Commits:** {N} | **Files:** {N} | **Changes:** +{insertions} -{deletions}
```

## Guidelines

- **`Write` tool for temp files** -- use the `Write` tool to create `/tmp/send-it-pr-body.md`. **DO NOT** use `echo`, `cat <<EOF`, or shell redirects. Always pass the body via `--body-file`.
- **No subagents** -- this is a straightforward linear workflow. Do not spawn Task agents.
- **Never push to trunk** -- if somehow on trunk, STOP in Step 1.
- **Never force-push** -- if `git push` fails, report the error and stop. Let the user decide how to resolve.
- **Clean up temp files** -- always `rm -f /tmp/send-it-pr-body.md` after PR creation, even if the `gh` command fails.
- **Respect user edits** -- when the user modifies the draft, apply changes exactly as specified. Do not re-derive the content from scratch.
- **Title quoting** -- when passing the title to `gh pr create`, ensure it's properly quoted to handle special characters.
- **Existing PR handling** -- if updating an existing PR, use `gh pr edit {number} --title "{title}" --body-file /tmp/send-it-pr-body.md` instead of `gh pr create`.
