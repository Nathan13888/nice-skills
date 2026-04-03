# Monorepo README

A monorepo README is a navigation hub. Its job is to explain the shape of the repo and
route people to the right sub-project. It should NOT try to document every package.

## Sections (in order)

### Overview
What is this collection of things? One paragraph explaining the project or organization
and what the packages do together.

### Packages / Projects table
The most important section. A table listing every significant package with a one-line
description and a link to its own README.

**Pattern:**
```markdown
## Packages

| Package | Description | Version |
|---------|-------------|---------|
| [`@scope/core`](packages/core) | Core runtime library | ![npm](https://img.shields.io/npm/v/@scope/core) |
| [`@scope/cli`](packages/cli) | Command-line interface | ![npm](https://img.shields.io/npm/v/@scope/cli) |
| [`@scope/plugin-react`](packages/plugin-react) | React integration | ![npm](https://img.shields.io/npm/v/@scope/plugin-react) |
| [`app`](apps/web) | Documentation website | — |
```

Only use version badges in this table if the packages are published to a registry.
Otherwise, drop the version column.

If the monorepo has distinct "categories" (e.g., `apps/` and `packages/`), group them
under separate `###` subheadings within this section.

### Getting started
Show how to set up the whole workspace — clone, install, and run the most common dev task.

**Pattern:**
```markdown
## Getting started

```bash
git clone https://github.com/user/my-mono && cd my-mono
pnpm install        # install all workspace dependencies
pnpm build          # build all packages
pnpm dev            # start dev mode for all packages
```
```

Mention the workspace tool (pnpm workspaces, Turborepo, Nx, Lerna, Cargo workspaces, etc.)
so people understand the build orchestration.

### Common tasks
A short section covering 3-5 workspace-level commands people will run often.

```markdown
## Common tasks

```bash
pnpm test           # test all packages
pnpm lint           # lint everything
pnpm build:core     # build just the core package
pnpm changeset      # create a changeset for versioning
```
```

### Adding a new package (optional)
If the monorepo has a convention for new packages (a template, a generator script, or
specific directory structure), briefly explain it. If there's a script, just show the
command.

## Things to skip for monorepos
- Detailed docs for individual packages (each package should have its own README)
- Full dependency graphs between packages
- Release process details (put in CONTRIBUTING.md or a release doc)
- CI/CD pipeline descriptions
