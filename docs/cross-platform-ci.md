# Cross-platform CI (Linux / Windows / macOS)

Referenced from CLAUDE.md → Testing. Read when setting up CI or writing code that runs on more than one OS.

CI MUST work on **Linux, Windows, and macOS** whenever possible.

- MUST use `node:path` with `path.join()` / `path.resolve()` instead of hardcoded `/` or `\\` separators
- MUST use `node:url` (`fileURLToPath`, `pathToFileURL`) for file URL conversions
- NEVER use shell-specific syntax in npm scripts; use cross-platform alternatives:
  - `rimraf` instead of `rm -rf`
  - `shx` or `cpy-cli` instead of `cp` / `mv`
  - Or use Node.js scripts for complex build steps
- NEVER rely on case-sensitive file systems (macOS/Windows are case-insensitive by default)
- SHOULD use `node:os` for platform-specific logic when unavoidable
- MUST include all three runners in GitHub Actions matrix:
  ```yaml
  strategy:
    matrix:
      os: [ubuntu-latest, windows-latest, macos-latest]
  ```

## Tests that handle paths

A rule that resolves paths with `node:path` produces a **different string on each OS** — on Windows
it is drive-qualified (`path.resolve("/lib")` is `<drive>:\lib`, not `\lib`) — so a test that
hardcodes the expected value only agrees with the developer's machine. `yarn test` stays
green locally and the Windows job goes red — the one place that can see it is the one place that
is not run before merge.

- MUST build the **expected** path with `path.resolve()` / `path.join()` too, never as a POSIX
  literal:
  ```ts
  const BASE = path.resolve("/repo");            // C:\repo on Windows, /repo elsewhere
  const SIBLING = path.resolve(BASE, "../lib");  // the expectation, computed the same way
  ```
- MUST resolve the operand of a **stub that compares paths** as well. This is the case that fails
  silently: a predicate asking `p === "/denied"` simply never matches on Windows, so the branch it
  was meant to exercise (a throwing check, a rejected path) never runs and the test passes while
  asserting nothing. A path-comparing stub is a path comparison.
- SHOULD dispatch the Windows job at the branch ref before merging when a change touches path
  handling — `gh workflow run <workflow>.yaml --ref <branch>`. On a macOS/Linux-only developer
  machine no amount of local testing can see this class.

## Windows-specific traps

**Windows-specific traps** → [`windows-gotchas.md`](windows-gotchas.md). MUST read before debugging a Windows-only failure, and before writing path comparisons or `fs.watch` calls that will run there. Covers: `fs.watch` on an 8.3 short path (`C:\Users\RUNNER~1\…`) making libuv `abort()` the process uncatchably; `path.resolve("/etc")` becoming `<drive>:\etc`, so a POSIX path list silently matches nothing; case-folding path comparisons; reading system dirs from `SystemRoot` / `ProgramFiles` rather than hardcoding a drive letter; and checking that the Windows CI job actually runs on PRs before trusting a green check.
