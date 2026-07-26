# Global Claude Code Settings

> Keywords: **MUST** / **NEVER** = mandatory. **SHOULD** = recommended unless there is a clear reason not to. **MAY** = optional.

## Package Manager

- MUST use **yarn** (`yarn`, `yarn add`, `yarn remove`)
- NEVER use npm commands
- MUST use `yarn add` instead of manually editing package.json
- During upgrade work, if a dependency turns out to be unused, MUST propose removing it (`yarn remove`) instead of upgrading it

## General

- When today's date is needed, MUST run the `date` command to get it — NEVER rely on model's internal knowledge

## Git Operations

- NEVER perform git commit, push, or other git operations without explicit user permission
- MUST **check current branch** with `git branch` or `git status` before making changes
  - If the branch is different from expected, MUST ask the user which branch to use
- MUST **create a feature branch** before starting implementation work
  - MUST `git fetch` first and check `git log HEAD..origin/<default-branch>` — branch only from an up-to-date base. A local default branch dozens of commits behind produces work built on files the mainline has since moved, renamed, or re-linted, and the whole implementation then has to be relocated. "The repo looked fine when I read it" is not evidence — the working copy can be stale.
- MUST ask before running: `git commit`, `git push`, `git merge`, `git rebase`, etc.
- NEVER use `git add .` or `git add <directory>` — MUST add files individually
- NEVER delete untracked files
- NEVER push directly to main — MUST create a feature branch and open a PR
- MUST use merge commit (`--merge`) when merging PRs — NEVER use squash merge
- NEVER use `git rebase`
- NEVER use `git push --force` (unconditional overwrite). `git push --force-with-lease` is permitted only on a feature branch you just pushed yourself, after `git commit --amend` or similar local rewrite — it aborts safely if anyone else pushed in the meantime. NEVER force-push (any variant) to `main` or shared branches.
- MUST use commit message prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- When asked to 'create a PR' or 'PR、マージ', this MUST be interpreted as CREATE a pull request, not merge it
- MUST confirm the correct default/target branch before creating PRs
- Read-only operations (`git status`, `git diff`, `git log`) MAY be run freely
- SHOULD commit after each meaningful change (e.g., schema done → commit, utility functions done → commit)

## PR Bot Review Handling

After pushing to a PR, MUST triage **every** bot reviewer — not just the first one (CodeRabbit `coderabbitai[bot]`, Sourcery `sourcery-ai[bot]`, Codex, plus project-specific reviewers like Socket Security). Use the `/gh-review-loop` skill: it reads what all the GitHub-side bots posted on the latest commit (including inline threads that `gh pr view` omits), applies real fixes, pushes, and waits for re-review until every bot signs off, CI is green, and the user confirms.

Core principles it enforces — and which MUST hold even when triaging by hand:
- MUST NOT blindly apply suggestions — verify each against the actual codebase; bots disagree, so pick the right answer rather than satisfying both mechanically.
- Classify each comment: actionable fix (apply + add tests), valid nitpick (fix if cheap, else note as intentional), false positive / outdated (verify and skip with reason), rate-limited (note; re-check later).
- MUST commit fixes as `fix: address <bot-name> review comments` (name the specific bot), batched into one commit when possible.
- MUST post a follow-up PR comment summarizing what was addressed vs. deliberately skipped, so the human reviewer doesn't re-walk the bot threads.

## Change Scope Rules

- MUST only make changes that were explicitly requested — NEVER autonomously add features, tools, packages, or content
- MUST ask first if something additional seems needed
- MUST keep PR comments, commit messages, and documentation concise unless asked otherwise

## Bug Fix / Feature Request Workflow

When asked to fix a bug or implement a new feature:

1. MUST discuss and clarify requirements with the user
2. MUST create a GitHub issue summarizing the task once scope is clear
3. MUST create a plan file under the repository's `plans/` directory (e.g., `plans/fix-xxx.md` or `plans/feat-xxx.md`) — this file MUST be committed to the repo
4. MUST implement based on the plan
5. MUST include the user's original request in the PR description as a "User Prompt" section — when the request spans multiple conversation turns, consolidate the user's messages into a coherent summary that preserves all of the user's intent without omitting details. Clean up formatting but NEVER add content beyond what the user said. If there are multiple distinct requests, list each as a bullet point
6. MUST include detailed implementation approach, proposed steps, and key decisions in the PR description — important information discussed in chat MUST be persisted as files or PR comments, NEVER only in chat
7. For AI-generated code PRs, MUST put a **Summary** section and an **Items to Confirm / Review** section at the **very top** of the PR description (before User Prompt, implementation details, etc.). The reviewer should see the summary and explicit review-focus points first, so they know what changed and what the author specifically wants a human to check (e.g., risky decisions, assumptions, unverified behaviors).

## Debugging Approach

