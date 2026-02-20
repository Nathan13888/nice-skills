---
name: sus
description:
  Finds suspicious, architecturally problematic, or high-impact maintainability
  issues in a codebase. Deploys parallel analysis agents to explore code, then synthesizes
  findings into a prioritized report. Use when user says "find problems", "audit code",
  "what's sus", "code review the repo", "find tech debt", or asks about code quality.
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

# Suspicious Code Finder

Deploy parallel analysis agents to find architecturally problematic, suspicious, or high-impact maintainability issues in a codebase. Produces a prioritized report of at most 10 findings.

## Quick Start

Follow all 5 steps sequentially. Do NOT skip the compilation gate.

## Workflow

### Step 1: Verify Compilation

Before analyzing code quality, confirm the project actually builds. This is a hard gate.

1. Detect the build system by checking for these files (in order):
   - `Cargo.toml` -> `cargo check`
   - `go.mod` -> `go build ./...`
   - `tsconfig.json` with `package.json` -> check for build script in package.json, run it (e.g., `tsc --noEmit`, `npm run build`)
   - `package.json` (no tsconfig) -> check for build script, run if present
   - `pyproject.toml` -> `python -m py_compile` on main entry or `python -m compileall src/`
   - `Makefile` -> `make` (default target, or `make check` if available)
   - `CMakeLists.txt` -> `cmake --build .`

2. Run the compile/build command.

3. **Hard gate**: If compilation fails, report the errors and STOP. Do not proceed to analysis.

   ```
   ## Compilation Failed

   The project does not compile. Fix these errors before running a code quality audit:

   {error output}
   ```

4. Lint warnings, style warnings, and CI checks are irrelevant here. Only "does the code compile/parse?" matters.

5. If no build system is detected (e.g., standalone scripts, config files), ask the user using `AskUserQuestion`:
   - **Provide a build command** — let the user specify how to compile/check the project
   - **Skip compilation gate** — proceed directly to analysis

### Step 2: Determine Agent Roster

1. **Determine the candidate roster.** Check in this priority order:
   - **Inline overrides** from the user's invocation:
     - `/sus focus: security` -> swap default roster for security-focused agents
     - `/sus agents: Architecture Scout, Security Sentinel, Performance Oracle` -> use exactly those agents
     - `/sus add: Concurrency Analyst` -> default roster + the specified agent
   - **User's harness configuration**: Check if the user has an `/agents` skill or agent configuration that overrides the default roster.
   - **Default roster**: If no customization, use the 5 default agents from the agent table in Step 3.

2. **Present the roster for confirmation.** Use `AskUserQuestion` with `multiSelect: true` to show the candidate agents. List each agent with a short description so the user can disable any they don't want. All agents should be presented as options — the user can select any combination or select all.

   Example framing: "Which agents should I deploy? (select all that apply)"
   - Architecture Scout — module boundaries, layer violations, god modules
   - Complexity Hunter — deep nesting, long functions, complex conditionals
   - Coupling Detector — hidden deps, shotgun surgery, feature envy
   - Consistency Auditor — mixed patterns, half-migrated code, convention drift
   - Risk Assessor — missing error handling, unsafe casts, cascading failures

   If the user's roster includes optional specialists, list those too.

3. **Deploy only the selected agents** in Step 3.

### Step 3: Deploy Agents in Parallel

Launch the user-selected agents simultaneously using the `Task` tool.

#### Agent Roster

Each agent below has a `subagent_type`, focus area, and exploration strategy. Use this information to construct the prompt for each agent's `Task` call.

**Architecture Scout** (`Plan`)

- **Focus:** Structural integrity — module boundaries, responsibility placement, dependency direction.
- **Look for:** Misplaced responsibilities (business logic in controllers, I/O in domain models), layer violations, circular dependencies, god modules (5+ distinct concerns), entrypoints that do too much.
- **Strategy:** Start with directory tree and module structure. Read entry points, follow the import graph. Focus on boundaries between modules.

**Complexity Hunter** (`Explore`)

- **Focus:** Code disproportionately hard to understand, modify, or test.
- **Look for:** Deep nesting (4+ levels), long functions (50+ lines of logic), complex conditionals (3+ boolean conditions, nested ternaries), boolean/string flag parameters, functions with 5+ parameters, implicit state machines as if/else chains.
- **Strategy:** Search for long files first, scan for nesting and complex conditionals. Prioritize most-imported files.

**Coupling Detector** (`Explore`)

- **Focus:** Hidden dependencies and tight coupling that make changes expensive.
- **Look for:** Shotgun surgery (changing one behavior touches 4+ files), feature envy, abstraction leaks, implicit contracts (ordering/naming/magic values without enforcement), shared mutable state, connascence of meaning.
- **Strategy:** Examine imports across module boundaries. Find functions that take or return types from other modules. Search for shared constants used across many files.

**Consistency Auditor** (`general-purpose`)

- **Focus:** Inconsistency signaling unfinished migrations or unclear conventions.
- **Look for:** Same problem solved 3+ ways, half-migrated patterns, convention drift, naming inconsistency that creates real confusion across module boundaries, mixed paradigms without clear boundaries.
- **Strategy:** Sample files across directories. Compare how common operations (error handling, logging, data access, validation) are implemented in different parts of the codebase.

**Risk Assessor** (`Plan`)

