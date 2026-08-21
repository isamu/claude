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

## What survives a differential harness

The `/refactor-safely` harness runs the OLD code beside the new one. Once the refactor lands the old
code is gone, so the harness cannot be kept — half of it no longer exists. Two things inside it can,
and they are the parts that cost the thinking:

- **the generator** — which inputs matter for this function. Empty, one element, unicode past the BMP,
  CRLF, a value at the boundary, malformed, larger than anything real. That list is domain knowledge
  and it is the same list next time somebody touches the function.
- **the property** — what must hold with nothing to compare against. A round-trip (`parse(render(x))
  === x`), an invariant (the output is sorted, the total is preserved), or an oracle: a slow obviously
  correct implementation kept in the test file on purpose.

Harvest those two into a permanent test before deleting the rest. What you keep is no longer a
differential harness; it is a property test, and property tests get slow precisely because they are
worth running over many inputs — which is the real reason to move one off the per-PR path, not that
deleting it felt wasteful.

## A scheduled suite needs an owner and a seed

A generated-input suite too slow for every pull request MAY run nightly or weekly. Two conditions,
and neither is optional:

- **A named route for failures.** A scheduled job that goes red and reaches nobody teaches everyone to
  ignore it, and then it is worse than not existing — it reads as coverage while reporting to no one.
  File an issue automatically, or make it stop a person. If neither is on offer, do not create the job.
- **A printed seed, on every run.** A random generator that fails at 3am is unactionable unless the
  failing seed is in the log and re-running with it reproduces the case. Print the seed on success too:
  the run that passes today is the one you will want to re-run against tomorrow's change.

Also worth deciding before it exists: what happens when it finds something on a commit from three days
ago. A nightly failure names a range rather than a change, so the first move is bisecting the seed
across that range, not reading the diff.

## Test file organization

- MUST place tests in `test/` directory at repository root
- File naming: `test_xxx.ts` (e.g., `test_parser.ts`, `test_utils.ts`)
- For large repositories, SHOULD split into subdirectories (e.g., `test/api/`, `test/utils/`)

## Designing for testability

- MUST separate the **decision** from the I/O: the rule (filter, ordering, cap, validation, formatting, retention) belongs in a **pure function in its own file**; the caller keeps the file reads, spawns, sockets, and HTTP. A rule that can only be reached by booting the app does not get tested.
- MUST use **dependency injection** at the boundary purity can't cross — pass `now()`, `isValidId`, `hasTmux`, `reapSession`, a file path, and the like as parameters instead of importing the real thing. A module that binds a clock, a home directory, or a process at import time cannot be tested without touching the developer's machine; that is a design defect, not a testing inconvenience.
- MUST NOT treat "it is a behaviour-preserving refactor" as a reason to ship no tests. Moving code proves nothing about the rules inside it — extract at least the rules the move exposed, and test those.
- MUST verify that a new test **fails when the code it covers is broken**. Temporarily invert the condition, delete the guard, or revert the fix; watch it go red; restore. A test that also passes against the broken code is testing something else — this happens often and stays invisible unless checked.
- MUST break-verify the **CALL SITE**, not only the rule. Extracting a rule and covering it exhaustively proves the rule; it proves nothing about whether the caller still reaches it, or ever did. Break the call site — swap the reader back for a direct access, drop the guard the caller applies — and watch something go red. If nothing does, the extraction moved the code **out of** the tests' reach rather than into it, and the exhaustive suite you just wrote is describing a function nothing calls.
- When the call-site break stays green, say **which** of the two it is, in the test file. Either the mutation changes no observable answer at that boundary — a real no-op, and the coverage is honest — or it changes one the test cannot see, and the file must say what it does not cover. Both are acceptable; recording neither is how "the rule has thirty cases" comes to read as "the path is covered".
- SHOULD prefer a fake or stub passed in as a parameter over a module mock. Needing a module mock to reach the logic usually means the logic wants extracting.
- SHOULD split into the **smallest pure functions that still have a name worth saying**, and give each its own tests — not one test per file, one per rule. A 40-line function holding four decisions can only be tested through combinations; four named functions can be tested directly, and each one's edge cases become obvious to write.
- MUST apply this to EXISTING code, not only new code. When touching a large file, look for pure rules already buried in it and extract + test them as part of the work. "Testable" is a property of the codebase to actively restore, not a rule that binds only new lines.
- When choosing what to test first, **rank by how silently it fails**. A function that throws is already reporting itself; one that returns a plausible wrong value — an off-by-one index, a byte-vs-character length, a wrong date, a mis-cased extension, a permissive validator, a lookup that reads through the prototype chain — is invisible until a user notices bad data. Those come first.
- SHOULD pin **deliberate asymmetries and known limitations** as tests, with the reason in a comment. Two near-identical helpers that differ on purpose (one trims, one doesn't), a validator that intentionally skips some types, a date epoch correct only after a certain year — record these, or the next reader "fixes" them.
- In a monorepo, check whether a test imports through the **package name** (resolving to built `dist/`) or the source path. If it is the package name, editing the source changes nothing until that package is rebuilt — mutation checks then report "still green" and prove nothing.
