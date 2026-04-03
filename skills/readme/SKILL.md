---
name: readme
description: >
  Generate a high-quality README.md for an existing repository. Use this skill whenever a user
  asks to write, create, generate, draft, or improve a README for a codebase, project, or repo.
  Also trigger when the user says things like "document this project", "add a README",
  "write project docs", or "make this repo presentable". Works for any language or repo shape --
  libraries, CLIs, applications, APIs, monorepos, frameworks, and plugins.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# README Writer

Write clear, concise, well-targeted READMEs for open-source projects. The goal is a README
that helps a stranger understand what the project is, try it in under 2 minutes, and know
where to go next — nothing more.

## Philosophy

A good open-source README is **short enough to read and long enough to be useful**. Avoid
walls of text, badge spam, and sections that repeat what the code already says. Every section
must earn its place by answering a question a newcomer actually has.

The three questions every README must answer fast:
1. **What is this?** — one or two sentences.
2. **How do I use it?** — install + minimal working example.
3. **Where do I go from here?** — links to docs, issues, contributing guide.

---

## Step 1 — Analyze the repo

Before writing anything, scan the repo to classify it. Run through this checklist.

### Detect language & ecosystem

Look for these marker files (check in order, stop on first match per ecosystem):

| Marker file              | Ecosystem              | Package manager hint                            |
|--------------------------|------------------------|-------------------------------------------------|
| `pyproject.toml`         | Python                 | check `[build-system]` for pip/poetry/hatch/uv  |
| `setup.py` / `setup.cfg` | Python                 | pip / setuptools                                |
| `package.json`           | JavaScript/TypeScript  | check for yarn.lock, pnpm-lock.yaml, bun.lockb  |
| `Cargo.toml`             | Rust                   | cargo                                           |
| `go.mod`                 | Go                     | go modules                                      |
| `Gemfile` / `*.gemspec`  | Ruby                   | bundler / gem                                   |
| `build.gradle` / `pom.xml` | Java / Kotlin        | gradle / maven                                  |
| `*.sln` / `*.csproj`     | C# / .NET              | dotnet / nuget                                  |
| `mix.exs`                | Elixir                 | mix / hex                                       |
| `CMakeLists.txt` / `Makefile` | C / C++           | cmake / make                                    |
| `Package.swift`          | Swift                  | swift package manager                           |

If multiple ecosystems are present, note all of them — the repo may be a monorepo.

### Detect repo type

Use these heuristics:

- **Library / Package** — publishes to a registry (version field, `[lib]` in Cargo.toml,
  `main`/`exports` in package.json pointing to lib code, `py.typed` marker). No Dockerfile,
  no deployment config.
- **CLI tool** — has a `bin` field, `[[bin]]` in Cargo.toml, `console_scripts` entry point,
  or a prominent `main()` with arg parsing.
- **Web application** — has a framework entry point (Next.js/Rails/Django/Flask/Phoenix),
  Dockerfile, or deployment configs (Vercel/Fly/Heroku/K8s manifests).
- **API / Service** — similar to web app but serves data, not pages. Look for route
  definitions, OpenAPI specs, GraphQL schemas.
- **Monorepo** — has `workspaces` in package.json, a `packages/`, `crates/`, or `apps/`
  directory, or a workspace Cargo.toml. Contains multiple semi-independent sub-projects.
- **Framework / Plugin** — designed to be extended by others. Exports hooks, middleware, or
  plugin interfaces. Often has an `examples/` directory.

If ambiguous, default to the simpler classification. A CLI that's also a library → document
as a library with a CLI section.

### Extract key metadata

Pull these from config files and code — don't ask the user for things the repo already
tells you:

- Project name (from config `name` field, or repo directory name)
- Description / tagline (from config `description`, or infer from code)
- License (from LICENSE file or config field)
- Minimum language/runtime version (from config or CI files)
- Entry points and exports
- Dependencies (just the notable ones, not the full tree)
- Existing docs (`docs/`, wiki link, `mkdocs.yml`, `docusaurus.config.js`)
- CI setup (GitHub Actions, etc.)
- Whether CONTRIBUTING.md, CODE_OF_CONDUCT.md, or CHANGELOG.md exist
- Existing README.md — if one exists, read it and preserve any project-specific information
  (maintainer-chosen badges, custom sections) unless the user asks for a fresh start

---

## Step 2 — Assemble the README

