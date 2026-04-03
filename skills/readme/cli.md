# CLI Tool README

CLI tools are used from a terminal. The README should show installation, the 2-3 most
useful commands, and what the output looks like.

## Sections (in order)

### Installation
Show the preferred install method. For CLIs, there are often multiple ways — show the most
universal one first (e.g., a package manager install), then put alternatives in a
`<details>` block.

**Pattern:**
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

### Quick start
Show 1-3 commands that demonstrate the core use case. Include realistic (but concise)
output so people know what to expect.

**Pattern:**
```markdown
## Quick start

```bash
$ my-tool analyze src/
✓ 42 files scanned
✓ 3 issues found

  src/auth.py:12    unused import 'os'
  src/db.py:45      connection not closed
  src/api.py:89     hardcoded secret
```
```

The `$` prefix on the command line and the output block help readers distinguish what they
type from what the tool prints. Use this pattern consistently.

### Commands / Usage
Show the 3-5 most important subcommands or flags. Don't reproduce `--help` output — 
summarize the most useful ones in a table or short list.

**Pattern:**
```markdown
## Usage

| Command | Description |
|---------|-------------|
| `my-tool analyze <dir>` | Scan files for issues |
| `my-tool fix <dir>` | Auto-fix what it can |
| `my-tool init` | Create a config file |

Run `my-tool --help` for the full command reference.
```

If the tool has a small number of flags (< 6), a table works well. If it has many
subcommands, list just the top ones and link to full docs.

### Configuration (if applicable)
If the tool reads a config file, show a minimal example with the most common options.
Don't show every option — link to a config reference doc.

```markdown
## Configuration

Create a `.my-tool.toml` in your project root:

```toml
[rules]
ignore = ["src/vendor/**"]
severity = "warning"
```

See [Configuration Reference](docs/config.md) for all options.
```

## Things to skip for CLIs
- Library/import usage (unless it's also a library — in that case, document as library
  with a CLI section)
- API reference tables
- Detailed flag descriptions (that's what `--help` is for)
