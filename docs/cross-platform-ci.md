# Cross-platform CI (Linux / Windows / macOS)

Referenced from CLAUDE.md → Testing. Read when setting up CI, when PR CI takes longer than 10 minutes, or when writing code that runs on more than one OS.

CI MUST work on **Linux, Windows, and macOS** whenever possible.

- MUST use `node:path` with `path.join()` / `path.resolve()` instead of hardcoded `/` or `\\` separators
- MUST use `node:url` (`fileURLToPath`, `pathToFileURL`) for file URL conversions
- NEVER use shell-specific syntax in npm scripts; use cross-platform alternatives:
  - `rimraf` instead of `rm -rf`
  - `shx` or `cpy-cli` instead of `cp` / `mv`
  - Or use Node.js scripts for complex build steps
- NEVER rely on case-sensitive file systems (macOS/Windows are case-insensitive by default)
- SHOULD use `node:os` for platform-specific logic when unavoidable
- MUST cover all three OSes in CI. That does NOT mean one matrix running everything on every
  PR — what runs per OS, and when, is set by the 10-minute budget below. This is the starting
  point for a small repo, and the first thing to split once CI slows down:
  ```yaml
  strategy:
    matrix:
      os: [ubuntu-latest, windows-latest, macos-latest]
  ```

## Keep PR CI under 10 minutes

The wall-clock of a PR's CI — push to the **last required check** finishing — MUST stay under
**10 minutes**. Past that, every review round and every bot fix waits on it, and pushes pile up
behind each other. When it crosses 10 minutes, fixing it is the next CI task, not a someday.

Reference implementation: mulmoterminal `.github/workflows/ci.yml`, `windows-pr.yaml`,
`windows-daily.yaml`, `.github/actions/changed-scope/`. Copy from there rather than re-deriving.

### 1. Measure first — find the critical path

Never shard or move jobs on a guess. Find which job finishes last, and which step inside it costs:

```bash
gh run list --workflow ci.yml --limit 10 --json databaseId,createdAt,updatedAt,conclusion
gh run view <run-id> --json jobs \
  --jq '.jobs[] | {name, startedAt, completedAt}'
gh run view <run-id> --json jobs \
  --jq '.jobs[] | {job: .name, steps: [.steps[] | {name, startedAt, completedAt}]}'
```

Only the job that finishes last is worth speeding up — shaving one off the critical path changes
nothing. Then apply the steps below **in order**; each is cheaper and safer than the next.

### 2. Run platform-independent checks once, apart from the tests

`eslint`, `tsc` / `vue-tsc` and `vite build` answer the same on every runner. Running them per OS
multiplies cost for zero signal.

- One `checks` job on `ubuntu-latest`: lint → typecheck → build.
- A separate `test` job carries the OS matrix. A lint error then surfaces without waiting behind a
  full test run, and neither job waits for the other.
- Only `yarn test` goes into the OS matrix. ubuntu + macOS on every PR when native modules,
  shells or platform branches exist (node-pty, tmux, path handling).

### 3. Shard the test suite

When `yarn test` is the critical path, split it across parallel jobs:

```yaml
env:
  TEST_SHARD_COUNT: "3"   # keep in step with the matrix; a matrix cannot be computed from env
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
        shard: [1, 2, 3]
    steps:
      # ...
      - run: yarn test --shard=${{ matrix.shard }}/${{ env.TEST_SHARD_COUNT }}
```

- vitest: `vitest run --shard=i/N`. `node:test`: `--test-shard=i/N` — confirm the oldest supported
  Node has it before relying on it.
- MUST verify, once when introducing sharding, that the shards together run exactly the files one
  unsharded run does: emit a JSON report per shard (`--reporter=json --outputFile=...`) and compare
  the union of file names against an unsharded run. A shard that collects nothing is still green.
- Files MUST already be isolated from each other (no shared temp dir, port, or global state carried
  between files). If the suite already runs files across workers, this holds.
- Cap the shard count by the runner concurrency your plan allows, macOS above all on a public repo
  — two PRs at once must not queue behind each other's shards. Each shard also pays its own
  install; once install dominates a shard, more shards stop helping.

### 4. Windows: test-only on PRs, the wide run daily

Windows is usually the slowest leg (NTFS cache extraction, Defender, a second tsc pass). Split it
into two workflows, as mulmoterminal does:

| workflow | trigger | runs | why |
|---|---|---|---|
| `windows-pr.yaml` | `pull_request` | `yarn test` only, one Node version, sharded | test-side portability bugs (POSIX path literals, chmod, `HOME` vs `USERPROFILE`, `fs.watch`) are caught **before** merge |
| `windows-daily.yaml` | `schedule` + `push` to main + `workflow_dispatch` | lint, typecheck, build, test, across the Node matrix (e.g. 22.x / 24.x) | the breadth is what makes it slow, and lint/typecheck/build answer the same as ubuntu anyway |

```yaml
# windows-daily.yaml
on:
  schedule:
    - cron: "0 18 * * *"   # 03:00 JST
  push:
    branches: [main]
    paths-ignore: ["docs/**", "plans/**", "**/*.md"]
  workflow_dispatch:
```

- Prefer keeping the test-only PR gate. **Daily-only** means every Windows bug is found after it is
  on main — mulmoterminal ran daily-only first, and a long run of Windows issues landed that way
  before the PR gate was added. Daily-only is the fallback when even the sharded test-only gate
  cannot fit in 10 minutes.
- When Windows IS daily-only, a change touching paths, spawn, env or fs MUST be dispatched at the
  branch before merge: `gh workflow run windows-daily.yaml --ref <branch>`.
