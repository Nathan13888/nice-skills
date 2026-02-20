---
name: combine-docs
description: Discovers all documentation in a project, analyzes it for redundancy,
  contradictions, and staleness, proposes a consolidation plan, then executes it.
  Produces optimized docs for both humans and agents. Use when user says "combine docs",
  "consolidate documentation", "merge docs", "clean up docs", "fix documentation",
  "optimize docs", "documentation sprawl", "too many docs", or asks about doc organization.
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

# Documentation Consolidator

Discover all documentation in a project, analyze for redundancy and contradictions, propose a consolidation plan, then execute it with user approval.

Follow all 4 steps sequentially. Do NOT execute changes without user approval at Step 3.

## Step 1: Discover

Find every documentation file in the project.

1. **Glob for docs** — search for `**/*.md`, `**/*.mdx`, `**/*.txt`, `**/*.rtf` (exclude `node_modules/`, `.git/`, `vendor/`, `target/`, `dist/`, `build/`, `__pycache__/`, `archive/`).

2. **List what you found** — output each file with its path. Note how many docs exist total.

3. **Flag monorepo READMEs separately** — if the project has READMEs inside subdirectories (packages, modules, services), list them in their own group. These need extra care in Step 3.

## Step 2: Map & Categorize

Read each doc and build a landscape table for the user.

**For each doc, note:**
- Path
- Type: readme / guide / reference / changelog / config / adr / agent-config / other
- What it covers (brief summary)
- Overlaps with other docs (list paths)

**Then identify:**
- **Duplicated sections** — same content in multiple files (e.g., setup steps in README + CONTRIBUTING + docs/getting-started)
- **Contradictions** — different files say different things about the same topic

**Present the map:**

```
## Documentation Map: {project}

**Documents found:** {count}

| Path | Type | Covers | Overlaps With |
|------|------|--------|---------------|
| README.md | readme | setup, overview | CONTRIBUTING.md (setup) |
| ... | ... | ... | ... |

### Duplicated Content
- **Setup instructions** appear in: README.md, CONTRIBUTING.md, docs/getting-started.md

### Contradictions
- README.md says `npm start`, CONTRIBUTING.md says `yarn dev`
```

Ask the user using `AskUserQuestion` with `multiSelect: true`:
- Are any of these docs **auto-generated** or **externally published** (should not be edited)?
- Are any docs **off-limits** for other reasons?

## Step 3: Plan Structure

Propose where docs should live and what changes to make.

**For each doc, assign an action:**

| Action | Meaning |
|--------|---------|
| **MERGE** | Combine multiple docs into one target |
| **MOVE** | Relocate to a better path (uses `git mv`) |
| **UPDATE** | Rewrite or refresh in place |
| **CREATE** | New doc (usually CLAUDE.md) |
| **ARCHIVE** | Move to `archive/` — never delete |
| **SKIP** | Leave as-is, with reason |

**Present the plan:**

```
## Consolidation Plan

| # | Action | Target | Details |
|---|--------|--------|---------|
| 1 | MERGE | docs/setup.md | README setup + CONTRIBUTING setup |
| 2 | UPDATE | README.md | Remove duplicated setup, link to docs/setup.md |
| 3 | CREATE | CLAUDE.md | Commands, architecture, conventions |
| 4 | ARCHIVE | docs/old-api.md | Superseded by docs/api.md |
| 5 | SKIP | LICENSE | No changes needed |
```

**Hard gate:** Ask the user to approve, modify, or reject using `AskUserQuestion`:
- **Approve and execute** (Recommended)
- **Modify** — user adjusts specific actions
- **Reject** — stop, no changes

**If uncertain about any monorepo README or module-level doc, ask the user explicitly before including it in the plan.**

Do NOT proceed to Step 4 without explicit approval.

## Step 4: Execute & Summarize

Execute approved actions in order: ARCHIVE first, then MERGE, then UPDATE, then CREATE, then MOVE.

**Execution rules:**
- **ARCHIVE** — `git mv {path} archive/{path}`. Add a header: `<!-- Archived by combine-docs. Superseded by {replacement}. -->`
- **MERGE** — Read all sources. Combine content — keep all unique information, deduplicate overlapping sections, resolve contradictions (prefer fresher source; ask user if unclear).
- **UPDATE** — Apply specified changes.
- **CREATE** — For CLAUDE.md: commands first, terse, structured, under 300 lines. For other docs: scannable headings, examples over prose.
- **MOVE** — `git mv` so git tracks history.

**After all actions,** grep for broken internal links (`](`, `href=`) pointing to moved/archived files and fix them.

**Present results:**

```
## Consolidation Complete

### Changes

| Before | After | What Changed |
|--------|-------|-------------|
| README.md (setup + overview) | README.md (overview only) | Setup moved to docs/setup.md |
| docs/old-api.md | archive/docs/old-api.md | Archived |
| (new) | CLAUDE.md | Created with commands, architecture |

### What's Next
1. **Review changes** — walk through each modified file
2. **Undo** — revert all changes (`git checkout`)
3. **Commit** — commit as a single commit
```

## Style Guide

**General principle:** Same information in exactly one place. Links replace duplication.

**CLAUDE.md (agent-optimized):**
- Commands section first — build, test, lint, deploy
- Terse: `Run tests: npm test` not a paragraph about the test framework
- Structured: bullet lists and tables, not prose
- Under 300 lines total

**Human docs:**
- Scannable headings — a reader skimming should understand the structure
- Examples over explanations — show, don't tell
- One doc per concern — setup, API, architecture each get their own file
- Reference config files instead of duplicating their content

**Monorepo READMEs:**
- Be careful — module READMEs often serve a real purpose for that module's users
- Keep module-specific content in module READMEs
- Only consolidate if content is truly duplicated project-wide info
- When in doubt, ask the user

## Rules

- **Never delete docs** — archive to `archive/` if obsolete
- **Don't remove content when merging** — combine duplicated sections, keep unique info from each source
- **If uncertain whether a change is safe, ask the user**
- **Use `git mv` for all moves** so git tracks history
- **Always present the summary** at the end with before/after changes
