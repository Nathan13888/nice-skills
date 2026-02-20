---
name: explore
description:
  Strategic discovery of a project's capabilities from a solutions architect
  perspective. Deploys parallel discovery agents to map architecture, inventory
  features, and assess infrastructure, then synthesizes findings into a
  capabilities report with strategic improvement recommendations. Use when user
  says "explore", "what does this do", "project overview", "capabilities",
  "feature inventory", or asks about strategic direction.
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

# Project Explorer

Deploy parallel discovery agents to catalog a project's capabilities and identify strategic improvements and expansion opportunities. Produces a structured report of existing features grouped by domain, plus prioritized recommendations.

`Write` and `Edit` are intentionally excluded from allowed-tools. This skill MUST NOT modify any files.

## Quick Start

Follow all 4 steps sequentially.

## Workflow

### Step 1: Orient

Gather project context to feed into agent prompts. Do NOT present this to the user yet.

1. Read these files (skip any that don't exist):
   - `README.md` or `README.*`
   - `package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml`, `Gemfile`, `pom.xml`, `build.gradle`
   - `docker-compose.yml`, `Dockerfile`
   - `CLAUDE.md`, `.claude/settings.json`
   - Top-level directory listing

2. Extract orientation context:
   - **Project name** — from package manifest or README title
   - **Purpose** — one-sentence description from README or manifest
   - **Stack** — languages, frameworks, key dependencies
   - **Scale** — approximate file count and directory depth
   - **Architecture type** — monolith, monorepo, microservices, library, CLI tool, etc.

3. Count source files (exclude `node_modules`, `vendor`, `target`, `dist`, `.git`, `__pycache__`, `build`):

   ```bash
   find . -type f \( -name '*.ts' -o -name '*.tsx' -o -name '*.js' -o -name '*.jsx' -o -name '*.py' -o -name '*.rb' -o -name '*.go' -o -name '*.rs' -o -name '*.java' -o -name '*.kt' -o -name '*.swift' -o -name '*.c' -o -name '*.cpp' -o -name '*.h' -o -name '*.cs' -o -name '*.ex' -o -name '*.exs' -o -name '*.php' -o -name '*.vue' -o -name '*.svelte' \) -not -path '*/node_modules/*' -not -path '*/vendor/*' -not -path '*/target/*' -not -path '*/dist/*' -not -path '*/.git/*' -not -path '*/__pycache__/*' -not -path '*/build/*' | wc -l
   ```

4. **Small-project escape hatch:** If there are fewer than 10 source files, skip Step 2 (parallel agents) entirely. Instead, read all source files directly, then jump to Step 3 with the information gathered from your own reading. This avoids the overhead of parallel agents for trivial projects.

5. **Parse focus area** (if provided):
   - `/explore focus: API layer` -> set `FOCUS = "API layer"` and include it in all agent prompts
   - `/explore focus: auth` -> set `FOCUS = "auth"` and include it in all agent prompts
   - `/explore` (bare) -> `FOCUS = null`, full-project exploration

### Step 2: Deploy 3 Discovery Agents in Parallel

Launch all 3 agents simultaneously using the `Task` tool. Each agent receives the orientation context from Step 1.

#### Agent Roster

**Architecture Cartographer** (`Plan`)

- **Mission:** Map the project's structural organization — modules, components, data models, integrations, and boundaries.
- **Exploration targets:** Directory structure and module hierarchy, data models and schemas, external integrations (APIs, databases, message queues, third-party services), internal boundaries (what talks to what), configuration and environment management.
- **Strategy:** Start with the directory tree and entry points. Follow imports to map the dependency graph. Identify clear module boundaries and where they blur. Note shared types and cross-cutting concerns.
- **Output format:** Group findings by architectural domain. For each domain, list components with file paths and a one-line description of responsibility.

**Capability Analyst** (`Explore`)

- **Mission:** Inventory all user-facing and developer-facing capabilities — routes, commands, APIs, background jobs, UI screens, event handlers.
- **Exploration targets:** HTTP routes and API endpoints, CLI commands and subcommands, background jobs and scheduled tasks, event handlers and webhooks, UI pages and components (if applicable), exported functions and public APIs (for libraries).
- **Strategy:** Search for route definitions, command registrations, job schedulers, event listeners, and exported interfaces. Read handler implementations to understand what each capability does. For web apps, trace from routes to controllers to services.
- **Output format:** List each capability with: name, type (route/command/job/etc.), file path, and a one-sentence description of what it does.

**Infrastructure Scout** (`general-purpose`)

- **Mission:** Assess the project's operational maturity — CI/CD, testing, deployment, observability, and developer experience tooling.
- **Exploration targets:** CI/CD configuration (GitHub Actions, GitLab CI, Jenkinsfile, etc.), test setup and coverage patterns, deployment configuration (Docker, K8s, serverless, PaaS), observability (logging, metrics, tracing, error tracking), developer experience (linting, formatting, pre-commit hooks, scripts, Makefile), documentation (README quality, API docs, architecture docs).
- **Strategy:** Check for CI config files, test directories, Dockerfiles, deployment manifests, monitoring config. Read package scripts and Makefile targets. Assess documentation completeness.
- **Output format:** For each infrastructure area, report: what exists (with file paths), what's configured, and notable gaps.

#### Agent Prompt Template

For each agent, construct the `Task` prompt as:

```
You are the {Agent Name} exploring a project at {project_root}.

## Orientation
- **Project:** {name} — {purpose}
- **Stack:** {stack}
- **Scale:** ~{file_count} source files
- **Architecture:** {architecture_type}
{if FOCUS: - **Focus area:** Narrow your exploration to aspects related to "{FOCUS}", but still cover your full mission within that lens.}

## Your Mission
{mission from the agent roster above}

## Exploration Targets
{exploration targets from the agent roster above}

## Strategy
{strategy from the agent roster above}

## Rules
- This is a READ-ONLY exploration. Do not modify any files.
- Include specific file paths for every finding.
- Group findings by domain/area.
- Be comprehensive but concise — list capabilities, don't explain obvious ones in detail.
- If a target area has nothing to report, say so briefly rather than omitting it.

## Output Format
{output format from the agent roster above}
```

### Step 3: Synthesize

Once all agents return (or after your own reading for small projects):

1. **Merge and deduplicate** agent reports into a unified domain-grouped view. Domains should emerge naturally from the project (e.g., "Authentication", "API", "Data Layer", "Background Processing") rather than mapping 1:1 to agents.

2. **Generate recommendations** in three categories:

   **Improving existing capabilities** — things that already exist but could be better:
   - Incomplete features (partial implementations, TODO markers, stub endpoints)
   - Missing infrastructure (no tests for critical paths, no CI, no error tracking)
   - Robustness gaps (missing error handling at boundaries, no retry logic, no rate limiting)
   - Tag each: **High** / **Medium** / **Exploratory**

   **Novel capability opportunities** — things that don't exist but naturally extend what does:
   - Natural feature extensions based on existing data models and integrations
   - New integrations that complement the current stack
   - User-facing features enabled by existing infrastructure
   - Tag each: **High** / **Medium** / **Exploratory**

   **Other suggestions** — DX, ecosystem, operational maturity:
   - Developer experience improvements (better scripts, documentation, onboarding)
   - Operational maturity (monitoring, alerting, deployment automation)
   - Ecosystem plays (plugins, extensions, API consumers)
   - Tag each: **High** / **Medium** / **Exploratory**

3. **Impact tagging criteria:**
   - **High** — Addresses a clear gap that affects users or reliability today
   - **Medium** — Meaningful improvement but not urgent; good next-quarter work
   - **Exploratory** — Interesting direction worth investigating; may or may not pan out

### Step 4: Present Report

Output the final report in this format:

```markdown
## Explore: {project name}

**Purpose:** {one-sentence description}
**Stack:** {languages, frameworks, key deps}
**Architecture:** {architecture type}
**Scale:** ~{file_count} source files across {directory_count} directories
{if FOCUS: **Focus:** {FOCUS}}

---

### Capabilities

{For each domain, a table or grouped list:}

#### {Domain Name}

| Capability | Type                     | Location      | Description                |
| ---------- | ------------------------ | ------------- | -------------------------- |
| {name}     | {route/command/job/etc.} | `{file:line}` | {one-sentence description} |

{Repeat for each domain}

---

### Strategic Improvements

Improvements to existing capabilities:

| #   | Recommendation | Impact | Domain   | Details                    |
| --- | -------------- | ------ | -------- | -------------------------- |
| 1   | {title}        | High   | {domain} | {one-sentence explanation} |
| 2   | {title}        | Medium | {domain} | {one-sentence explanation} |

---

### Novel Opportunities

New capabilities this project could grow into:

| #   | Opportunity | Impact      | Builds On                    | Details                    |
| --- | ----------- | ----------- | ---------------------------- | -------------------------- |
| 1   | {title}     | High        | {existing capability/domain} | {one-sentence explanation} |
| 2   | {title}     | Exploratory | {existing capability/domain} | {one-sentence explanation} |

---

### Other Suggestions

DX, operational, and ecosystem improvements:

| #   | Suggestion | Impact | Area   | Details                    |
| --- | ---------- | ------ | ------ | -------------------------- |
| 1   | {title}    | Medium | {area} | {one-sentence explanation} |

---

## What's Next?

Pick an option or tell me what you'd like to do:

1. **Deep-dive** into a specific domain (give me the name)
2. **Elaborate** on a recommendation (give me the number and table)
3. **Compare** two directions — I'll analyze trade-offs
4. **Export** this report to a markdown file
```

## Guidelines

- This skill is strictly read-only. Never modify any files.
- Deploy all 3 agents in parallel, never sequentially (unless small-project path).
- Recommendations should be strategic, not code-level. "Add rate limiting to the API gateway" not "fix the off-by-one on line 42."
- Include file paths in the capabilities table so the user can navigate directly.
- Keep recommendations to 3-5 per category. Quality over quantity.
- The follow-up menu is important — always include it.
- If the user asks to deep-dive into a domain, spawn a targeted agent to explore that domain in depth.
- If the user asks to elaborate on a recommendation, provide a concrete implementation sketch (still read-only — describe the approach, don't write the code).
- If the user asks to compare directions, present a structured trade-off analysis with pros, cons, effort, and risk for each option.
