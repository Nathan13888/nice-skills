# Severity Guide

The core question for every potential finding:

> **"Would a senior engineer flag this in a design review?"**

If the answer is "probably not" or "it depends on taste," filter it out.

---

## CRITICAL

Issues that represent active danger to production reliability, data integrity, or system correctness. These should be fixed before shipping.

- **Silent data corruption:** Writes that can produce incorrect data without any error signal. Lossy type conversions applied to user data. Encoding mismatches that silently mangle content.
- **Concurrency hazards:** Shared mutable state accessed from multiple threads/goroutines/async tasks without synchronization. Race conditions on data that affects correctness (not just logging).
- **Security boundary violations:** Authentication or authorization checks that can be bypassed. Injection vectors on untrusted input. Hardcoded secrets in source code.
- **Cascading failure risk:** A single component failure that brings down unrelated systems. Missing circuit breakers or timeouts on external dependencies in the critical path. Retry loops without backoff or limits.
- **Untestable architecture:** Core business logic that cannot be tested without standing up external services, databases, or network connections. Tight coupling that makes isolated testing impossible.

---

## MAJOR

Issues that significantly increase maintenance cost, bug risk, or cognitive overhead. These should be on the team's radar and addressed during normal development.

- **God classes/modules:** A single file or class with 5+ distinct responsibilities that should be separate concerns. The "everything drawer" that every feature touches.
- **Shotgun surgery:** Changing one behavior requires coordinated edits to 4+ files across different modules. High blast radius for routine changes.
- **Pattern confusion:** The same problem (error handling, data fetching, validation, etc.) solved 3+ different ways in the codebase with no documented reason for the variation.
- **Complexity walls:** Functions or methods with cyclomatic complexity 15+ or nesting depth 4+. Code that requires holding too much state in your head to understand.
- **Missing boundary error handling:** External calls (network, file I/O, database, APIs) with no error handling, or only happy-path handling that ignores failure modes.
- **Implicit contracts:** Code that depends on undocumented ordering, naming conventions, or magic values. Assumptions that will break silently when someone makes a reasonable change.
- **Abstraction leaks:** Internal implementation details exposed through public interfaces, making it impossible to change the implementation without breaking consumers.

---

## FILTERED OUT (Never Report)

These are real observations but do not meet the bar for a `/sus` finding. Agents must never include these in their output.

- **Naming preferences:** `getUserById` vs `fetchUser` vs `find_user`. Style choices that don't cause confusion.
- **Formatting and whitespace:** Indentation, brace style, trailing commas, import ordering.
- **Missing documentation:** No docstrings, missing README sections, undocumented parameters. (Exception: genuinely misleading or dangerously wrong docs ARE reportable.)
- **Minor DRY violations:** Two occurrences of similar code. Only report at 3+ occurrences AND when the duplication creates a real consistency risk.
- **Unused code:** Dead imports, unreachable branches, commented-out blocks. These are noise, not architecture.
- **TODOs and FIXMEs:** Tracking items left by developers. Not a finding.
- **Test code quality:** Test-specific patterns (verbose setup, repetitive assertions, test helpers) unless they make tests actively misleading.
- **Single-use abstractions:** An interface with one implementation, a factory that creates one type. Only report if the abstraction adds confusion, not just indirection.
- **Language idiom preferences:** Functional vs imperative, classes vs functions, for-loops vs map/filter. Style choices within a language's accepted range.
- **Dependency versions:** Outdated packages, pinned versions, version ranges. This is ops/security tooling territory, not code architecture.
- **Magic numbers/strings in tests:** Test data that is literal values. This is normal and expected.
- **Boilerplate and ceremony:** Language or framework required patterns (e.g., Go error checks, Java getters/setters, React useEffect).
