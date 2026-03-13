---
name: mvp
description:
  Product design, feature planning, and technical architecture for new projects.
  Explores the problem space through deep requirements gathering, suggests
  creative features, makes architecture decisions, and produces a structured MVP
  plan with scope boundaries, a future roadmap, and a deliverable tracker.
  Uses plan mode for deliberate thinking before writing any artifacts.
  Use when the user says "mvp", "plan a product", "design features", "what should I build",
  "feature planning", "scope an MVP", or describes a product they want to plan.
allowed-tools:
  - Bash
  - Write
  - Read
  - Glob
  - Grep
  - Task
  - AskUserQuestion
  - EnterPlanMode
  - ExitPlanMode
---

# MVP Product Planner

Deep requirements gathering, feature ideation, scope management, technical architecture, and living documentation for new products. Produces 5 planning artifacts: OVERVIEW, FEATURES, MVP_SCOPE, ROADMAP, and DELIVERABLES.

## Quick Start

Follow all steps sequentially across 3 phases. Do NOT write any files until the plan is approved in Step 6.

## Workflow

### Phase 1: Understand (Steps 0-3)

---

### Step 0: Enter Plan Mode & Parse Invocation

1. Call `EnterPlanMode` immediately -- no files are written until the plan is approved in Step 6.
2. Parse the invocation:
   - `/mvp` (bare) -- start the full workflow from Step 1
   - `/mvp a fitness tracker` -- seed description provided, use it as the answer to Step 1's seed question and skip that question
   - `/mvp --resume` -- Glob for existing docs in `./docs/mvp/`, `./plans/`, `./mvp/`. If found, read them and resume from the appropriate step based on what's already documented. If not found, start fresh from Step 1.

### Step 1: Problem Space Exploration

Deep requirements gathering via `AskUserQuestion` with concrete options. Ask ALL of these questions, adapting options based on prior answers:

1. **Seed question** (skip if a description was provided in the invocation):

   > What are you building? Describe the problem or product in a few sentences.

2. **Target users** (select one):
   - Yourself / Developers / Internal team / End consumers (B2C) / Businesses (B2B)

3. **Problem severity** (select one):
   - Hair on fire / Annoying / Nice to have / New behavior

4. **Existing solutions** (multiSelect):
   - Nothing / Manual processes / Competitors / Internal tools / Open source

5. **Scale expectations** (select one):
   - Just me / Small group (<100) / Medium (hundreds-thousands) / Large (10k+) / Unknown

After gathering all answers, produce a **Problem Brief** and confirm it with the user via `AskUserQuestion`:

```markdown
## Problem Brief

**Product:** {one-line description}
**Target users:** {audience}
**Core problem:** {the pain being solved}
**Current alternatives:** {what exists today}
**Scale:** {expected scale}
**Key insight:** {what makes this solution different/better}
```

Ask: "Does this Problem Brief capture your vision correctly? (Yes / Adjust -- tell me what to change)"

### Step 2: Requirements Deep-Dive

Targeted questions based on Step 1 answers. Use `AskUserQuestion` for each:

1. **Core workflows** -- Ask:

   > What is the SINGLE most important thing a user does?

   Then ask clarifying sub-questions based on the answer:
   - Does this require authentication?
   - Is it real-time or async?
   - What data does it involve (structured, unstructured, files)?

   Repeat for up to 2 more workflows: "What's the SECOND most important workflow?" and "Any third critical workflow?"

2. **Technical constraints** (multiSelect):

   > Which of these apply to your product?

   Options: Auth required / Real-time updates / Offline support / Mobile-first / API integrations / Data persistence / Multi-tenancy / Privacy compliance (GDPR etc.) / None of these

3. **Anti-goals** (multiSelect):

   > What should we explicitly EXCLUDE from the MVP? These are common features that add scope without proving the core idea.

   Options: Admin dashboard / Analytics / Payments / Notifications / Mobile app / User customization / Social features / Search / Internationalization

### Step 3: Technical Architecture

Architecture questions tailored to the product type. Use `AskUserQuestion` for each:

1. **Application type** (select one):
   - Web app / CLI tool / API service / Mobile app / Desktop app / Library/SDK

2. **Framework** (options based on app type):

   For **Web app**: Next.js / SvelteKit / Remix / Plain React + Vite / Astro / Other
   For **CLI tool**: Node.js (Commander) / Python (Click/Typer) / Rust (Clap) / Go (Cobra) / Other
   For **API service**: Express / Fastify / FastAPI / Actix / Go net/http / Other
   For **Mobile app**: React Native / Flutter / Swift / Kotlin / Other
   For **Desktop app**: Electron / Tauri / Other
   For **Library/SDK**: TypeScript / Python / Rust / Go / Other

   Mark the best fit as "(Recommended)" based on requirements gathered in Steps 1-2. Explain why in one sentence.

3. **Database** (select one):
   - PostgreSQL / SQLite / MongoDB / Redis / None (stateless) / Other
   - Recommend based on scale expectations and data model needs from Step 2.

4. **Deployment** (select one):
   - Cloudflare Workers / Vercel / AWS / Self-hosted / Local only / Other

