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
- Run / verify / UI-test / profile a web app → `run`, `/verify`, `/pr-ui-test`, `/web-perf`
- Tech-blog article from a merged PR → `/pr-to-tech-blog`

## Environment & Packages

- When today's date is needed, MUST run the `date` command — NEVER rely on the model's internal knowledge
- MUST use **yarn** (`yarn`, `yarn add`, `yarn remove`); NEVER use npm commands
- MUST use `yarn add` instead of manually editing package.json
- During upgrade work, if a dependency turns out to be unused, MUST propose removing it (`yarn remove`) rather than upgrading it

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

- After pushing, MUST triage **every** bot reviewer, not just the first. Use `/gh-review-loop`; principles, and the "never wait for CodeRabbit / docs-only PRs don't wait at all" exceptions → [`docs/pr-bot-review.md`](docs/pr-bot-review.md)
- A PR addressing only PART of an issue MUST NOT use `Closes #N` — GitHub ignores the prose around the keyword. See [`docs/issue-filing.md`](docs/issue-filing.md)

## Change Scope Rules

- MUST only make changes that were explicitly requested — NEVER autonomously add features, tools, packages, or content
- MUST ask first if something additional seems needed
- MUST keep PR comments, commit messages, and documentation concise unless asked otherwise

## Code Quality

- MUST run after making code changes: `yarn format` → `yarn lint` → `yarn build` → `yarn typecheck`
  - If a `typecheck` script exists, MUST run it. Many repos split `build` (compile-only, tsconfig.build.json, often excludes `test/`) from `typecheck` (full project, includes tests). CI runs typecheck, so `build` passing proves nothing when a test file references a type you tightened — skipping this is the most common cause of "passed locally, failed in CI"
- MUST check for duplication against the existing codebase after a significant implementation, and refactor to eliminate it. Prioritise readability
- SHOULD run `/code-review` after a significant implementation (`--fix` to apply, `--comment` to post inline PR comments); `/simplify` for quality-only refactors, `/security-review` when the change has a security surface

### "This behaves the same" MUST be proved by running both, never by reasoning

Any change whose claim is behaviour preservation — an extraction, a rule lifted into a function, a regex rewritten as a scan, a condition "simplified", a deletion, a lint sweep — MUST be verified by copying the OLD code verbatim into a throwaway harness, running it beside the new code over **generated** inputs, comparing whole results, and stating the count. Unit tests on the new code pin what you *meant*; the risk is what you changed without noticing. Then delete the harness. **MUST use `/refactor-safely`** — it carries the method, the shapes these changes actually break, and the traps that make a broken change look green.

### A wide blast radius is verified by RUNNING THE APP, not by a green suite

A passing suite proves the code you thought about still behaves. It does not prove the app boots, the route is still wired, or the middleware still runs in the order the framework needs. When a change touches something EVERY request or EVERY caller passes through — a route entry point, a middleware, a handler signature, a shared function with many call sites — MUST start the stack and drive the real path: the happy path AND the rejection, varying what the change assumes. MUST name what you could NOT exercise locally and why; silence reads as "the run covered everything". Confirm the process you started is the checkout you changed — something already answering on the port is not evidence. Details → `/refactor-safely` → *What a green suite does not prove*.

## Debugging Approach

- MUST diagnose the ROOT CAUSE before attempting fixes; NEVER reach for a quick fix (hardcoded values, JSON workarounds)
- MUST understand what the user is asking before jumping to debug
- MUST check git history / diffs when investigating a regression
- MUST verify a fix against an **external ground truth**, never against another of your own outputs. Two things you produced agreeing proves only that they share your assumptions. Find the authority that already knows the answer — `tmux capture-pane` for a terminal screen, the server's own log for what was sent, the real file on disk — and diff against that (mulmoterminal #1073: "render with the fix" vs "without it" came out identical and shipped; both were diverging from the real screen, which `capture-pane` would have shown in one command)
- MUST vary the conditions the fix depends on before declaring it verified. One run at one size, one timing, one ordering tests a single point — and the bug lives in what you held constant. Name what the fix assumes and move each one
- Bug-family matrices, sweeping a rule across every call site, adversarially reviewing retry/replay, deterministic per-branch repro, and never trusting an error string as the only evidence → [`docs/debugging-methodology.md`](docs/debugging-methodology.md). MUST read before a non-trivial bug hunt