Refer to the **Appendix: Type-Specific Templates** section below for the repo type you
detected, then refer to **Appendix: Language & Ecosystem Conventions** for the correct
install commands and code patterns.

### Universal structure (all repo types share this skeleton)

```markdown
# Project Name

One-line description of what this does and who it's for.

## Installation

[ecosystem-appropriate install command]

## Quick start

[Minimal working example — 5-15 lines of code or 1-3 shell commands]

## [Type-specific sections — see appendix]

## Contributing

[Short paragraph or link to CONTRIBUTING.md]

## License

[License name, linked to LICENSE file]
```

### Rules for every section

- **Title**: Use the project name as-is. Don't add taglines to the H1.
- **Description**: One to three sentences max. State what it does, what problem it solves,
  and (if not obvious) what language/ecosystem it targets. Skip marketing language.
- **Installation**: Show the one canonical install command. If there are multiple valid
  methods, show the primary one first and put alternatives in a `<details>` block.
- **Quick start**: The single most common use case, runnable as-is. Prefer a code block
  that someone can copy-paste. If it's a library, show an import + call. If it's a CLI,
  show a command + expected output.
- **Contributing**: If a CONTRIBUTING.md exists, link to it with one sentence. If not,
  write 2-3 sentences covering: how to run tests, how to submit a PR, and where to ask
  questions.
- **License**: One line. e.g., `MIT — see [LICENSE](LICENSE)`.

### Things to deliberately leave out

- Badges beyond build status (no "downloads", "code style", "chat" badges unless the
  project already uses them)
- Detailed API documentation (belongs in generated docs or a docs site)
- Full configuration reference (link to a config doc instead)
- Changelog (link to CHANGELOG.md)
- Long feature lists — pick the 3-5 most important, link to docs for the rest
- Screenshots unless the project is visual (UI library, app, CLI with rich output)

---

## Step 3 — Write it

Write the README in a single Markdown file. Follow these formatting rules:

- Use ATX headings (`#`, `##`, `###`). Don't go deeper than `###`.
- Use fenced code blocks with language tags (` ```python `, ` ```bash `, etc.)
- Keep lines under ~100 characters where practical for diff readability.
- Use reference-style links at the bottom for URLs that appear multiple times.
- Prefer one blank line between sections, not two.
- Don't wrap prose in HTML tags unless necessary (e.g., `<details>` for collapsible
  sections).

### Tone

Write like a knowledgeable colleague explaining the project at a whiteboard — direct,
specific, no filler. Avoid:
- "Simply", "just", "easily" — condescending if the thing isn't actually simple.
- "Powerful", "blazing fast", "modern" — show, don't tell.
- "This project aims to..." — it either does or it doesn't.

### Length targets

| Repo type   | Target length |
|-------------|---------------|
| Library     | 80–150 lines  |
| CLI         | 80–150 lines  |
| Application | 100–200 lines |
| Monorepo    | 100–200 lines |
| Framework   | 120–200 lines |

These are guidelines, not hard limits. A tiny utility might need 40 lines. A complex
framework might need 250. But if you're past 200 lines, look for things to cut or move
to separate docs.

---

## Step 4 — Review against checklist

Before delivering, verify:

- [ ] A newcomer can understand what the project does from the first 3 lines
- [ ] The install command is correct and complete (includes any prerequisite steps)
- [ ] The quick start example actually works (uses real exports/functions from the codebase)
- [ ] No placeholder text or TODOs remain
- [ ] Links to files (LICENSE, CONTRIBUTING.md, docs) point to real files that exist
- [ ] Code examples use the project's actual API, not made-up function names
- [ ] The README doesn't duplicate information available in existing docs — it links instead
- [ ] Length is within target range for the repo type

---

## Guidelines

- Always explore the repo before writing. Do not ask the user for information the codebase
  already provides (project name, language, license, dependencies).
- If the repo type is ambiguous, default to the simpler classification.
- Code examples must use real exports, function names, and import paths from the codebase.
  Never invent API names.
- If an existing README.md is present, read it first. Preserve project-specific content
  (maintainer-chosen badges, custom sections) unless the user explicitly asks for a fresh
  start.
- Do not add badges beyond build status unless the project already uses them or the user
  requests them.
- Keep the README within the length target for the repo type. If the output exceeds 200
  lines, look for content to cut or move to separate docs.
- When the project has no existing CONTRIBUTING.md, write 2-3 sentences covering how to
  run tests, submit a PR, and ask questions. Do not generate a full contributing guide.
