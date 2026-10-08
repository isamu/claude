# Global Claude Code Settings

> **MUST** / **NEVER** = mandatory. **SHOULD** = recommended unless there is a clear reason not to. **MAY** = optional.
>
> This file is the rules. The depth behind them — methodology, case studies, platform traps — lives in [`docs/`](docs/) and in skills. MUST follow the pointer before non-trivial work in that area; a rule here without its doc is the summary, not the whole answer.

## Prefer a skill over doing the work by hand

The harness lists every available skill with its description — check it before hand-rolling a multi-step task. The house defaults:

- Start a new project → `/init-project`
- Publish an npm package / release the MulmoClaude app → `/publish`, `/release-app`
- Draft a GitHub issue → `/issue-draft`; triage PR bot reviews → `/gh-review-loop`
- Review, refactor, or security-check a change → `/code-review`, `/simplify`, `/security-review`
- Any change claiming "this behaves the same" → `/refactor-safely`
- Split a long or complex function, or clear a lint bound on one → `/decompose-function`
- Run / verify / UI-test / profile a web app → `run`, `/verify`, `/pr-ui-test`, `/web-perf`
- Tech-blog article from a merged PR → `/pr-to-tech-blog`

## Environment & Packages

- When today's date is needed, MUST run the `date` command — NEVER rely on the model's internal knowledge
- MUST use **yarn** (`yarn`, `yarn add`, `yarn remove`); NEVER use npm commands
- MUST use `yarn add` instead of manually editing package.json
- During upgrade work, if a dependency turns out to be unused, MUST propose removing it (`yarn remove`) rather than upgrading it
- **NEVER run `npm publish` yourself** — it needs interactive auth (OTP / 2FA) on every run, so it cannot complete in a tool call. MUST print the exact command and the directory, hand it to the user, and wait for them to report the result before continuing

## Git Operations

- NEVER run `git commit`, `push`, `merge`, `rebase`, or similar without explicit user permission. Read-only operations (`git status`, `git diff`, `git log`) MAY be run freely
- MUST **check the current branch** before making changes; if it differs from expected, ask the user which to use
- MUST **create a feature branch** before starting implementation work — and MUST `git fetch` first, checking `git log HEAD..origin/<default-branch>`. Branch only from an up-to-date base: work built on a stale mainline has to be relocated wholesale. "The repo looked fine when I read it" is not evidence — the working copy can be stale
- NEVER push directly to main — MUST open a PR, and MUST confirm the target branch first
- NEVER use `git add .` or `git add <directory>` — add files individually
- NEVER delete untracked files
- NEVER use `git rebase`. MUST merge PRs with a merge commit (`--merge`) — NEVER squash
- NEVER `git push --force`. `--force-with-lease` is permitted only on a feature branch you just pushed yourself, after a local `git commit --amend` or similar — it aborts safely if anyone else pushed. NEVER force-push, any variant, to `main` or a shared branch
- MUST use commit prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- SHOULD commit after each meaningful change (schema done → commit, utilities done → commit)
- "create a PR" / "PR、マージ" MUST be read as CREATE a pull request, not merge it

## GitHub Issues & PRs

When asked to fix a bug or implement a feature:

1. MUST discuss and clarify requirements with the user
2. MUST create a GitHub issue once scope is clear — **before** the work, not after it is written. Duplicate check, reachability, observed-vs-inferred, regression archaeology, thresholds → [`docs/issue-filing.md`](docs/issue-filing.md), or use `/issue-draft`
3. MUST create a plan file under the repo's `plans/` directory (`plans/fix-xxx.md`, `plans/feat-xxx.md`), committed to the repo
4. MUST implement based on the plan
5. The PR description MUST open with **Summary** and **Items to Confirm / Review** — what changed, and what the author specifically wants a human to check (risky decisions, assumptions, unverified behaviour). AI-generated code makes this mandatory, not optional
6. MUST then include a **User Prompt** section carrying the user's original request. Across multiple turns, consolidate the user's messages into a coherent summary preserving all intent; clean up formatting but NEVER add content beyond what the user said. Multiple distinct requests → one bullet each
7. MUST include the implementation approach and key decisions in the PR description — information discussed in chat MUST be persisted as files or PR comments, NEVER left only in chat

