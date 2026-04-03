# Library / Package README

Libraries exist to be imported by other people's code. The README's job is to get someone
from "what is this" to "working in my project" as fast as possible.

## Sections (in order)

### Features (optional)
Only include if the library does multiple distinct things and the one-line description can't
capture them. Use a short bullet list — 3 to 5 items, each one sentence. Don't list things
that are table stakes (e.g., "well-tested" or "typed" for a TypeScript library).

### Installation
Show the canonical install command. If the library has optional extras or feature flags,
show the base install first and mention extras below.

**Pattern:**
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

### Quick start
Show the single most common use case. This should be a self-contained code block that
someone can paste into a file and run. Include the import, one function call, and the
expected output as a comment.

**Pattern:**
```markdown
## Quick start

```python
from my_lib import process

result = process("input data")
print(result)  # → "processed: input data"
```
```

### Usage (optional)
If the library has 2-3 major use cases beyond the quick start, show them here. Each gets
a `###` subsection with a short code example. Don't turn this into API docs — just cover
the "second thing people want to do" and "third thing."

Keep each example under 10 lines. If it needs more, the library probably has a docs site —
link to it.

### API overview (optional, brief)
Only for small libraries with a small surface area (<10 public functions/classes). List
the main exports in a table:

```markdown
| Function | Description |
|----------|-------------|
| `process(input)` | Transforms input data |
| `validate(schema, data)` | Checks data against schema |
```

For larger libraries, link to generated API docs instead.

### Compatibility
State the minimum language/runtime version. If the library supports multiple versions,
show them. Mention any platform restrictions (e.g., "Linux and macOS only").

```markdown
## Compatibility

Requires Python 3.9+. Tested on CPython and PyPy.
```

## Things to skip for libraries
- Deployment instructions (libraries don't deploy)
- Configuration files (unless the library reads a config file)
- Docker setup
- Environment variables (usually)