5. **Architecture pattern** (select one):
   - Monolith (Recommended for MVP) / Microservices / Serverless functions / Hybrid

After gathering, add a **Tech Stack** section and confirm with the user:

```markdown
## Tech Stack

**Type:** {app type}
**Framework:** {framework}
**Database:** {database}
**Deployment:** {platform}
**Architecture:** {pattern}
**Key rationale:** {why these choices fit the requirements}
```

Ask: "Does this tech stack look right? (Yes / Adjust -- tell me what to change)"

---

### Phase 2: Design (Steps 4-6)

---

### Step 4: Feature Ideation (Agent-Powered)

Deploy 2 `Task` agents in parallel (both with `Plan` subagent_type) with the full context from Steps 1-3.

Construct the prompts from the templates in `AGENT_PROMPTS.md` (located in this skill's directory). Read that file to get the prompt templates.

**Agent 1: Product Architect**

- Proposes a structured feature set organized by user workflow
- For each feature: name, description, complexity (S/M/L), dependencies, MVP-essential vs enhancement
- Considers how the tech stack choices enable or constrain features

**Agent 2: Domain Explorer**

- Thinks from the user's perspective
- Suggests features the user hasn't considered: onboarding UX, error states as features, data lifecycle (import/export/backup), sharing/collaboration, accessibility, admin needs
- Outputs "likely needed" vs "nice surprise" categories

After both agents return, synthesize their outputs into a **unified, deduplicated feature list**. Merge overlapping suggestions, preserve the best description, and track the source of each feature.

### Step 5: Feature Presentation & Prioritization

Present the unified feature table to the user:

```markdown
## Proposed Features

| #   | Feature    | Description          | Complexity | Source            | Suggested Tier |
| --- | ---------- | -------------------- | ---------- | ----------------- | -------------- |
| 1   | User auth  | Email/password login | M          | User              | MVP            |
| 2   | Dashboard  | Main overview screen | M          | Product Architect | MVP            |
| 3   | CSV export | Download data        | S          | Domain Explorer   | Future         |
```

**Source** column values:

- `User` -- explicitly requested by the user
- `Inferred` -- derived from requirements (e.g., auth implied by multi-user)
- `Suggested` -- proposed by an agent

Then use `AskUserQuestion` twice:

1. (multiSelect): "Which features MUST be in your MVP? (select all that apply)"
   - List all features as options

2. "Any features to ADD that are missing, or REMOVE entirely? (Add / Remove / Looks good)"
   - If Add: ask what to add, then incorporate and re-present
   - If Remove: ask which to remove

### Step 6: Scope Definition & Approval

Organize features into two sections:

**MVP Scope** -- ordered by implementation sequence respecting dependencies:

```markdown
### MVP Scope (v1)

| Order | Feature     | Depends On  | Complexity | Notes                     |
| ----- | ----------- | ----------- | ---------- | ------------------------- |
| 1     | Data models | --          | S          | Foundation                |
| 2     | User auth   | Data models | M          | Required by most features |
```

**Future Roadmap** -- grouped into release waves:

```markdown
### Future Roadmap

#### v1.1 -- Quick Wins

| Feature | Complexity | Why deferred |

#### v1.2 -- Enhanced Experience

| Feature | Complexity | Why deferred |

#### Backlog -- Investigate Later

| Feature | Complexity | Why deferred |
```

Present both sections to the user, then use `AskUserQuestion` as a **hard approval gate**:

> Ready to finalize this plan? Pick an option:
>
> 1. **Approve** -- proceed to documentation (Recommended)
> 2. **Adjust scope** -- move features between MVP and future
> 3. **Adjust order** -- resequence the implementation plan
> 4. **Deep-dive** -- expand a specific feature before approving

Handle each option:

- **Approve**: Call `ExitPlanMode` to unlock file writes, proceed to Step 7.
- **Adjust scope**: Ask which features to move and in which direction, update the tables, re-present, and ask for approval again.
- **Adjust order**: Ask what to resequence, update, re-present, and ask again.
- **Deep-dive**: Expand the specified feature with more detail (user stories, edge cases, technical considerations), then ask for approval again.

Loop until the user approves.

On approval, call `ExitPlanMode` to unlock file writes.

---

### Phase 3: Document (Steps 7-8)

---

### Step 7: Documentation Output

First, ask where to save using `AskUserQuestion`:

> Where should I save the MVP planning documents?
>
> 1. `./docs/mvp/` (Recommended)
> 2. `./plans/`
> 3. `./mvp/`
> 4. Custom path

Create the chosen directory if it doesn't exist.

Produce 5 artifacts:

#### 1. `OVERVIEW.md` -- Product Vision & Tech Stack

```markdown
# {Product Name}

## Vision

{2-3 sentence product vision}

## Problem Brief

**Product:** {one-line description}
**Target users:** {audience}
**Core problem:** {the pain being solved}
**Current alternatives:** {what exists today}
**Scale:** {expected scale}
**Key insight:** {what makes this solution different/better}

## Tech Stack

**Type:** {app type}
**Framework:** {framework}
**Database:** {database}
**Deployment:** {platform}
**Architecture:** {pattern}
**Key rationale:** {why these choices fit}

## Product Principles

1. {principle 1}
2. {principle 2}
3. {principle 3}

## Competitive Landscape

| Alternative | Strengths | Weaknesses | Our Differentiator |
| ----------- | --------- | ---------- | ------------------ |
```

#### 2. `FEATURES.md` -- Complete Feature Catalog

Every feature (MVP + future) with user stories, acceptance criteria, complexity, dependencies, and tier.

```markdown
# Feature Catalog

## MVP Features

### {Feature Name}

- **Tier:** MVP
- **Complexity:** {S/M/L}
- **Dependencies:** {list or none}
- **Source:** {User/Inferred/Suggested}

**User Story:** As a {user type}, I want to {action} so that {benefit}.

**Acceptance Criteria:**

- [ ] {criterion 1}
- [ ] {criterion 2}
- [ ] {criterion 3}

---

## Future Features

### {Feature Name}

...
```

#### 3. `MVP_SCOPE.md` -- Ordered Implementation Plan

```markdown
# MVP Implementation Plan

## Implementation Sequence

| Order | Feature | Depends On | Complexity | Notes |
| ----- | ------- | ---------- | ---------- | ----- |
| 1     | ...     | --         | S          | ...   |

## Architecture Decisions

- {Decision 1 and rationale}
- {Decision 2 and rationale}

## Explicitly Out of Scope

These items are intentionally excluded from the MVP:

- {item 1} -- {reason}
- {item 2} -- {reason}

## Mini-Specs

### {Feature 1}

{Brief technical approach -- 3-5 bullets on how to implement}

### {Feature 2}

...
```

#### 4. `ROADMAP.md` -- Future Release Waves

```markdown
# Product Roadmap

## v1.1 -- Quick Wins

| Feature | Complexity | Why Deferred | Trigger to Reconsider |
| ------- | ---------- | ------------ | --------------------- |

## v1.2 -- Enhanced Experience

| Feature | Complexity | Why Deferred | Trigger to Reconsider |
| ------- | ---------- | ------------ | --------------------- |

## Backlog -- Investigate Later

| Feature | Complexity | Why Deferred | Trigger to Reconsider |
| ------- | ---------- | ------------ | --------------------- |

## Open Questions

- {question 1}
- {question 2}
```

#### 5. `DELIVERABLES.md` -- Living Progress Tracker

Nested markdown checklists with complexity tags inline:

```markdown
# Deliverables Tracker

## MVP (v1)

- [ ] **Data models** (S) -- Define core schemas
  - [ ] User model
  - [ ] {Domain model 1}
  - [ ] {Domain model 2}
- [ ] **User auth** (M) -- Email/password authentication
  - [ ] Registration flow
  - [ ] Login flow
  - [ ] Session management

## Future

- [ ] **v1.1: {Feature}** (S)
- [ ] **v1.1: {Feature}** (M)
- [ ] **v1.2: {Feature}** (L)
- [ ] **Backlog: {Feature}** (L)
```

### Step 8: Summary & Handoff

Print a summary of what was created:

```markdown
## MVP Plan: {product name}

**Product:** {one-line description}
**Tech stack:** {framework} + {database} on {deployment}
**MVP features:** {count} features ({S count} small, {M count} medium, {L count} large)
**Future features:** {count} features across {wave count} release waves
**Documentation:** {path}

### Files Created

| File                   | Purpose                              |
| ---------------------- | ------------------------------------ |
| {path}/OVERVIEW.md     | Product vision and tech stack        |
| {path}/FEATURES.md     | Complete feature catalog with specs  |
| {path}/MVP_SCOPE.md    | MVP implementation plan and sequence |
| {path}/ROADMAP.md      | Future release roadmap               |
| {path}/DELIVERABLES.md | Implementation checklist tracker     |

---

## What's Next?

1. **Deep-dive** into a specific feature -- I'll expand the spec
2. **Run /init-repo** -- scaffold the project infrastructure
3. **Start building** -- pick the first MVP feature and begin
4. **Refine scope** -- revisit prioritization or add features
5. **Export** -- create GitHub Issues, Linear tasks, etc. from the plan
```

## Guidelines

- Call `EnterPlanMode` at the very start -- no file writes until scope is approved in Step 6
- Call `ExitPlanMode` only after the user explicitly approves the plan
- Ask before assuming -- use `AskUserQuestion` with concrete options everywhere, not open-ended prompts
- Features from agents must be clearly marked as `Suggested` in the Source column
- Think step-by-step about dependencies and edge cases before presenting any feature list
- Resist scope creep -- actively push back when the user tries to put everything in MVP. A good MVP is small and focused.
- Balanced suggestions -- practical core features with some creative ideas; ambitious ones clearly flagged so the user can easily defer them
- Markdown checklists for deliverable tracking -- no external tools needed
- Never write code -- this skill produces planning documents only
- Suggest `/init-repo` as the natural next step for scaffolding the project infrastructure
- If the user asks to deep-dive into a feature, expand with user stories, edge cases, technical approach, and acceptance criteria
- The "What's Next?" menu is important -- always include it at the end