- **One feature, one set of invariants, one PR.** Changes that are right or wrong *together* belong together; anything that could be merged or reverted on its own belongs apart. When the reviewer is another model this matters more than line count — a 2000-line generated migration can be fine and a 150-line change touching auth, caching and concurrency is not. Before opening, ask which parts could be **reverted on their own**; every answer is a PR that should have been separate. Sizing, the PR contract a reviewing model needs, and why a bigger PR compounds through a review loop → [`docs/pr-contract.md`](docs/pr-contract.md)
- After pushing, MUST triage **every** bot reviewer, not just the first. Use `/gh-review-loop`; principles, and the "never wait for CodeRabbit / docs-only PRs don't wait at all" exceptions → [`docs/pr-bot-review.md`](docs/pr-bot-review.md)
- A PR addressing only PART of an issue MUST NOT use `Closes #N` — GitHub ignores the prose around the keyword. See [`docs/issue-filing.md`](docs/issue-filing.md)

## Working one backlog issue in parallel

A tracking issue holding a list of entries — a lint backlog, a migration, an audit — invites several agents at once. What breaks is never the code; it is the *coordination*, and every rule below was paid for. Depth and the case studies → [`docs/parallel-tranches.md`](docs/parallel-tranches.md). MUST read before running more than two at a time.

**Before starting**

- MUST **claim the entry on the issue before the work starts** — a comment naming what will be touched as a **function or line range**, not just a file. Two tranches at opposite ends of one file is fine; two in one function is not, and only a specific claim tells them apart. Dispatching a batch → one comment covering all of them, plus a "not claimed" line so the remaining surface stays readable
- MUST run the collision check as **two** commands: `gh pr list --state open` **and** `git log origin/main --oneline -20 -- <file>`. A PR that merged an hour ago appears in neither the open list nor the working tree
- MUST confirm the target is **reachable** before extracting from it — `grep` for the caller, `git log --all -S "<symbol>"`. Extracting from an orphan gives it named exports and tests, and argues to the next reader that it is load-bearing. `knip` answers *"is this file imported"*, NOT *"can this be reached"*: a type-only import and a test-only reader each keep a file "used", so a `.tsx` target needs `grep -rn "<ComponentName"` as well
- **One file per agent.** NEVER two agents in one file, whatever the line distance between them

**How many**

- MUST check `uptime` before adding a worktree or an agent, and MUST NOT add one while the load average is already high — it is the *existing* work that pays, in timeouts that then have to be re-run and re-judged. Budget against cores, not against ambition
- "The machine is loaded" MUST stop new work, not merely slow it. Reviews of what is already in flight continue, sequentially

**Isolation**

- Each agent MUST get its own git worktree. `node_modules` MUST be symlinked rather than installed per worktree — and the symlinks MUST be removed before the agent finishes. A leftover one makes `npx eslint .` lint a whole second copy of the repo, silently corrupting every count taken afterwards
- NEVER run a command that mutates shared state from a worktree — `prisma generate` above all, when several tranches share one client. A stale client's errors belong to the environment and MUST NOT be chased as if they were the change

**Measuring under concurrency**

- A timeout at high load is the load, not the change. MUST re-run the file standalone (`--testTimeout=120000 --hookTimeout=120000`) before concluding a failure is yours, and MUST say which failures were re-run
- A mutation sweep MUST assert the file matches a pristine copy **before** each mutation as well as after each restore. A fifty-minute sweep is a fifty-minute window for a sibling's `cp`, and a stray mutation reads as *more* coverage rather than less

**When an agent dies mid-flight** (session limit, crash)