- For monorepos, generate only the root README. Do not generate sub-package READMEs unless
  explicitly asked.
- If the project is not yet published to a package registry, show install-from-source only.

---

## Appendix: Type-Specific Templates

### Library / Package

Libraries exist to be imported by other people's code. The README's job is to get someone
from "what is this" to "working in my project" as fast as possible.

**Sections (in order):**

**Features** (optional) — Only include if the library does multiple distinct things and
the one-line description can't capture them. Use a short bullet list — 3 to 5 items, each
one sentence. Don't list things that are table stakes (e.g., "well-tested" or "typed" for
a TypeScript library).

**Installation** — Show the canonical install command. If the library has optional extras
or feature flags, show the base install first and mention extras below.

Pattern:
```markdown
## Installation

```bash
pip install my-lib
```

For async support:
```bash
pip install my-lib[async]
```
```

**Quick start** — Show the single most common use case. A self-contained code block that
someone can paste and run. Include the import, one function call, and the expected output
as a comment.

Pattern:
```markdown
## Quick start

```python
from my_lib import process

result = process("input data")
print(result)  # → "processed: input data"
```
```

**Usage** (optional) — If the library has 2-3 major use cases beyond the quick start,
show them here. Each gets a `###` subsection with a short code example. Keep each example
under 10 lines.

**API overview** (optional, brief) — Only for small libraries with a small surface area
(<10 public functions/classes). List main exports in a table:

```markdown
| Function | Description |
|----------|-------------|
| `process(input)` | Transforms input data |
| `validate(schema, data)` | Checks data against schema |
```

For larger libraries, link to generated API docs instead.

**Compatibility** — State the minimum language/runtime version. Mention any platform
restrictions.

Pattern:
```markdown
## Compatibility

Requires Python 3.9+. Tested on CPython and PyPy.
```

**Things to skip for libraries:** deployment instructions, Docker setup, environment
variables (usually), configuration files (unless the library reads one).

---

### CLI Tool

CLI tools are used from a terminal. The README should show installation, the 2-3 most
useful commands, and what the output looks like.

**Sections (in order):**

**Installation** — Show the preferred install method first, then alternatives in a
`<details>` block.

Pattern:
```markdown
## Installation

```bash
brew install my-tool
```

<details>
<summary>Other installation methods</summary>

**From crates.io:**
```bash
cargo install my-tool
```

**From source:**
```bash
git clone https://github.com/user/my-tool && cd my-tool
make install
```
</details>
```

**Quick start** — Show 1-3 commands that demonstrate the core use case. Include realistic
(but concise) output so people know what to expect. Use `$` prefix on command lines.

Pattern:
```markdown
## Quick start

```bash
$ my-tool analyze src/
✓ 42 files scanned
✓ 3 issues found

  src/auth.py:12    unused import 'os'
  src/db.py:45      connection not closed
```
```

**Usage** — Show the 3-5 most important subcommands or flags in a table. Don't reproduce
`--help` output — summarize the most useful ones.

Pattern:
```markdown
## Usage

| Command | Description |
|---------|-------------|
| `my-tool analyze <dir>` | Scan files for issues |
| `my-tool fix <dir>` | Auto-fix what it can |
| `my-tool init` | Create a config file |

Run `my-tool --help` for the full command reference.
```

**Configuration** (if applicable) — If the tool reads a config file, show a minimal
example with the most common options. Link to a reference doc for the rest.