- MUST diagnose the ROOT CAUSE before attempting fixes
- NEVER try quick-fix approaches (hardcoding values, JSON workarounds)
- MUST check git history/diffs when investigating regressions
- MUST understand what the user is asking before jumping to debug
- Deeper methodology — bug-family matrices, sweeping a rule across every call site (then extracting ONE shared helper), adversarially reviewing retry/replay mechanisms, deterministic per-branch repro, `git fetch` before judging another repo's state, and never trusting an error string as the only evidence → [`docs/debugging-methodology.md`](docs/debugging-methodology.md). Read before a non-trivial bug hunt.

## Code Quality

- MUST check for duplicate code with existing codebase after completing a significant implementation, and refactor to eliminate redundancy. Prioritize readability.
- SHOULD run `/code-review` after a significant implementation to catch bugs and cleanups (`--fix` to apply, `--comment` to post inline PR comments); use `/simplify` for quality-only refactors and `/security-review` when the change has a security surface
- MUST run after making code changes:
  1. `yarn format` - Format code with Prettier
  2. `yarn lint` - Check for linting errors
  3. `yarn build` - Verify build succeeds
  4. `yarn typecheck` - If the script is defined in package.json, MUST run it. Many repos split `yarn build` (compile-only, tsconfig.build.json, often excludes `test/`) from `yarn typecheck` (full project, tsconfig.json, includes tests). CI runs typecheck; `yarn build` can pass while typecheck fails when test files reference types you tightened. Skipping this step is the most common cause of "passed locally, failed in CI."

## Documentation Maintenance

- MUST check README.md after changes and update it to reflect the correct specification
- MUST ensure command examples, options, and usage instructions are accurate
- MUST update CLAUDE.md/AGENTS.md if they contain relevant CLI documentation
- MUST check the repository's README.md and docs/ when specs are added or changed, update them accordingly, and include the updates in the same commit
- MUST generate proper web components (Vue/Astro) for web documentation — NEVER plain markdown files, unless explicitly asked for markdown
- MUST VERIFY the actual implementation before writing API/tool documentation — NEVER guess API names or parameters

## Skills

Prefer these skills over doing the work by hand:

- **New project** → `/init-project`
- **Publish an npm package** → `/publish`
- **Release the MulmoClaude app** (GitHub release, not npm) → `/release-app`
- **PR bot review triage** → `/gh-review-loop` (see PR Bot Review Handling)
- **Code review / refactor / security** → `/code-review`, `/simplify`, `/security-review` (see Code Quality)
- **Web verify / run / UI test / perf** → `/verify`, `run`, `/pr-ui-test`, `/web-perf` (see Web Design & Debugging)
- **Tech-blog article from a merged PR** → `/pr-to-tech-blog` (see Sharing Knowledge)

## Import Style

- MUST use top-level `import` for npm packages — NEVER use `await import()` for packages that are always needed
- Dynamic `import()` MAY only be used for conditional/optional dependencies that are not always loaded

- NEVER re-export modules unless there is a specific, justified reason

## Coding Style

### Philosophy: Code for Human Comprehension

Human context and memory are limited. MUST write code with this in mind:

- **Compact functions**: Split into small, focused functions that humans can fully comprehend at a glance
- **Clear naming**: Function and variable names MUST tell a story; a beginner should understand the flow
- **Minimal scope**: Keep variable scope as small as possible to reduce cognitive load
- **Readable flow**: Code MUST read like a narrative

### Rules