## Testing

- SHOULD use Node.js native `node:test` and `node:assert` by default; if the project already uses another runner (e.g. vitest, as in Cloudflare Workers projects), MUST follow the existing one
- MUST mock external APIs — tests MUST run without API keys
- MUST place tests in `test/` at the repo root, named `test_xxx.ts`; MUST add a `test` script to package.json and run it in CI
- Unit-test pattern checklist, golden tests, and the **designing-for-testability** rules → [`docs/testing.md`](docs/testing.md). MUST read before writing or refactoring tests
- Cross-platform CI (Linux/Windows/macOS matrix, `node:path` / `node:url` portability) → [`docs/cross-platform-ci.md`](docs/cross-platform-ci.md); Windows-only traps (`fs.watch`, `path.resolve`) → [`docs/windows-gotchas.md`](docs/windows-gotchas.md). MUST read before debugging a Windows failure

## Coding Style

**Write for human comprehension.** Human context and memory are limited: compact functions a reader can hold at a glance, names that tell a story, minimal variable scope, a flow that reads as a narrative.

- MUST keep functions under 20 lines; split into smaller functions if needed
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

### Comments

- **Default to writing none.** Lean on names, types, and argument structure. A comment restating the next line (`// Initialize counter`) MUST be deleted
- **NEVER explain WHAT the code does** — rename the identifier, tighten the type, or extract a smaller function instead
- **ONLY when the WHY is non-obvious**: a hidden constraint, a subtle invariant, a workaround for a specific bug, a library quirk, behaviour that would surprise a reader. The *reason* must be in the comment — not just the rule — so a future maintainer can judge "is this still the right call?"
- **NEVER reference the current task, fix, or callers** (`// used by X`, `// see issue #123`) — that belongs in the PR description and rots as the codebase evolves
- One short line is the cap. Multi-paragraph docstrings only where an external contract requires them (public-API JSDoc on a published package)
- When refactoring, delete WHAT comments aggressively rather than keeping them "just in case" — the source of truth is the code

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

## Vue.js

- MUST use Composition API (NEVER Options API)
- MUST use relative paths for imports (NEVER alias paths like `@/`)
- MUST use `emit` instead of passing functions as props
- SHOULD prefer `ref` over `reactive`
- NEVER use `v-html` (security risk)
- MUST use vue-i18n for text; NEVER hardcode strings in templates (use `$t()`)

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

## Continuous Learning

When learning something worth remembering, MUST first choose the destination:

- **Rules / workflows / coding standards** (apply to all future work) → this file, kept concise and actionable. MUST confirm with the user before adding
- **Depth behind a rule** — methodology, case studies, platform traps → a file under [`docs/`](docs/), linked from the rule here
- **A repeatable multi-step procedure** → a skill
- **Facts about the user, feedback/corrections, or project context** not derivable from code or git history → file-based memory (`~/.claude/.../memory/`, with a one-line pointer in `MEMORY.md`)

After completing a task (PR merge, command completion), MUST review the session: if the user gave corrections, redirections, or repeated instructions, evaluate whether they indicate a missing rule, a fact worth persisting, or a candidate for a new skill — and propose saving it.

## Automation Proposals

When the same instruction or pattern is given 2+ times in a session, MUST recognise the repetition and propose automation — CLAUDE.md for a rule, a **skill/command** for a parameterizable action, a **script** for a complex multi-step operation. MUST explain the trade-offs, let the user decide, and confirm it works as expected afterwards.
