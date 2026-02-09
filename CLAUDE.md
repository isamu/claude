# Global Claude Code Settings

> Keywords: **MUST** / **NEVER** = mandatory. **SHOULD** = recommended unless there is a clear reason not to. **MAY** = optional.

## Package Manager

- MUST use **yarn** (`yarn`, `yarn add`, `yarn remove`)
- NEVER use npm commands
- MUST use `yarn add` instead of manually editing package.json

## Git Operations

- NEVER perform git commit, push, or other git operations without explicit user permission
- MUST **check current branch** with `git branch` or `git status` before making changes
  - If the branch is different from expected, MUST ask the user which branch to use
- MUST **create a feature branch** before starting implementation work
- MUST ask before running: `git commit`, `git push`, `git merge`, `git rebase`, etc.
- NEVER use `git add .` or `git add <directory>` — MUST add files individually
- NEVER delete untracked files
- NEVER push directly to main — MUST create a feature branch and open a PR
- MUST use merge commit (`--merge`) when merging PRs — NEVER use squash merge
- NEVER use `git rebase`
- NEVER use `git push --force`
- MUST use commit message prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- When asked to 'create a PR' or 'PR、マージ', this MUST be interpreted as CREATE a pull request, not merge it
- MUST confirm the correct default/target branch before creating PRs
- Read-only operations (`git status`, `git diff`, `git log`) MAY be run freely
- SHOULD commit after each meaningful change (e.g., schema done → commit, utility functions done → commit)

## Change Scope Rules

- MUST only make changes that were explicitly requested — NEVER autonomously add features, tools, packages, or content
- MUST ask first if something additional seems needed
- MUST keep PR comments, commit messages, and documentation concise unless asked otherwise

## Bug Fix / Feature Request Workflow

When asked to fix a bug or implement a new feature:

1. MUST discuss and clarify requirements with the user
2. MUST create a GitHub issue summarizing the task once scope is clear
3. MUST create a plan file under the repository's `plans/` directory (e.g., `plans/fix-xxx.md` or `plans/feat-xxx.md`)
4. MUST implement based on the plan
5. MUST include the user's original request in the PR description as a "User Prompt" section — clean up formatting but NEVER add content beyond the original wording. If there are multiple requests, list each as a bullet point

## Debugging Approach

- MUST diagnose the ROOT CAUSE before attempting fixes
- NEVER try quick-fix approaches (hardcoding values, JSON workarounds)
- MUST check git history/diffs when investigating regressions
- MUST understand what the user is asking before jumping to debug

## Code Quality

- MUST check for duplicate code with existing codebase after completing a significant implementation, and refactor to eliminate redundancy. Prioritize readability.
- MUST run after making code changes:
  1. `yarn format` - Format code with Prettier
  2. `yarn lint` - Check for linting errors
  3. `yarn build` - Verify build succeeds

## Documentation Maintenance

- MUST check README.md after changes and update it to reflect the correct specification
- MUST ensure command examples, options, and usage instructions are accurate
- MUST update CLAUDE.md/AGENTS.md if they contain relevant CLI documentation
- MUST check the repository's README.md and docs/ when specs are added or changed, update them accordingly, and include the updates in the same commit
- MUST generate proper web components (Vue/Astro) for web documentation — NEVER plain markdown files, unless explicitly asked for markdown
- MUST VERIFY the actual implementation before writing API/tool documentation — NEVER guess API names or parameters

## New Project Setup

When creating a new project, MUST use `/init-project`.

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
- NEVER use magic numbers; MUST use named constants
- SHOULD include units in variable names when applicable (e.g., `timeout_ms`, `distance_km`)
- MUST follow DRY principle (Don't Repeat Yourself)
- MUST add try/catch for operations that can fail
  - Network requests (fetch, API calls) MUST include timeout handling with AbortController
  - MUST provide meaningful error messages with context (URL, file path, etc.)

## Vue.js

- MUST use Composition API (NEVER Options API)
- MUST use relative paths for imports (NEVER alias paths like `@/`)
- MUST use `emit` instead of passing functions as props
- SHOULD prefer `ref` over `reactive`
- NEVER use `v-html` (security risk)
- MUST use vue-i18n for text; NEVER hardcode strings in templates (use `$t()`)

## Testing

- MUST use Node.js native `node:test` and `node:assert` for testing
- MUST mock external APIs (tests MUST run without API keys)
- For unit tests, MUST cover the following patterns:
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
- For CI integration tests (CLI execution, program output), MUST use **golden tests**:
  - Store the expected correct output (text, files) in git as golden files
  - Compare actual output against golden files in CI
  - Update golden files explicitly when output intentionally changes
- Test file organization:
  - MUST place tests in `test/` directory at repository root
  - File naming: `test_xxx.ts` (e.g., `test_parser.ts`, `test_utils.ts`)
  - For large repositories, SHOULD split into subdirectories (e.g., `test/api/`, `test/utils/`)
- MUST add `test` script to package.json and run tests in CI

### CI / Cross-Platform Compatibility

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

## npm Package Release

When releasing an npm package, MUST use `/publish`.

## Web Design with Playwright

When modifying web design (CSS, HTML, layouts):

1. MUST use `browser_navigate` to open the page
2. MUST use `browser_snapshot` to get DOM structure (preferred for understanding layout)
3. MUST use `browser_take_screenshot` to capture visual state
4. Make code changes
5. MUST refresh and verify with screenshot

Useful tools:
- `browser_resize` - Test responsive design at different breakpoints
- `browser_evaluate` - Inspect computed styles via JavaScript

## TypeScript Best Practices

- NEVER use `as` type casts; MUST use type guards instead (e.g., `const isXxx = (x: unknown): x is Type => { ... }`)
- MUST use existing utility functions from libraries (e.g., `isObject` from graphai) instead of writing your own
- MUST use `z.infer<typeof schema>` to derive types from Zod schemas; NEVER define duplicate local types
- MUST use array + `push()` + `join()` pattern for building strings with `const` instead of `let` + `+=`
- SHOULD separate pure data transformation functions into their own files for reusability and testability
- MUST use descriptive format names (e.g., "object format" vs "text format") instead of "new/legacy"
- MUST verify the correct API signatures for the TARGET version when migrating or upgrading packages — NEVER assume old APIs still work

## Continuous Learning

When learning something new during a session that should be remembered:
1. MUST confirm with the user before adding to this file
2. MUST add the learning to the appropriate section (or create a new section if needed)
3. MUST keep entries concise and actionable

After completing a task (PR merge, command completion, etc.), MUST review the session:
- If the user gave corrections, redirections, or repeated instructions during the session, MUST evaluate whether they indicate a missing rule in CLAUDE.md or a skill
- If so, MUST propose adding it to the appropriate location

## Automation Proposals

When the same instruction or pattern is given 2+ times in a session:
1. MUST recognize the repetition and propose automation
2. MUST choose the most appropriate method:
   - **CLAUDE.md**: For rules, guidelines, or workflows (e.g., release process, coding standards)
   - **Skill/Command**: For executable actions that can be parameterized (e.g., `/release`, `/deploy`)
   - **Script**: For complex multi-step operations that benefit from scripting
3. MUST explain the trade-offs and let the user decide
4. MUST confirm it works as expected after implementation