- MUST resume rather than restart — the worktree holds the work. MUST re-establish state first (`git status`, `git log`, re-run the lint) and MUST verify the source matches the intended version before measuring anything: an interrupted sweep may have left a mutation behind
- MUST re-sync `origin/main` on resume and re-run the gates on the merged result — CI runs against a combination nobody has executed

## Change Scope Rules

- MUST only make changes that were explicitly requested — NEVER autonomously add features, tools, packages, or content
- MUST ask first if something additional seems needed
- MUST keep PR comments, commit messages, and documentation concise unless asked otherwise

## Code Quality

- MUST run after making code changes: `yarn format` → `yarn lint` → `yarn build` → `yarn typecheck`
  - If a `typecheck` script exists, MUST run it. Many repos split `build` (compile-only, tsconfig.build.json, often excludes `test/`) from `typecheck` (full project, includes tests). CI runs typecheck, so `build` passing proves nothing when a test file references a type you tightened — skipping this is the most common cause of "passed locally, failed in CI"
  - **When the machine is already loaded** (check `uptime`), a SMALL change MAY skip the local run and be verified by pushing and reading CI instead. Adding one more build to a machine that is thrashing costs the work already in flight, and CI is the ground truth those commands are approximating anyway. Say in the PR that CI is the verification. This is for a one-line fix, a doc edit, a rename — NOT for anything the "behaves the same" or "wide blast radius" rules cover, which need a run whatever the load is
- MUST check for duplication against the existing codebase after a significant implementation, and refactor to eliminate it. Prioritise readability
- SHOULD run `/code-review` after a significant implementation (`--fix` to apply, `--comment` to post inline PR comments); `/simplify` for quality-only refactors, `/security-review` when the change has a security surface

### "This behaves the same" MUST be proved by running both, never by reasoning

Any change whose claim is behaviour preservation — an extraction, a rule lifted into a function, a regex rewritten as a scan, a condition "simplified", a deletion, a lint sweep — MUST be verified by copying the OLD code verbatim into a throwaway harness, running it beside the new code over **generated** inputs, comparing whole results, and stating the count. Unit tests on the new code pin what you *meant*; the risk is what you changed without noticing. **Before deleting it, harvest the two parts that outlive it**: the *generator* (which inputs matter for this function) and the *property* (what must hold with the old code gone). Those become a permanent test — the differential harness itself cannot survive, because half of it is the code you just deleted. **MUST use `/refactor-safely`** — it carries the method, the shapes these changes actually break, and the traps that make a broken change look green.

### A wide blast radius is verified by RUNNING THE APP, not by a green suite

A passing suite proves the code you thought about still behaves. It does not prove the app boots, the route is still wired, or the middleware still runs in the order the framework needs. When a change touches something EVERY request or EVERY caller passes through — a route entry point, a middleware, a handler signature, a shared function with many call sites — MUST start the stack and drive the real path: the happy path AND the rejection, varying what the change assumes. MUST name what you could NOT exercise locally and why; silence reads as "the run covered everything". Confirm the process you started is the checkout you changed — something already answering on the port is not evidence. Details → `/refactor-safely` → *What a green suite does not prove*.

### The point of an extraction is SAFETY, not size — and a costed "no" beats a risky lift

Splitting a large function is worth doing because it makes a rule **catchable**, not because the
number of lines fell. Rank the outcomes and take the highest one available:

> a decision made testable > a block moved with its behaviour **proved** > a block moved with its
> behaviour **argued** > nothing

**The last is a real option and it beats the third.** When you cannot establish that a lift preserves
behaviour, do not lift it — report it costed (what it would buy, what the risk is, what would have to
be established first) and move on. That report is a first-class deliverable: the next person reads it
instead of rediscovering it. This overrides the 20-line rule, which is a target and not a licence.