- MUST keep functions under 20 lines; split into smaller functions if needed
- MUST prefer `const` over `let`; NEVER use `var`
- MUST prefer functional approaches (`forEach`, `map`, `filter`, `reduce`) over `for` loops
- MUST prefer `async/await` over `.then()` chains
- MUST use explicit type definitions; NEVER use `any`
- NEVER silence lint/type errors with `eslint-disable`, `@ts-ignore`, or `@ts-expect-error` — fix the types / root cause instead (define proper type files if needed)
- NEVER use magic numbers; MUST use named constants
- SHOULD include units in variable names when applicable (e.g., `timeout_ms`, `distance_km`)
- MUST follow DRY principle (Don't Repeat Yourself)
- MUST add try/catch for operations that can fail
  - Network requests (fetch, API calls) MUST include timeout handling with AbortController
  - MUST provide meaningful error messages with context (URL, file path, etc.)

### Comments

- **Default to writing no comments.** Lean on names, types, and argument structure to do the explaining. If a comment restates what the next line obviously does (e.g. `// Initialize counter` followed by `let counter = 0;`), delete it.
- **NEVER explain WHAT the code does** — well-named functions, variables, and types already do that. If a comment is needed to understand WHAT, the better fix is to rename the identifier, tighten the type, or extract a smaller function.
- **ONLY add a comment when the WHY is non-obvious**: a hidden constraint, a subtle invariant, a workaround for a specific bug, a browser / library quirk, behavior that would surprise a reader. A future maintainer should be able to look at the comment and judge "is this still the right call?" — which means the *reason* must be in the comment, not just the *rule*.
- **NEVER reference the current task, fix, or callers** in comments (`// used by X`, `// added for the Y flow`, `// see issue #123`) — that context belongs in the PR description / commit message, and rots as the codebase evolves.
- **Don't write multi-paragraph docstrings or multi-line comment blocks** unless absolutely required by an external contract (public-API JSDoc on a published package). One short line is the cap.
- When refactoring, **delete WHAT comments aggressively** rather than keeping them around "just in case" — the source of truth is the code.

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

## Testing

- SHOULD use Node.js native `node:test` and `node:assert` by default; if the project already uses another runner (e.g. vitest, as in Cloudflare Workers projects), MUST follow the existing one
- MUST mock external APIs (tests MUST run without API keys)
- MUST place tests in `test/` at the repo root, named `test_xxx.ts`; MUST add a `test` script to package.json and run it in CI
- Full unit-test pattern checklist (happy/edge/corner/boundary/empty/null/invalid/error/negative/regression), golden tests, and the **designing-for-testability** rules → [`docs/testing.md`](docs/testing.md). Read before writing or refactoring tests.
- Cross-platform CI (Linux/Windows/macOS matrix, `node:path` / `node:url` portability) → [`docs/cross-platform-ci.md`](docs/cross-platform-ci.md); Windows-only traps (`fs.watch`, `path.resolve`) → [`docs/windows-gotchas.md`](docs/windows-gotchas.md) — MUST read before debugging a Windows failure.

## Web Design & Debugging

MUST prefer the dedicated skills over driving a browser by hand:

- `/verify` — exercise a change end-to-end and observe real behavior (run before committing nontrivial UI changes)
- `run` — launch and drive the project's app to see a change working / take a screenshot
- `/pr-ui-test` — UI regression check for a PR
- `/web-perf` — web performance investigation

Falling back to the Playwright MCP by hand (web-design steps, debugging a live flow, the `browser_*` tool list) → [`docs/web-debugging.md`](docs/web-debugging.md).

## TypeScript Best Practices

- NEVER use `as` type casts; MUST use type guards instead (e.g., `const isXxx = (x: unknown): x is Type => { ... }`)
- MUST use existing utility functions from libraries (e.g., `isObject` from graphai) instead of writing your own
- MUST use `z.infer<typeof schema>` to derive types from Zod schemas; NEVER define duplicate local types
- MUST use array + `push()` + `join()` pattern for building strings with `const` instead of `let` + `+=`
- MUST separate pure data transformation functions into their own files for reusability and testability (see [`docs/testing.md`](docs/testing.md) → Designing for testability)
- MUST use descriptive format names (e.g., "object format" vs "text format") instead of "new/legacy"
- MUST verify the correct API signatures for the TARGET version when migrating or upgrading packages — NEVER assume old APIs still work

## Sharing Knowledge as Tech-Blog Articles

During development, when a generally-shareable insight emerges — a tool we picked, a workaround we discovered, a non-obvious gotcha — propose turning it into a short tech-blog article.

- Trigger: insights with **value beyond the current repository** (e.g. CI tooling choices, language / framework gotchas, security setups, integration patterns). Skip repo-specific bug fixes, refactors, or anything that only makes sense with full project context.
- When you have the merged PR for the change, MUST use the `/pr-to-tech-blog` skill — it knows the destination directory, frontmatter shape, and house style.
- ALWAYS confirm with the user before drafting, and pick a title together if it isn't obvious.

## Continuous Learning

When learning something worth remembering, MUST first choose the right destination:
- **Rules / workflows / coding standards** (apply to all future work) → this file (CLAUDE.md)
- **Facts about the user, feedback/corrections, or project context** (not derivable from code or git history) → file-based memory (`~/.claude/.../memory/`, with a one-line pointer added to `MEMORY.md`)

When adding to this file:
1. MUST confirm with the user before adding
2. MUST add the learning to the appropriate section (or create a new section if needed)
3. MUST keep entries concise and actionable

After completing a task (PR merge, command completion, etc.), MUST review the session:
- If the user gave corrections, redirections, or repeated instructions during the session, MUST evaluate whether they indicate a missing rule (→ CLAUDE.md), a fact/preference worth persisting (→ memory), or a candidate for a new skill
- If so, MUST propose saving it to the appropriate location

## Automation Proposals

When the same instruction or pattern is given 2+ times in a session:
1. MUST recognize the repetition and propose automation
2. MUST choose the most appropriate method:
   - **CLAUDE.md**: For rules, guidelines, or workflows (e.g., release process, coding standards)
   - **Skill/Command**: For executable actions that can be parameterized (e.g., `/release`, `/deploy`)
   - **Script**: For complex multi-step operations that benefit from scripting
3. MUST explain the trade-offs and let the user decide
4. MUST confirm it works as expected after implementation
