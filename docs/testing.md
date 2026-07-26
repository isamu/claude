# Testing — patterns, golden tests, and designing for testability

Referenced from CLAUDE.md → Testing. Read when writing or refactoring tests.

## Unit-test pattern checklist

For unit tests, MUST cover the following patterns:

- Happy path (expected normal inputs)
- Edge cases (unusual but valid inputs)
- Corner cases (multiple edge conditions combined)
- Boundary cases (min/max values, off-by-one)
- Empty cases (empty string, empty array, empty object)
- Null/undefined cases
- Invalid input (wrong types, corrupted data)
- Error cases (expected failures, thrown exceptions)
- Negative cases (what should NOT happen)
- Regression tests (previously found bugs)

## Golden tests (CI integration tests)

For CI integration tests (CLI execution, program output), MUST use **golden tests**:

- Store the expected correct output (text, files) in git as golden files
- Compare actual output against golden files in CI
- Update golden files explicitly when output intentionally changes

## Test file organization

- MUST place tests in `test/` directory at repository root
- File naming: `test_xxx.ts` (e.g., `test_parser.ts`, `test_utils.ts`)
- For large repositories, SHOULD split into subdirectories (e.g., `test/api/`, `test/utils/`)

## Designing for testability

- MUST separate the **decision** from the I/O: the rule (filter, ordering, cap, validation, formatting, retention) belongs in a **pure function in its own file**; the caller keeps the file reads, spawns, sockets, and HTTP. A rule that can only be reached by booting the app does not get tested.
- MUST use **dependency injection** at the boundary purity can't cross — pass `now()`, `isValidId`, `hasTmux`, `reapSession`, a file path, and the like as parameters instead of importing the real thing. A module that binds a clock, a home directory, or a process at import time cannot be tested without touching the developer's machine; that is a design defect, not a testing inconvenience.
- MUST NOT treat "it is a behaviour-preserving refactor" as a reason to ship no tests. Moving code proves nothing about the rules inside it — extract at least the rules the move exposed, and test those.
- MUST verify that a new test **fails when the code it covers is broken**. Temporarily invert the condition, delete the guard, or revert the fix; watch it go red; restore. A test that also passes against the broken code is testing something else — this happens often and stays invisible unless checked.
- SHOULD prefer a fake or stub passed in as a parameter over a module mock. Needing a module mock to reach the logic usually means the logic wants extracting.
- SHOULD split into the **smallest pure functions that still have a name worth saying**, and give each its own tests — not one test per file, one per rule. A 40-line function holding four decisions can only be tested through combinations; four named functions can be tested directly, and each one's edge cases become obvious to write.
- MUST apply this to EXISTING code, not only new code. When touching a large file, look for pure rules already buried in it and extract + test them as part of the work. "Testable" is a property of the codebase to actively restore, not a rule that binds only new lines.
- When choosing what to test first, **rank by how silently it fails**. A function that throws is already reporting itself; one that returns a plausible wrong value — an off-by-one index, a byte-vs-character length, a wrong date, a mis-cased extension, a permissive validator, a lookup that reads through the prototype chain — is invisible until a user notices bad data. Those come first.
- SHOULD pin **deliberate asymmetries and known limitations** as tests, with the reason in a comment. Two near-identical helpers that differ on purpose (one trims, one doesn't), a validator that intentionally skips some types, a date epoch correct only after a certain year — record these, or the next reader "fixes" them.
- In a monorepo, check whether a test imports through the **package name** (resolving to built `dist/`) or the source path. If it is the package name, editing the source changes nothing until that package is rebuilt — mutation checks then report "still green" and prove nothing.