**MUST measure the safety, not the shrinkage:** invert each decision you extracted and run the
**pre-existing** suite against both trees, with any test you added **excluded from the denominator**.
A denominator holding your own new tests reports a perfect score and says nothing about what shipped.
The number that means something is *"N decisions that could be silently broken now go red"* — which is
how a session found a `?? → ||` on a token count that inverted with every one of 163 shipped tests
green, on a value that is billing input. Depth, and the ways this measurement lies →
[`docs/testing.md`](docs/testing.md)

## Debugging Approach

- MUST establish the issue is REAL before designing a fix — reproduce it, or trace one concrete call chain from a real entry point to the line and name the caller that supplies the input. A report from a user, a bot, a failing test, or your own reading of the code is a **claim**, and the entire fix rests on it being true. If it cannot be reproduced or reached, MUST say so with what was checked, and stop — a fix for a case that cannot happen costs a real review and hides the real defect
- Once it is real, MUST widen the view before narrowing the fix: the reported case is one instance of a rule — which other call sites decide the same thing, which sibling inputs take the same branch, which layer the rule actually belongs in. MUST state the scope chosen and what was deliberately left out; the symptom is where the bug surfaced, not where it lives
- MUST shape the fix so the defect becomes testable: extract the rule around the bug into a **pure function in its own file** (no fs / network / clock / process — inject what it needs), route the caller through it, and cover it exhaustively in BOTH directions — normal inputs AND abnormal ones (empty, null/undefined, wrong type, boundary, malformed, thrown errors). A rule reachable only by booting the app is a bug that comes back → [`docs/testing.md`](docs/testing.md)
- MUST diagnose the ROOT CAUSE before attempting fixes; NEVER reach for a quick fix (hardcoded values, JSON workarounds)
- MUST understand what the user is asking before jumping to debug
- MUST check git history / diffs when investigating a regression
- MUST verify a fix against an **external ground truth**, never against another of your own outputs. Two things you produced agreeing proves only that they share your assumptions. Find the authority that already knows the answer — `tmux capture-pane` for a terminal screen, the server's own log for what was sent, the real file on disk — and diff against that (mulmoterminal #1073: "render with the fix" vs "without it" came out identical and shipped; both were diverging from the real screen, which `capture-pane` would have shown in one command)
- MUST vary the conditions the fix depends on before declaring it verified. One run at one size, one timing, one ordering tests a single point — and the bug lives in what you held constant. Name what the fix assumes and move each one
- Judging whether a report is real, bug-family matrices, sweeping a rule across every call site, adversarially reviewing retry/replay, deterministic per-branch repro, and never trusting an error string as the only evidence → [`docs/debugging-methodology.md`](docs/debugging-methodology.md). MUST read before a non-trivial bug hunt

## Testing

- SHOULD use Node.js native `node:test` and `node:assert` by default; if the project already uses another runner (e.g. vitest, as in Cloudflare Workers projects), MUST follow the existing one
- MUST mock external APIs — tests MUST run without API keys
- MUST place tests in `test/` at the repo root, named `test_xxx.ts`; MUST add a `test` script to package.json and run it in CI
- Unit-test pattern checklist, golden tests, and the **designing-for-testability** rules → [`docs/testing.md`](docs/testing.md). MUST read before writing or refactoring tests
- A generated-input suite too slow for every PR MAY run on a schedule instead — but only with a **named route for failures** (an issue filed automatically, or a person it stops) and a **printed seed**. Without the first it is a red job everyone learns to ignore; without the second "it failed last night" is unreproducible → [`docs/testing.md`](docs/testing.md)
- Cross-platform CI (Linux/Windows/macOS matrix, `node:path` / `node:url` portability) → [`docs/cross-platform-ci.md`](docs/cross-platform-ci.md); Windows-only traps (`fs.watch`, `path.resolve`) → [`docs/windows-gotchas.md`](docs/windows-gotchas.md). MUST read before debugging a Windows failure
- PR CI (push → last required check) MUST finish in **under 10 minutes**. When it doesn't, measure the critical path, then split: lint/typecheck/build once on ubuntu, shard the tests, Windows test-only on PRs with the wide run daily, skip docs-only changes inside the job (never `paths-ignore` on a required check) → [`docs/cross-platform-ci.md`](docs/cross-platform-ci.md) → *Keep PR CI under 10 minutes*. MUST read before restructuring CI