- A daily job MUST have a **named route for failures**, or it becomes a red job everyone learns to
  ignore. Add a final step that files an issue (needs `permissions: issues: write`):
  ```yaml
      - name: File an issue on failure
        if: failure() && github.event_name == 'schedule'
        shell: bash   # windows-latest defaults to pwsh, which cannot read ${GITHUB_SHA::7}
        env:
          GH_TOKEN: ${{ github.token }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        run: gh issue create --title "Windows daily failed (${GITHUB_SHA::7})" --body "$RUN_URL" --label ci
  ```
- Defender: disabling realtime monitoring (`Set-MpPreference -DisableRealtimeMonitoring $true`)
  fixes yarn's EPERM-on-rename, but MAY only be done in the daily job (trusted code on main).
  NEVER in a `pull_request` workflow — that switches protection off and then runs code the PR
  supplied.

### 5. Skip what a change cannot reach — inside the job, never in `on:`

A docs/plans/markdown-only PR cannot change lint, build, or a second platform's test result.

- NEVER use `paths-ignore` on a workflow that is (or may become) a **required check**: a skipped
  workflow produces no check at all, and the PR sits on "Expected — waiting for status" forever.
- Instead the job always runs and decides for itself with a scope step (mulmoterminal's
  `.github/actions/changed-scope`): `git diff --name-only $BASE_SHA $HEAD_SHA`, output
  `code=false` only when every path matches `^(docs/|plans/)|\.md$`. Needs `fetch-depth: 0`.
  A non-`pull_request` event, or a diff that fails, answers `code=true` — run everything rather
  than silently skip.
- Gate each step with `if: steps.scope.outputs.code == 'true'`.
- If the test suite **reads docs** (sample validators, README claims), the FIRST platform still
  runs tests on a docs-only PR; only the extra platforms are skipped.
- `paths-ignore` is fine on the daily workflow's `push` trigger — it is not a required check.

### 6. Cache the built tree, not the download cache

- Cache `node_modules` directly with `actions/cache`, keyed on OS + **resolved** Node version
  (`steps.setup-node.outputs.node-version`, for native ABI) + `hashFiles('yarn.lock')`, with a
  prefix `restore-keys` so a lockfile change starts from the previous tree. Still run
  `yarn install --frozen-lockfile` — warm, it is a no-op plus postinstall.
- Do NOT use `setup-node`'s `cache: yarn` for a large tree: it restores the download cache and the
  install still runs in full. On Windows, restoring a yarn cache tar onto NTFS is slower than a
  fresh install.
- With sharding, only shard 1 uses `actions/cache` (save + restore); the others use
  `actions/cache/restore`. Otherwise every shard uploads the same tree and all but one are rejected.
- Restore tool download caches (e.g. `~/.cache/puppeteer`) **before** install so postinstall skips
  the download.
- eslint cache: `eslint --cache --cache-strategy content` in CI (checkout rewrites every mtime, so
  `metadata` invalidates everything). Key it on OS + resolved Node version + lockfile hash +
  `github.run_id`, restore by prefix. The lockfile hash is for **correctness**: eslint does not hash
  the dependency tree, so a cached type-aware result outlives the types it was computed from.

### 7. Stop paying for superseded pushes

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

Cancel on PRs (a new push supersedes the old one); never on pushes to main or release branches,
where nothing else would hold a result for that commit. Every job also gets `timeout-minutes` so a
hang fails instead of holding the runner.

### After restructuring — check it still guards

- Each shard's log reports a **test count**, and the shards add up to the unsharded run.
- On a docs-only PR every required check reports; none sit on "Expected".
- `gh run list --workflow windows-daily.yaml --limit 5` shows the daily actually running, and its
  failure step would fire.
- Re-measure with step 1. Name the command that produced the timing rather than writing the minutes
  into the PR body (CLAUDE.md → *NEVER put a figure in prose*).

## A globbing test script is a Windows failure — and two of the three repairs are worse

`node --test` only gained glob support in Node 22. On Node 20 the positional arguments are literal
paths, and **neither `cmd.exe` nor PowerShell expands a glob on your behalf**, so a script like
`tsx --test ./test/test_*.ts` passes on every POSIX developer machine and fails the moment a Windows
runner joins the matrix.

The failure is not the trap. The trap is that two of the obvious repairs turn the test step into one
that **exits 0 having collected no tests** — measured by running each form under each runtime:

| form passed to `--test` | Node 20 | Node 22+ |
|---|---|---|
| unexpanded glob (`./test/test_*.ts`) | `Could not find '.../test_*.ts'` — fails loudly | ok |
| a directory (`test/`) | **0 tests, exit 0** | `ERR_UNSUPPORTED_DIR_IMPORT` |
| bare `--test`, no positional arg | **0 tests, exit 0** | ok |
| explicit file list | ok | ok |

- MUST name the test files explicitly in the script for as long as Node 20 is supported. It is the
  only form that takes the shell out of the path on every OS and every supported runtime.
- NEVER repair a glob by pointing at the directory or by dropping the argument. On the oldest
  supported Node both are green and both assert nothing, which is strictly worse than the red they
  replaced — a loud failure was traded for a silent one.
- MUST open the Windows job's own log and confirm it reports a **test count** before believing the
  green tick. A real run and a vacuous one are indistinguishable from the checks list.
- The price of the explicit list is that a new `test_*.ts` must be added to the script or it never
  runs. Pay it with a guard test that compares the script's list against the directory, or accept it
  knowingly — but NEVER trade it back for a form that is vacuous on your oldest supported Node.

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