- **Focus:** Code one mistake away from a production incident.
- **Look for:** Missing error handling at system boundaries (network, file I/O, DB, external APIs), implicit contracts that break silently, unsafe casts/coercions (`as any`, `unsafe`, unchecked unwrap), cascading failure risk, data integrity gaps, race conditions.
- **Strategy:** Focus on system boundaries. Read error handling paths. Search for `unwrap`, `as any`, bare `except:`, empty catch blocks.

**Optional specialist agents** (available via `/sus add:` or `/sus agents:`):

- **Security Sentinel** (`general-purpose`) — injection vectors, hardcoded secrets, auth bypasses, CSRF/XSS/SSRF
- **Performance Oracle** (`Explore`) — N+1 queries, unbounded data loading, sync blocking in async, missing caching, O(n^2) on large data
- **Concurrency Analyst** (`Plan`) — shared mutable state without sync, deadlocks, TOCTOU, unawaited promises
- **API Contract Reviewer** (`general-purpose`) — breaking change risks, inconsistent endpoints, missing error responses, leaking internals

#### Severity Criteria

Include these criteria in every agent prompt so agents filter correctly.

**CRITICAL** (active danger to production):

- Silent data corruption, lossy conversions on user data
- Concurrency hazards on data affecting correctness
- Security boundary violations, injection vectors, hardcoded secrets
- Cascading failure risk, missing circuit breakers/timeouts, unbounded retries
- Untestable architecture (core logic requires external services to test)

**MAJOR** (significant maintenance/bug risk):

- God classes (5+ responsibilities), shotgun surgery (4+ files), pattern confusion (3+ approaches)
- Complexity walls (CC 15+, nesting 4+), missing boundary error handling
- Implicit contracts, abstraction leaks

**FILTERED OUT** (never report):
Naming preferences, formatting, missing docs, minor DRY (2 occurrences), unused code, TODOs, test code quality, single-use abstractions, language idiom preferences, dependency versions, magic numbers in tests, boilerplate/ceremony.

#### Agent Prompt Template

For each selected agent, construct the `Task` prompt as:

```
You are the {Agent Name} analyzing a codebase at {project_root}.

## Your Focus
{focus from the agent roster above}

## What to Look For
{look-for list from the agent roster above}

## Exploration Strategy
{strategy from the agent roster above}

## Severity Criteria
CRITICAL: Silent data corruption, concurrency hazards, security boundary violations, cascading failure risk, untestable architecture.
MAJOR: God classes (5+ responsibilities), shotgun surgery (4+ files), pattern confusion (3+ approaches), complexity walls (CC 15+, nesting 4+), missing boundary error handling, implicit contracts, abstraction leaks.
NEVER REPORT: Naming preferences, formatting, missing docs, minor DRY (2 occurrences), unused code, TODOs, test code quality, single-use abstractions, language idiom preferences, dependency versions.

## Rules
- Return at most 5 findings, prioritized by severity
- Quality over quantity: only report issues a senior engineer would flag in a design review
- Use the finding format below for each finding
- Include specific file paths and line numbers for every finding
- If you find fewer than 5 issues worth reporting, return fewer. Zero is acceptable.

## Finding Format
### [SEVERITY] Title
- **Category:** {category}
- **Location:** {file_path:line_number} (and related locations)
- **Description:** {what's wrong and why it matters}
- **Impact:** {what breaks, degrades, or becomes unmaintainable}
- **Suggested approach:** {high-level fix direction, not a full implementation}

## Output
Return your findings as a markdown list, or "No findings." if nothing meets the severity bar.
```

### Step 4: Synthesize Findings

Once all agents return:

1. **Deduplicate**: If multiple agents flagged the same location or issue, merge them into a single finding. Note which agents flagged it (this boosts confidence).

2. **Rank**: Sort findings by:
   - Severity (CRITICAL before MAJOR)
   - Cross-agent agreement (findings flagged by 2+ agents rank higher)
   - Impact scope (broader blast radius ranks higher)

3. **Final filter**: Re-check each finding against the severity guide. Remove anything that slipped through that should be filtered out.

4. **Cap at 10**: Keep the top 10 findings maximum. If there are more, drop the lowest-ranked ones.

### Step 5: Present Report

Output the final report in this format:

```markdown
## Sus Report: {project name}

**Agents deployed:** {list of agent names}
**Files explored:** {approximate count from agent outputs}
**Findings:** {count}

---

### 1. [SEVERITY] Title

- **Category:** {category}
- **Location:** `{file_path:line_number}` (and related locations)
- **Flagged by:** {agent name(s)}
- **Description:** {what's wrong and why it matters}
- **Impact:** {what breaks, degrades, or becomes unmaintainable}
- **Suggested approach:** {high-level fix direction}

---

### 2. [SEVERITY] Title

...

---

## What's Next?

Pick an option or tell me what you'd like to do:

1. **Deep-dive** into a specific finding (give me the number)
2. **Start fixing** — I'll tackle findings in priority order
3. **Export** this report to a markdown file
4. **Re-run** with a different focus (e.g., `/sus focus: security`)
```

## Guidelines

- Always run the compilation gate first. No exceptions.
- Deploy agents in parallel, never sequentially.
- Trust the severity guide. When in doubt, filter it out.
- Never report formatting, naming, or style issues.
- Findings must include specific file paths and line numbers.
- Prefer fewer, higher-quality findings over a long list.
- The follow-up menu is important — always include it.
- If the user asks to deep-dive into a finding, use the appropriate agent type to explore that specific issue in depth.
- If the user asks to start fixing, work through findings in priority order, one at a time.