## Coding Style

**Write for human comprehension.** Human context and memory are limited: compact functions a reader can hold at a glance, names that tell a story, minimal variable scope, a flow that reads as a narrative.

- MUST keep functions under 20 lines; split into smaller functions if needed — but **safety outranks size**, and an extraction you cannot show to be behaviour-preserving is one to decline rather than to make carefully (see *The point of an extraction is SAFETY, not size*)
- MUST prefer `const` over `let`; NEVER use `var`
- MUST prefer `forEach` / `map` / `filter` / `reduce` over `for` loops
- MUST prefer `async/await` over `.then()` chains
- MUST use explicit type definitions; NEVER use `any`
- NEVER silence lint/type errors with `eslint-disable`, `@ts-ignore`, or `@ts-expect-error` — fix the types / root cause instead (define proper type files if needed)
- NEVER use magic numbers; MUST use named constants
- SHOULD include units in variable names (`timeout_ms`, `distance_km`)
- MUST follow DRY
- MUST add try/catch for operations that can fail. Network requests MUST include AbortController timeout handling, and errors MUST carry context (URL, file path)
- MUST NOT read a file whole unless you know it is bounded — `readFile` throws past ~512 MB and the `catch` reports it as empty, so the biggest data reads as the emptiest → [`docs/large-file-reading.md`](docs/large-file-reading.md)

### A stale plan or comment is not worth a review round — a wrong line of code is

The rule above is about figures. This one is about the whole surface, and it came from the same
place: **the review loop is for the code.** A plan file that describes an earlier version of the
change, a comment that names a helper since renamed, a PR body a few commits behind — none of
those breaks anything, and every round spent correcting one is a round not spent on the diff.

- **Do not let a reviewer spend rounds on prose.** Say so in the brief: report code findings;
  prose only when it would mislead someone into a WRONG CHANGE, not when it is merely behind.
- **Do not fix prose mid-loop** unless it is load-bearing — a comment that would make the next
  reader do the wrong thing, or a claim the code contradicts in a way that matters. Staleness
  that a reader would simply skip past is not that.
- **Bring the prose up to date once, at the end**, when the code has stopped moving. Correcting
  it repeatedly against a moving diff is how the same paragraph gets rewritten several times and
  is stale again by the merge.
- The exception stays what it always was: a comment that would cause a defect if believed. That
  is a code finding wearing prose, and it is worth the round.

### NEVER put a figure in prose — say what moved, not by how much

**A number written into a PR body, a commit message, a plan file or a code comment is a
liability with no upside.** It is true for one commit. The next commit — often your own, later
the same hour — makes it false, and nobody notices because nothing executes prose. Then a
reviewer checks it, finds it wrong, and a whole review round is spent on a correction that
improves no code.

Measured on one orion session: a single figure went stale THREE times in a row, each time
because the next commit added comment lines to the function being measured; a "six of eight"
was really seven; a ratchet floor was lowered by two because a comment claimed a cushion that a
previous tranche had already spent. Every one of those cost a round, and not one of them changed
what shipped.

So:

- **Write the direction and the mechanism, never the magnitude.** "The bound is clear now",
  "the entry is gone", "one warning left, and this removes it" — not "114 → 96", not
  "8,549 passed", not "seven of sixteen shapes".
- **Numbers belong where something executes them**: a constant in code, an assertion in a test,
  a threshold in a config. Those are checked by running, so they cannot rot silently. A number
  in a sentence is checked by a human who will eventually be you.
- **Say what to run instead of what it printed.** "`yarn lint` on this file is clean" beats
  "warnings 1 → 0", because the reader can reproduce the first and can only doubt the second.
