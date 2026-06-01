# Skills Catalog

A deeper map of the skills that ship in this repository. Use this when you are deciding _which_ skill fits a task, when you want to understand how two similar skills differ, or when you are weighing whether a proposed new skill earns its place. The root [README](./README.md) handles the elevator pitch and install commands; this document is for understanding the shape of the collection.

## At a glance

| Skill                               | One-line purpose                                                        | Category               |
| ----------------------------------- | ----------------------------------------------------------------------- | ---------------------- |
| [`check`](#check)                   | Verify a specific expectation about the current work                    | Verification & context |
| [`combine-docs`](#combine-docs)     | Consolidate a scattered doc landscape into one source of truth          | Documentation          |
| [`doc-n-fix`](#doc-n-fix)           | Turn PR review comments into a checklist and work it down               | Code workflow          |
| [`explore`](#explore)               | Map a project's capabilities from a solutions-architect lens            | Discovery & analysis   |
| [`feature-branch`](#feature-branch) | Create a new git branch off trunk using the project's naming convention | Code workflow          |
| [`init-repo`](#init-repo)           | Scaffold a new repo or add ops tooling to an existing one               | Planning & setup       |
| [`report`](#report)                 | Capture, verify, and route a bug/feature/proposal report                | Code workflow          |
| [`mvp`](#mvp)                       | Design product scope and feature plan before any code                   | Planning & setup       |
| [`readme`](#readme)                 | Generate a high-quality, type-aware README for one repo                 | Documentation          |
| [`send-it`](#send-it)               | Push a branch and open a structured GitHub PR in one shot               | Code workflow          |
| [`sus`](#sus)                       | Hunt architectural and maintainability problems via parallel agents     | Discovery & analysis   |
| [`tackle`](#tackle)                 | Turn arbitrary feedback into a dispatched action plan                   | Code workflow          |
| [`wtf`](#wtf)                       | Quick situational awareness for the current branch or trunk             | Verification & context |

## Categories

The collection clusters into five roles. **Verification & context** answers narrow questions about the current state of work. **Discovery & analysis** explores a codebase at scale — `explore` catalogs what's there, `sus` flags what's broken. **Code workflow** automates the branch-and-PR loop, from branching (`feature-branch`) through review comments (`doc-n-fix`) and shipping (`send-it`). **Planning & setup** front-loads a project — `mvp` decides _what_ to build, `init-repo` decides _how_ the repo is wired. **Documentation** writes new docs (`readme`) or rationalises existing ones (`combine-docs`).

A few skills look similar at a glance and are worth telling apart up front:

- **`explore` vs `sus`** — both deploy parallel agents across a codebase. `explore` builds a capabilities map (what does this project _do_?). `sus` hunts problems (what's _wrong_ with this project?). Read-only either way.
- **`check` vs `wtf`** — both are fast and read-only. `check` answers one targeted question with evidence ("does the auth handler validate input?"). `wtf` orients you to a whole branch or recent trunk activity without touching files outside git.
- **`readme` vs `combine-docs`** — `readme` writes a single high-quality README for one repo. `combine-docs` refactors a sprawling collection of docs into a coherent set.
- **`init-repo` vs `mvp`** — both fire at project start. `mvp` is product-first: what to build, for whom, in what order. `init-repo` is technical: git, CI, linting, license, language toolchain. They compose well.
- **`report` vs `doc-n-fix`** — `doc-n-fix` consumes existing GitHub review comments and fixes them. `report` originates a new report from a description and routes it (upload, document, or hand off for a fix).
- **`tackle` vs `report`** — `report` produces a portable report artifact (upload/document/handoff). `tackle` executes against the feedback; the artifact is the diff, not the report.
- **`tackle` vs `doc-n-fix`** — `doc-n-fix` is GitHub-PR-bound and pulls review comments via `gh` API. `tackle` accepts any source (URL from any host, paste, image, file path, free text) and works without a PR.
- **`feature-branch` vs `send-it`** — both touch git workflow. `feature-branch` opens a unit of work (cuts a fresh branch off trunk with a project-appropriate name). `send-it` closes it (pushes and opens the PR). They bookend the same loop.

## Skills

### `check`

**What it is** — A read-only verification skill that answers one specific question about the current work using git state and the codebase.
**Why it exists** — A busy engineer often just wants a yes/no with evidence, not a full audit or an exploratory tangent.
**How to use it** — `/check <question>`, e.g. `/check does the migration backfill the new column?`. Gathers status, diffs, recent commits, and reads the relevant files; returns a verdict and 3-5 supporting bullets.
**Unique among these skills** — Single targeted question, never modifies anything. `sus` hunts unknown problems; `explore` maps capabilities; `check` confirms one specific claim.

### `combine-docs`

**What it is** — Finds every doc in the repo, identifies overlap and contradictions, proposes a consolidation plan, then executes it with approval.
**Why it exists** — Projects accumulate README + CONTRIBUTING + docs/setup.md that all explain the same thing slightly differently; this skill is the cleanup pass.
**How to use it** — `/combine-docs`. Globs `.md`/`.mdx`/`.txt`, builds an overlap map, presents a table of proposed actions (MERGE / MOVE / UPDATE / CREATE / ARCHIVE / SKIP), and waits for approval before changing anything. Uses `git mv` to preserve history.
**Unique among these skills** — The only skill that actively _refactors_ documentation. `readme` writes a single doc from scratch.

### `doc-n-fix`

**What it is** — Pulls PR review comments, organises them into a prioritised checklist, documents the checklist somewhere visible, then fixes items one-by-one and updates progress as it goes.
**Why it exists** — Review comments scatter across files, threads, and authors; without a single living checklist they get half-addressed or forgotten.
**How to use it** — `/doc-n-fix`, `/doc-n-fix 42` for a specific PR, or `/doc-n-fix --manual` to paste feedback. Deduplicates via the `gh` API, classifies by P1/P2/P3 and S/M/L effort, lets you pick a destination (PR description, PR comment, or local file), then walks the list.
**Unique among these skills** — Bridges _feedback_ and _fix_; the checklist is the deliverable, the fixes update it in place. `send-it` ships PRs; `doc-n-fix` resolves them.

### `explore`

**What it is** — Strategic discovery: parallel agents map architecture, inventory capabilities, and assess infrastructure, then synthesise a capabilities report with improvement recommendations.
**Why it exists** — A new joiner or solutions architect needs to understand _what a project can do_ without reading every file.
**How to use it** — `/explore`, or `/explore focus: auth` to narrow. Launches three agents in parallel — Architecture Cartographer, Capability Analyst, Infrastructure Scout — and merges results into a capabilities table plus ranked improvement suggestions.
**Unique among these skills** — Read-only, capability-focused, architect's lens. Contrast with `sus` (problem-focused) and `wtf` (single-branch git-only).

### `feature-branch`

**What it is** — A one-shot helper that turns `/feature-branch <feature description>` into a properly named branch checked out from trunk.
**Why it exists** — Cutting a branch by hand means re-checking which trunk this repo uses (`main`? `master`?), guessing the local naming convention (`feat/...`? `<username>/...`?), and slugifying the description without typos. This skill collapses all three into one command.
**How to use it** — `/feature-branch add rate limiting`, `/feature-branch` (will prompt for the description). Runs a single `git branch -vv --all` plus a few inline lookups to identify trunk and read the dominant prefix pattern from sibling branches, then runs `git checkout -b`. Confirms via `AskUserQuestion` only when the convention is ambiguous or a name collides.
**Unique among these skills** — The only skill that _opens_ a unit of work. `send-it` closes it; `wtf` orients you to one already in flight; `feature-branch` starts a new one. Strictly local — never fetches, pulls, or pushes.

### `init-repo`

**What it is** — Scaffolds a new repo for agentic development, or retrofits ops tooling (git, precommits, gitignore, license, CI) onto an existing project.
**Why it exists** — Every new project needs the same boring scaffolding done correctly; this skill enforces a consistent baseline across runtimes.
**How to use it** — `/init project` or `/new project`, optionally seeded with a problem description. Phase 1 is a single bulk intake form (name, runtime, package manager, test runner, CI platform, license, project kind, etc.). Phase 2 executes 12 sequential setup steps and writes CLAUDE.md plus a README via the `readme` skill.
**Unique among these skills** — Multi-runtime aware (Node/Bun, Python, Rust, Go) with ecosystem-appropriate conventions. The intake form is bulk to minimise back-and-forth. Pairs naturally with `mvp` (run `mvp` first to decide _what_ the project is).
**Notable internals** — A single large `SKILL.md` (~68KB) with runtime-specific dependency lists and CI templates inlined.

### `report`

**What it is** — Turns a free-text problem report into a structured report artifact, verifies it lightly against the codebase, then routes the result to one of four destinations: upload to GitHub/GitLab, document in code, hand off for an implementation, or a free-text custom action.
**Why it exists** — Reporting a problem usually means writing a half-formed bug report, then context-switching to GitHub to file it, then back to the editor to TODO it, then maybe trying to fix it. This skill collapses that loop into one confirmed flow.
**How to use it** — `/report <description>`, e.g. `/report auth middleware doesn't validate JWT expiry in src/auth.ts`. Light verification reads referenced files; for descriptions naming 3+ files or "across the codebase" wording, escalates to a Task agent. Drafts a structured report (bug/enhancement/proposal/question template), confirms with you, then asks for one of four next steps. Uses `gh` for GitHub and `glab` for GitLab.
**Unique among these skills** — Originates new reports from a description. `doc-n-fix` works the other direction (existing review comments → fixes). `send-it` ships PRs, not issues. `check` only verifies one specific claim without producing a portable artifact.

### `mvp`

**What it is** — Product design and scope-management skill for new projects. Gathers requirements deeply, ideates features with help from sub-agents, locks scope, then writes a structured set of planning artifacts.
**Why it exists** — MVPs die from feature creep and vague scope; this skill front-loads the deliberate-thinking work and commits decisions to writing.
**How to use it** — `/mvp`, `/mvp <seed description>`, or `/mvp --resume`. Enters plan mode, walks problem space → workflows → architecture → feature ideation (2 parallel agents) → scope lock with approval gate. Only after approval does it write 5 artifacts: OVERVIEW, FEATURES, MVP_SCOPE, ROADMAP, DELIVERABLES.
**Unique among these skills** — Product-first, not technical. Uses plan mode as a discipline to defer file writes until scope is genuinely agreed. Pairs with `init-repo` (run `mvp` first, then `init-repo` to wire the scaffold).
**Notable internals** — `AGENT_PROMPTS.md` holds the prompt templates for the two feature-ideation agents (Product Architect, Domain Explorer).

### `readme`

**What it is** — Generates a well-targeted README for one repository, adapting shape to the repo _type_ (library, CLI, web app, API, monorepo, framework, plugin).
**Why it exists** — A good README is the first impression; bad README templates produce hype-laden, generic, or wrong-shaped docs. This skill makes the output match the repo.
**How to use it** — `/readme` or "generate a README". Detects language/ecosystem from marker files, classifies repo type via heuristics, assembles a type-appropriate skeleton, writes in a direct (not hype-y) tone, and validates against a checklist.
**Unique among these skills** — Type-aware: a CLI README looks different from a library README. `combine-docs` rationalises _many_ docs; `readme` produces one good one.
**Notable internals** — Six supporting template files in the skill directory: `application.md`, `cli.md`, `framework.md`, `library.md`, `monorepo.md`, and `language-conventions.md`.

### `send-it`

**What it is** — Pushes the current feature branch and opens a structured GitHub PR in a single, confirmed flow.
**Why it exists** — Opening a good PR involves git push, upstream tracking, branch checks, commit-history reading, and a clear title + body. Sequencing all of that by hand is error-prone.
**How to use it** — `/send it`, `/push and PR`, `/ship it`. Preflights (auth, branch, existing PR), collects logs and diff in parallel, drafts a title (no conventional-commit prefix) and a 4-section body (one-liner, Summary, Implementation, Followup), shows it, then pushes and creates the PR after confirmation. Offers draft vs ready.
**Unique among these skills** — The one skill dedicated to shipping. Pairs with `doc-n-fix` after the PR receives reviews.

### `sus`

**What it is** — Code-quality auditor. Deploys parallel analysis agents to surface architecturally problematic, suspicious, or high-impact maintainability issues, then ranks and caps findings at 10.
**Why it exists** — Manual review is slow and biased; parallel agents looking through different lenses (architecture, complexity, coupling, consistency, risk) catch more high-impact issues with less attention from the human.
**How to use it** — `/sus`, `/find problems`, or `/sus focus: security`. Runs a hard compilation gate first — if the build fails, it stops. Then confirms the agent roster (default five, customisable), deploys in parallel, deduplicates, ranks by severity and cross-agent agreement, and reports.
**Unique among these skills** — Problem-hunting, not style-checking; explicitly filters out lint-level findings. Contrast with `explore` (capabilities, not problems) and `check` (one specific question, not open-ended).
**Notable internals** — `AGENT_ROSTER.md` (5 default + 4 optional specialists) and `SEVERITY_GUIDE.md` (defines CRITICAL/MAJOR and what gets filtered out).

### `tackle`

**What it is** — Ingests arbitrary feedback about the current repo (GitHub/GitLab URL, pasted reviewer comments, an image, a file path, free text, or any combination), decomposes it into a prioritised action plan with per-item owners (human / main-agent / subagent), confirms with the user, then dispatches each item to the right executor.
**Why it exists** — Reviewer feedback arrives in shapes the editor can't natively act on — a Slack snippet, a screenshot from a designer, an issue URL from a sibling repo, a half-rambling rant. Without a single decomposition step you either lose items or context-switch to track them by hand.
**How to use it** — `/tackle <text / URL / file path / image>`, optionally combined. The skill light-verifies references against the codebase, drafts the plan, asks you to confirm, then walks items: subagents for explore/validate/execute jobs, an inline handoff for the parent agent, and a checkbox for anything that needs you. Persists the plan to `/tmp/tackle-{id}.md` and supports `/tackle --resume`.
**Unique among these skills** — Execution-first. `report` ends at the report; `doc-n-fix` is `gh`-PR-bound; `check` is read-only. `tackle` is the only skill that turns _arbitrary_ feedback into dispatched action.

### `wtf`

**What it is** — Quick situational awareness for the current git branch. On a feature branch, summarises what the branch is about. On trunk, highlights recent interesting activity.
**Why it exists** — Switching branches or rejoining a project after time away creates a "what is this even" moment; this skill answers it in one pass without reading source files.
**How to use it** — `/wtf`, `/what's going on`, or `/wtf <focus>`. Detects branch vs trunk; on branches, summarises divergence and key files; on trunk, groups commits by day from the last 7 (or 30) days and filters out routine churn (deps, CI, docs).
**Unique among these skills** — Strictly git-only — no file reads outside what git already shows. Much faster than `explore` for "what just happened" questions. `check` answers a specific question; `wtf` gives you a branch-shaped overview.

## Discrepancies and open questions

- **`/reflect`** is listed in the root `README.md` (line 12) and in the install block, but no `skills/reflect/` directory exists. Either the skill needs to be built or the README references should be removed.

## Maintaining this catalog

- **When you add a new skill**, add a row to the at-a-glance table, an entry under Skills, and — if it belongs to a new bucket — update the Categories prose.
- **Keep entries tight** (~10-15 lines each). The detail belongs in the skill's own `SKILL.md`; this doc is for orientation and contrast.
- **When two skills start overlapping**, make the contrast explicit in the _Unique among these skills_ line of both — that pressure-tests whether you actually need both.
- **Audience**: a contributor or new user deciding which skill to invoke, or whether a proposed new skill duplicates something that already exists.
