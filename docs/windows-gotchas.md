# Windows gotchas (Node.js)

Failure modes that only appear on Windows, each one hit and diagnosed in real
work rather than collected from documentation. The generic cross-platform rules
(`node:path` over string concatenation, no shell syntax in npm scripts, the
three-OS CI matrix) live in `CLAUDE.md` — this file is for the traps those rules
don't cover.

---

## `fs.watch` on an 8.3 short path aborts the process

**Symptom.** The process dies. Not an exception — the whole thing:

```
Assertion failed: !_wcsnicmp(filename, dir, dirlen), file src\win\fs-event.c, line 72
```

A test runner reports only `'test failed'` for the whole file, with no assertion
and no stack, which is what makes this expensive to diagnose.

**Cause.** `ReadDirectoryChangesW` reports filenames against the **long** path,
but a watch opened on an 8.3 short path keeps the short form. libuv asserts that
the reported filename is prefixed by the watched directory; when the two
spellings disagree it calls `abort()`.

`os.tmpdir()` on GitHub's Windows runners returns
`C:\Users\RUNNER~1\AppData\Local\Temp` — a short path. So **any watcher mounted
under tmpdir on CI is one filesystem event away from killing the run.**

**Why you can't catch it.** It is a native assert inside libuv, so
`watcher.on("error")`, `try`/`catch`, and `process.on("uncaughtException")` are
all bypassed. There is no defensive coding around it — the path has to be fixed
before `watch()` is called.

**Fix.** Resolve to the real path first:

```js
function watchablePath(dir) {
  if (process.platform !== "win32") return dir;
  try {
    return realpathSync.native(dir);   // C:\Users\RUNNER~1\… → C:\Users\runneradmin\…
  } catch {
    return dir;                        // no worse off than before
  }
}
```

Guard it to win32. On POSIX `realpath` also collapses symlinks (`/var` →
`/private/var` on macOS), which changes behaviour you probably didn't intend to
change.

---

## POSIX absolute paths silently become drive-relative

`path.isAbsolute("/etc")` is **true** on Windows, and `path.resolve("/etc")`
yields `<current-drive>:\etc`. Both facts together mean a hardcoded list of
POSIX paths passes every "is this absolute?" check and then matches nothing:

```js
// On Windows this list is dead weight — nothing ever equals "/etc"
const BLOCKED = ["/etc", "/root", "/var"];
BLOCKED.some((p) => path.resolve(input) === p);   // always false
```

A blocklist written this way looks like it is protecting you and is not. Give
each platform its own vocabulary rather than hoping one list covers both.

---

## Path comparison must be case-folded

Windows filesystems are case-insensitive: `c:\windows` and `C:\Windows` are the
same directory. A verbatim `===` or `.startsWith()` lets the other spelling
walk straight past an allowlist or blocklist.

```js
const key = (p) => (process.platform === "win32" ? p.toLowerCase() : p);
```

Keep the `+ path.sep` guard when testing "is under" — `key.startsWith(blocked)`
alone also swallows `C:\Windows-backup`.

---

## Never hardcode the drive letter

The system drive is not always `C:`. Hardcoding `C:\Windows` means the check
silently protects **nothing** on a machine booted from another drive — worse
than an obvious failure, because it looks correct in review.

Read them from the environment instead, and treat unset values as absent:

| Variable | Typical value |
| --- | --- |
| `SystemRoot` / `windir` | `C:\Windows` |
| `ProgramFiles` | `C:\Program Files` |
| `ProgramFiles(x86)` | `C:\Program Files (x86)` |
| `ProgramData` | `C:\ProgramData` |

---

## Confirm the Windows CI job actually runs on your PR

A repo may deliberately keep Windows out of the per-PR matrix to save
wall-clock, running it only on a schedule and on pushes to the default branch.
When that is the case, **a green PR proves nothing about Windows** — the job
never ran.

Check the workflow's `on:` block before trusting green, and trigger it by hand
when the change is Windows-specific:

```bash
gh workflow run <workflow>.yaml --ref <branch>
```

Seen in mulmoclaude, whose `lint_test_windows.yaml` triggers on `schedule`,
`push` to main, and `workflow_dispatch` — but not `pull_request`.
