# Agent Roster

## Default Agents (5)

These agents deploy on every `/sus` run unless overridden.

### Architecture Scout

- **subagent_type:** `Plan`
- **Focus:** Structural integrity of the codebase — module boundaries, responsibility placement, dependency direction.
- **What to look for:**
  - Misplaced responsibilities (business logic in controllers, I/O in domain models, presentation in data layers)
  - Layer violations (inner layers importing from outer layers, skipping layers)
  - Circular dependencies between modules or packages
  - God modules/files that have accumulated too many responsibilities (5+ distinct concerns)
  - Entrypoints that do too much (main functions that are mini-frameworks)
  - Orphaned abstractions (interfaces/traits with a single implementation that add indirection without value — but only when there's no testing or extensibility justification)
- **Exploration strategy:** Start with the directory tree and module/package structure. Read entry points, then follow the import/dependency graph. Focus on boundaries between modules, not internals.

### Complexity Hunter

- **subagent_type:** `Explore`
- **Focus:** Code that is disproportionately hard to understand, modify, or test due to structural complexity.
- **What to look for:**
  - Deep nesting (4+ levels of indentation in logic, not data structures)
  - Long functions (50+ lines of logic, excluding boilerplate/config)
  - Complex conditionals (3+ boolean conditions, nested ternaries, flag-driven branching)
  - Boolean/string flag parameters that create hidden control flow
  - Functions with 5+ parameters (especially when several are the same type)
  - State machines implemented as implicit if/else chains instead of explicit states
- **Exploration strategy:** Search for long files first, then scan for deep nesting patterns and complex conditionals. Look at the most-changed or most-imported files — complexity tends to accumulate there.

### Coupling Detector

- **subagent_type:** `Explore`
- **Focus:** Hidden dependencies and tight coupling that make changes expensive and risky.
- **What to look for:**
  - Shotgun surgery patterns (changing one behavior requires touching 4+ files)
  - Feature envy (a function that uses more data from another module than its own)
  - Abstraction leaks (internal implementation details exposed through public APIs)
  - Implicit contracts (code that depends on ordering, naming conventions, or magic values without enforcement)
  - Deep coupling through shared mutable state or global variables
  - Connascence of meaning (multiple places that must agree on a magic string, number, or convention)
- **Exploration strategy:** Look at imports/requires across module boundaries. Find functions that take or return types from other modules. Search for shared constants, enums, or config values used across many files.

### Consistency Auditor

- **subagent_type:** `general-purpose`
- **Focus:** Inconsistency that signals unfinished migrations, unclear conventions, or accumulated confusion.
- **What to look for:**
  - Same problem solved 3+ different ways (e.g., three error handling patterns, three ways to make HTTP calls)
  - Half-migrated patterns (old approach and new approach coexist with no clear migration path)
  - Convention drift (most of the codebase does X, but a few files do Y with no apparent reason)
  - Naming inconsistency that creates real confusion (not cosmetic — e.g., `user_id` vs `userId` vs `owner` all meaning the same thing across module boundaries)
  - Mixed paradigms that create cognitive overhead (OOP and FP patterns interleaved without a clear boundary)
- **Exploration strategy:** Sample files across different directories and time periods. Look at how common operations (error handling, logging, data access, validation) are implemented in different parts of the codebase. Focus on patterns, not individual instances.

### Risk Assessor

- **subagent_type:** `Plan`
- **Focus:** Code that is one mistake away from a production incident — missing guards, unsafe assumptions, fragile contracts.
- **What to look for:**
  - Missing error handling at system boundaries (network calls, file I/O, database queries, external APIs with no error handling or only happy-path handling)
  - Implicit contracts that will break silently (assumptions about data format, ordering, or presence that aren't validated)
  - Unsafe type coercions, casts, or conversions (especially `as any`, `unsafe`, unchecked unwrap on fallible operations)
  - Cascading failure risk (one component's failure bringing down unrelated components)
  - Data integrity gaps (writes without validation, reads without null checks at trust boundaries)
  - Race conditions or concurrency hazards (shared mutable state accessed without synchronization)
- **Exploration strategy:** Focus on system boundaries — where the code talks to databases, APIs, file systems, or user input. Read error handling paths and failure modes. Look for `unwrap`, `as any`, bare `except:`, empty catch blocks, and similar patterns.

---

## Optional Specialist Agents

These agents are available for targeted analysis. Activate them with `/sus add: {name}` or `/sus agents: {list}`.

### Security Sentinel

- **subagent_type:** `general-purpose`
- **Focus:** Security vulnerabilities and unsafe patterns that could be exploited.
- **What to look for:**
  - SQL injection, command injection, path traversal vectors
  - Hardcoded secrets, API keys, or credentials
  - Missing authentication/authorization checks on sensitive endpoints
  - Insecure deserialization or eval-like patterns
  - CSRF, XSS, or SSRF vulnerabilities
  - Overly permissive CORS, file permissions, or access controls

### Performance Oracle

- **subagent_type:** `Explore`
- **Focus:** Performance bottlenecks and scalability concerns.
- **What to look for:**
  - N+1 query patterns (database queries inside loops)
  - Unbounded data loading (no pagination, no limits on query results)
  - Synchronous blocking in async contexts
  - Missing caching where repeated expensive operations occur
  - O(n^2) or worse algorithms on potentially large datasets
  - Memory leaks (event listeners not cleaned up, growing caches without eviction)

### Concurrency Analyst

- **subagent_type:** `Plan`
- **Focus:** Concurrency bugs, race conditions, and thread-safety issues.
- **What to look for:**
  - Shared mutable state without synchronization primitives
  - Lock ordering violations (potential deadlocks)
  - Time-of-check-to-time-of-use (TOCTOU) bugs
  - Missing atomicity in multi-step operations
  - Async/await pitfalls (unawaited promises, fire-and-forget without error handling)
  - Unsafe concurrent collection modifications

### API Contract Reviewer

- **subagent_type:** `general-purpose`
- **Focus:** API design issues that will cause pain for consumers and future maintainers.
- **What to look for:**
  - Breaking change risks (public APIs that are hard to evolve without breaking consumers)
  - Inconsistent API patterns (different endpoints following different conventions)
  - Missing or misleading status codes/error responses
  - Overly chatty APIs (requiring many calls for common operations)
  - Leaking internal implementation details through API responses
  - Missing versioning strategy for public APIs

---

## Customization

### Inline Overrides

Override the agent roster directly when invoking the skill:

```
/sus focus: security          # Security-focused run (Security Sentinel + Risk Assessor + default)
/sus agents: Architecture Scout, Security Sentinel   # Exactly these agents
/sus add: Concurrency Analyst  # Default 5 + Concurrency Analyst
```

**Focus shortcuts:**

| Focus         | Agents deployed                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------ |
| `security`    | Security Sentinel, Risk Assessor, Coupling Detector, Architecture Scout, Consistency Auditor     |
| `performance` | Performance Oracle, Complexity Hunter, Coupling Detector, Architecture Scout, Risk Assessor      |
| `concurrency` | Concurrency Analyst, Risk Assessor, Coupling Detector, Complexity Hunter, Architecture Scout     |
| `api`         | API Contract Reviewer, Consistency Auditor, Coupling Detector, Risk Assessor, Architecture Scout |

### Harness Override

If your project has an `/agents` skill or agent configuration, `/sus` will respect it. Define your custom roster there and `/sus` will pick it up in Step 2.