**Things to skip for CLIs:** library/import usage (unless it's also a library), API
reference tables, detailed flag descriptions (that's what `--help` is for).

---

### Application / API Service

Applications and services are things people run, not import. The README needs to get
someone from clone to running locally, and point them to deployment docs.

**Sections (in order):**

**Prerequisites** — List what needs to be installed before setup, with specific versions.
Only list things that aren't obvious.

Pattern:
```markdown
## Prerequisites

- Node.js 20+
- PostgreSQL 15+
- Redis 7+ (for job queue)
```

**Setup** — Step-by-step from clone to running. Number the steps. Keep to the minimum
viable path. If there's a Docker path that's simpler, show that first and put the manual
setup in a `<details>` block.

Pattern:
```markdown
## Setup

1. Clone and install dependencies:
   ```bash
   git clone https://github.com/user/my-app && cd my-app
   npm install
   ```

2. Set up the database:
   ```bash
   cp .env.example .env      # then edit .env with your DB credentials
   npm run db:migrate
   ```

3. Start the dev server:
   ```bash
   npm run dev
   ```

   Open http://localhost:3000.
```

**Configuration** — Table of required env vars. Don't list every optional var — just the
ones needed to run. Link to `.env.example` for the full set.

Pattern:
```markdown
## Configuration

| Variable | Description | Required |
|----------|-------------|----------|
| `DATABASE_URL` | PostgreSQL connection string | Yes |
| `SECRET_KEY` | Session signing key | Yes |
| `PORT` | Server port (default: 3000) | No |

Copy `.env.example` to `.env` and fill in the values.
```

**Architecture** (optional, brief) — Only if the project has a non-obvious structure.
2-3 sentences describing major components and how they connect. Link to a full doc if
one exists.

**Deployment** — Link only. Don't put full deployment instructions in the README.

```markdown
## Deployment

See [docs/deployment.md](docs/deployment.md) for production setup.
```

**Development** — Brief notes on running tests and the dev workflow.

```markdown
## Development

```bash
npm test           # run tests
npm run lint       # check code style
npm run db:seed    # load sample data
```
```

**Things to skip for applications:** API endpoint documentation, detailed database schema,
full infrastructure diagrams, production configuration details.

---

### Monorepo

A monorepo README is a navigation hub. Its job is to explain the shape of the repo and
route people to the right sub-project. It should NOT try to document every package.

**Sections (in order):**

**Overview** — What is this collection of things? One paragraph explaining the project
or organization and what the packages do together.

**Packages / Projects table** — The most important section. A table listing every
significant package with a one-line description and a link to its own README.

Pattern:
```markdown
## Packages

| Package | Description |
|---------|-------------|
| [`@scope/core`](packages/core) | Core runtime library |
| [`@scope/cli`](packages/cli) | Command-line interface |
| [`app`](apps/web) | Documentation website |
```

Only add a version column if the packages are published to a registry (use version badges
then). If the monorepo has distinct categories (e.g., `apps/` and `packages/`), group
them under separate `###` subheadings.

**Getting started** — How to set up the whole workspace — clone, install, and run the
most common dev task. Mention the workspace tool (pnpm workspaces, Turborepo, Nx, Cargo
workspaces, etc.).

Pattern:
```markdown
## Getting started

```bash
git clone https://github.com/user/my-mono && cd my-mono
pnpm install        # install all workspace dependencies
pnpm build          # build all packages
pnpm dev            # start dev mode for all packages
```
```

**Common tasks** — 3-5 workspace-level commands people will run often.

**Adding a new package** (optional) — If there's a convention or generator script,
briefly explain it. If there's a command, just show the command.

**Things to skip for monorepos:** detailed docs for individual packages, full dependency
graphs, release process details, CI/CD pipeline descriptions.

---

### Framework / Plugin

Frameworks and plugins are designed to be extended by others. The README needs to show
how to install, how to use the basic API, and how to build on top of it.

**Sections (in order):**

**Features** — More important for frameworks than libraries — users are choosing an
opinionated tool. List 3-5 defining features that differentiate this framework. Focus
on what it enables, not implementation details.

Pattern:
```markdown
## Features

- **File-based routing** — directory structure defines your routes automatically
- **Hot reload** — changes reflect instantly without restarting the server
- **Built-in auth** — session management and OAuth out of the box
```

**Installation** — If the framework has a scaffolding CLI, show that. Otherwise, show
the manual install command.

Pattern (with scaffolding CLI):
```markdown
## Installation

```bash
npx create-my-framework my-app
cd my-app
npm run dev
```
```

**Quick start** — Show the minimal path from install to something running. For
frameworks, this is usually creating one file and starting a dev server.

**Core concepts** — Explain 2-3 concepts a newcomer must understand before they can
be productive. Each gets a `###` subsection with a short paragraph and a code example.
Keep each to one paragraph + one short code example. Link to docs for anything more.

Pattern:
```markdown
## Core concepts

### Routing

Routes map to files in `src/routes/`. A file named `src/routes/users/[id].js` matches
`/users/:id`:

```js
export function GET({ params }) {
  return Response.json({ userId: params.id });
}
```
```

**Plugin API** (if applicable) — If the framework supports plugins or extensions, show
how to create a minimal one. Skip this section if there's no extension mechanism.

**Examples** — Link to the `examples/` directory if one exists, with a brief table of
what's there. Omit if no examples directory exists.

**Things to skip for frameworks:** full configuration reference, migration guides,
comparison tables with competitors, internal architecture details.

---

## Appendix: Language & Ecosystem Conventions

When writing install commands, code examples, and project setup instructions, follow the
conventions of the ecosystem. People expect familiar patterns.

### Python

**Install commands** (pick based on what the project actually uses):
```bash
pip install my-package              # pip (most universal)
uv add my-package                   # uv
poetry add my-package               # poetry
pdm add my-package                  # pdm
```

If the project uses uv, poetry, or pdm internally, show that first. Always mention
`pip install` as the universal fallback unless the package isn't on PyPI.

**Code examples:**
- Always show the import first
- Use f-strings (not `.format()` or `%`)
- Type hints are nice if the library uses them, but don't add them if the library doesn't
- Show `if __name__ == "__main__":` only if the example is meant to be run as a script

**Version/compatibility:** Check `python_requires` in pyproject.toml or setup.cfg.

**Dev setup pattern:**
```bash
git clone ... && cd ...
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
```

If the project uses uv, prefer `uv sync` over manual venv creation.

---

### JavaScript / TypeScript

**Install commands** (detect from lockfile):
```bash
npm install my-package              # npm (package-lock.json)
yarn add my-package                 # yarn (yarn.lock)
pnpm add my-package                 # pnpm (pnpm-lock.yaml)
bun add my-package                  # bun (bun.lockb)
```

Show the one that matches the project's lockfile. For library READMEs consumed by others,
show `npm install` as the default.

**Code examples:**
- Use ESM (`import`) by default unless the project specifically targets CommonJS
- For TypeScript projects, show TypeScript examples (not JS)
- Include the `from` path — people need to know the exact import path
- For React libraries, show a JSX example

**Version/compatibility:** Check `engines` in package.json for minimum Node.js version.

**Dev setup pattern:**
```bash
git clone ... && cd ...
npm install
npm run dev          # or npm test
```

---

### Rust

**Install commands:**
```bash
cargo install my-tool               # CLI tools (installs binary)
cargo add my-crate                  # libraries (adds to Cargo.toml)
```

For libraries, show `cargo add`. For CLI tools, show `cargo install`. If the tool is also
available via homebrew or other package managers, show those as alternatives.

**Code examples:**
- Show `use` statements
- Include `fn main()` wrapper for runnable examples
- Add brief comments for non-obvious Rust patterns (lifetimes, trait bounds)
- Use `?` for error handling in examples

**Version/compatibility:** Check `rust-version` in Cargo.toml for MSRV.

**Dev setup pattern:**
```bash
git clone ... && cd ...
cargo build
cargo test
```

---

### Go

**Install commands:**
```bash
go install github.com/user/tool@latest    # CLI tools
go get github.com/user/lib                # libraries
```

**Code examples:**
- Show the full import path
- Include `package main` and `func main()` for runnable examples
- Handle errors explicitly (don't use `_` for error returns in examples)

**Version/compatibility:** Check the `go` directive in go.mod.

**Dev setup pattern:**
```bash
git clone ... && cd ...
go build ./...
go test ./...
```

---

### Ruby

**Install commands:**
```bash
gem install my-gem                  # standalone
# or in Gemfile:
gem 'my-gem', '~> 1.0'
bundle install
```

**Code examples:**
- Show `require` first
- Use modern Ruby syntax (keyword args, pattern matching if relevant)

**Dev setup pattern:**
```bash
git clone ... && cd ...
bundle install
bundle exec rake test
```

---

### Java / Kotlin

**Install commands** — show the dependency snippet for the project's build tool:

For Gradle:
```groovy
implementation 'com.example:my-lib:1.0.0'
```

For Maven:
```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>my-lib</artifactId>
  <version>1.0.0</version>
</dependency>
```

**Dev setup pattern:**
```bash
git clone ... && cd ...
./gradlew build      # or mvn package
./gradlew test       # or mvn test
```

---

### C / C++

Install patterns vary. Common approaches:

```bash
# CMake
cmake -B build && cmake --build build
sudo cmake --install build

# Make
make && sudo make install

# vcpkg / Conan
vcpkg install my-lib
conan install my-lib
```

Always mention build prerequisites (compiler version, cmake version, system libraries).

---

### General patterns (all languages)

- Show one install method prominently, put alternatives in `<details>` blocks
- Use the project's actual package/crate/gem name, not a placeholder
- If the project isn't published to a registry, show install-from-source only
- Match the code style of the project in examples (if the project uses tabs, use tabs)
- If a project uses Docker as the primary dev experience, show that as the main setup path