- **If a magnitude is genuinely the point** — a benchmark result, a cost, a regression
  threshold — put it in a file the tooling regenerates, or pin it to a sha, and say which
  command produced it.

This does not license vagueness about WHAT was verified. "Nine timeline shapes came back
identical" is a measurement someone can re-run; "much faster" is not. The rule is against the
digit, not against the evidence: name the instrument and the outcome, and let the instrument
hold the number.

### A name you have to look up is debt

**If a reader cannot tell WHAT a value is from its name alone, the name is wrong.** Not merely
unclear — wrong, and it is debt every later reader pays. The test is not *"is this a real word"*
but **"could this be several different things in this codebase?"** If yes, the name has not done
its job.

The names that keep failing are the short conventional ones, because convention is exactly what
stops people asking:

| name | why it fails | what it should say |
|---|---|---|
| `res` | response to whom, carrying what | `sseResponse` — the Express response SSE frames are written to |
| `ctx` | context of what | `context` at minimum; better if it names the context |
| `timers` | which timers, owned by what | `pollTimers`, `turnTimeoutTimers` |
| `usage` | whose usage, of what, in what unit | `turnTokenUsage` |
| `model` | the id, the row, or the config? | `modelId`, `modelPricing` |
| `response` | **still fails** — LLM response? `fetch` response? HTTP response? | `llmResponse`, `gatewayResponse`, `sseResponse` |

`response` is the instructive row: a full English word, and still ambiguous, because a codebase
that serves HTTP *and* calls an LLM *and* uses `fetch` has three of them. **Lengthening a name
is not the fix — disambiguating it is.** NEVER rename to a longer synonym carrying the same
ambiguity (`res` → `response`, `cfg` → `config` where several configs exist).

**"Which kind" is only the first axis. The second is WHOLE or PART, and it is missed more
often.** A value that is one fragment of a stream MUST say it is a fragment; a name that reads
as the complete thing while holding a piece is how a partial gets rendered, saved or billed as
if it were final:

| holds | a name that lies | a name that says it |
|---|---|---|
| one `text-delta` off the SSE stream | `text`, `content` | `textDelta`, `deltaText` |
| one chunk being replayed | `chunk` | `cosmeticChunk`, `replayChunk` |
| one SSE frame | `event`, `data` | `sseFrame` |
| everything accumulated so far | `text` | `assistantTextSoFar`, `accumulatedText` |
| the finished turn's text | `text` | `finalAssistantText` |

The last two matter together: **"so far" and "final" must be distinguishable at a glance**,
because every bug in a streaming path is one of them being used where the other belonged.

**A rename is behaviour-preserving ONLY if nothing reads the name as data.** Source-text guards,
AST passes keyed to an identifier, and census tests all break *silently*: an
`expect(source).not.toContain('writeSseData(res, …)')` becomes **vacuously true** the moment
`res` is renamed, and stays green while checking nothing. Before renaming, `grep` the tests for
the identifier **as a string literal** and treat every hit as part of the change.

Types carry the same duty — `MessageLoopContext` is a good name because it says *which* context.
This is *include units in variable names* one level up: `timeout_ms` beats `timeout` for exactly
the reason `sseResponse` beats `res`.

### Comments

