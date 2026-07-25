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

## Windows-specific traps

**Windows-specific traps** → [`windows-gotchas.md`](windows-gotchas.md). MUST read before debugging a Windows-only failure, and before writing path comparisons or `fs.watch` calls that will run there. Covers: `fs.watch` on an 8.3 short path (`C:\Users\RUNNER~1\…`) making libuv `abort()` the process uncatchably; `path.resolve("/etc")` becoming `<drive>:\etc`, so a POSIX path list silently matches nothing; case-folding path comparisons; reading system dirs from `SystemRoot` / `ProgramFiles` rather than hardcoding a drive letter; and checking that the Windows CI job actually runs on PRs before trusting a green check.