- **Default to writing none.** Lean on names, types, and argument structure. A comment restating the next line (`// Initialize counter`) MUST be deleted
- **NEVER explain WHAT the code does** — rename the identifier, tighten the type, or extract a smaller function instead
- **ONLY when the WHY is non-obvious**: a hidden constraint, a subtle invariant, a workaround for a specific bug, a library quirk, behaviour that would surprise a reader. The *reason* must be in the comment — not just the rule — so a future maintainer can judge "is this still the right call?"
- **NEVER reference the current task, fix, or callers** (`// used by X`, `// see issue #123`) — that belongs in the PR description and rots as the codebase evolves
- One short line is the cap. Multi-paragraph docstrings only where an external contract requires them (public-API JSDoc on a published package)
- When refactoring, delete WHAT comments aggressively rather than keeping them "just in case" — the source of truth is the code
- **A comment corrected TWICE is one to delete, or to replace with a command.** Explaining a WHY invites stating a rule, and a rule inferred from one measurement is almost always narrower than the real one. One orion comment cost FOUR review rounds: each replacement was a fresh guess and each was wrong in a new way. What ended it was `leave X in and run <command>: the errors land on …` — **a command the reader can run cannot be too narrow.** Prefer naming the instrument over stating the law
- **NEVER correct a comment from a measurement that changed two things at once.** Two of those four wrong sentences came from a compound mutation with the result credited to whichever half the sentence was about. One change per probe, and read back WHAT was flagged rather than how many
- **Do not keep re-explaining.** Past one short line, the next sentence is usually defending the first. Explaining a correction, then explaining the correction to the correction, is how a comment outgrows the code it sits on — cut back to the shortest true statement instead of adding a qualifier

## TypeScript

- NEVER use `as` type casts; MUST use type guards instead (`const isXxx = (x: unknown): x is Type => { ... }`)
- MUST use existing utility functions from libraries (e.g. `isObject` from graphai) instead of writing your own
- MUST use `z.infer<typeof schema>` to derive types from Zod schemas; NEVER define duplicate local types
- MUST build strings with array + `push()` + `join()` and `const`, never `let` + `+=`
- MUST separate pure data transformation functions into their own files for reusability and testability ([`docs/testing.md`](docs/testing.md) → Designing for testability)
- MUST use descriptive format names ("object format" vs "text format"), never "new/legacy"
- MUST verify the correct API signatures for the TARGET version when migrating or upgrading packages — NEVER assume old APIs still work
- MUST use top-level `import` for npm packages — `await import()` only for conditional/optional dependencies that are not always loaded
- NEVER re-export modules unless there is a specific, justified reason
- A type-aware lint rule reporting something that cannot be true, or a dependency whose exports are silently `any`, is an ENVIRONMENT fault and not a code fault — the two causes, the one-command diagnosis for each, and why `yarn typecheck` cannot see either → [`docs/typescript-tooling.md`](docs/typescript-tooling.md)

## Vue.js

- MUST use Composition API (NEVER Options API)
- MUST use relative paths for imports (NEVER alias paths like `@/`)
- MUST use `emit` instead of passing functions as props
- SHOULD prefer `ref` over `reactive`
- NEVER use `v-html` (security risk)
- MUST use vue-i18n for text; NEVER hardcode strings in templates (use `$t()`)
- **`:key` is identity, NEVER position.** MUST NOT change a `v-for` key from a value to `index` — on reorder, insert or delete the key stays with the slot instead of the item, so Vue reuses the wrong element's DOM and typed-in values, focus and component state land on the neighbouring row. A type checker rejecting `string | null` is the usual way in; the fix is the **value** (`:key="timeItem ?? ''"`), never the position
- MUST read `git log -S':key=' -- <file>` before changing an existing key. Why the current key was chosen lives only in the history — a key written as a value is the trace of someone deciding position was not enough

## Styling

- MUST style components with **Tailwind utilities only** — NEVER write CSS. No `<style>` / `<style scoped>` block, no per-component `.css` file, no `<style src="...">` import
- MUST convert an existing `<style>` block to utilities when touching that component, rather than extending it
- Repeated utility runs MUST be extracted as a shared **component** (or a `class` string constant) — NEVER as a shared CSS class
- Dynamic / themed values MUST go through design tokens consumed by a utility (`bg-[var(--cell-bg)]`), NEVER a stylesheet rule
- If something genuinely cannot be a utility (`@keyframes`, `:deep()` into injected markup), MUST put it in the **Tailwind theme or one global stylesheet** with a one-line reason — NEVER in a component
- Why: shared CSS silently stops applying when a component's template has a **fragment root** — Vue gives the parent's scope id to a single root element only, so scoped rules match nothing and the element falls back to browser defaults (mulmoterminal #787). Utilities are global and have no such failure mode

## Web Design & Debugging

MUST prefer the dedicated skills over driving a browser by hand: `/verify` (exercise a change end-to-end and observe real behaviour — run before committing a nontrivial UI change), `run` (launch the app / take a screenshot), `/pr-ui-test` (UI regression check for a PR), `/web-perf` (performance investigation).

Falling back to the Playwright MCP by hand → [`docs/web-debugging.md`](docs/web-debugging.md).

## Documentation Maintenance

- MUST check README.md after changes and update it to reflect the correct specification — command examples, options, and usage instructions MUST be accurate
- MUST update CLAUDE.md / AGENTS.md if they contain relevant CLI documentation
- MUST check the repository's README.md and `docs/` when specs are added or changed, and include the updates in the same commit
- MUST generate proper web components (Vue/Astro) for web documentation — NEVER plain markdown files, unless markdown was explicitly asked for
- MUST VERIFY the actual implementation before writing API/tool documentation — NEVER guess API names or parameters

## Sharing Knowledge as Tech-Blog Articles

When an insight with **value beyond the current repository** emerges — a tool we picked, a workaround we discovered, a non-obvious gotcha (CI tooling, language/framework gotchas, security setups, integration patterns) — propose turning it into a short tech-blog article. Skip repo-specific bug fixes and refactors. ALWAYS confirm with the user before drafting, and pick a title together. Once the change has a merged PR, MUST use `/pr-to-tech-blog`.

## Replying in Japanese

**日本語で書いた文の途中に英単語を置かない。** 英語のまま残してよいのは **そのまま打ち込める文字列だけ** — 識別子（`chat.ts`、`onUpstream`）、コマンド（`yarn lint`）、PR / issue 番号、`async` のような言語キーワード、エラー文そのもの。それ以外は概念語なので日本語にする。

判定は一つだけ: **「これは打ち込める文字列か？」** 違うなら訳す。英語で返すときはこの制約は関係ない。

作業の語彙が特に漏れやすい。訳語を決めておく:

| 書かない | 書く |
|---|---|
| tranche | 一区切り / 今回の分 |
| ratchet | 据え置き一覧（減らす方向にしか動かない免除リスト） |
| bound | **上限**（「境界」は boundary であって上限ではない） |
| guard | 守り |
| census | 全体走査 |
| lift / extract | 持ち上げ / 切り出し |
| probe | 試し測り |
| control（実験の） | 対照 |
| suspension | 待ちが一つ増える |
| costed no / costed refusal | 理由を測った見送り |
| mutation sweep | コードを壊して回る |

**Why:** 半分だけ訳した文は、読む側が英語と日本語のどちらの語感で読むか決められず二度読むことになる。技術用語を避けろという話ではなく、一文の中で言語を切り替えるなという話（"ルー大柴みたいな会話はやめて"）。

## Continuous Learning

When learning something worth remembering, MUST first choose the destination:

- **Rules / workflows / coding standards** (apply to all future work) → this file, kept concise and actionable. MUST confirm with the user before adding
- **Depth behind a rule** — methodology, case studies, platform traps → a file under [`docs/`](docs/), linked from the rule here
- **A repeatable multi-step procedure** → a skill
- **Facts about the user, feedback/corrections, or project context** not derivable from code or git history → file-based memory (`~/.claude/.../memory/`, with a one-line pointer in `MEMORY.md`)

After completing a task (PR merge, command completion), MUST review the session: if the user gave corrections, redirections, or repeated instructions, evaluate whether they indicate a missing rule, a fact worth persisting, or a candidate for a new skill — and propose saving it.

## Automation Proposals

When the same instruction or pattern is given 2+ times in a session, MUST recognise the repetition and propose automation — CLAUDE.md for a rule, a **skill/command** for a parameterizable action, a **script** for a complex multi-step operation. MUST explain the trade-offs, let the user decide, and confirm it works as expected afterwards.
