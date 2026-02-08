# Global Claude Code Settings

## Package Manager

Use **yarn** instead of npm.

- `yarn` for installing dependencies
- `yarn add <package>` for adding packages
- `yarn remove <package>` for removing packages
- Do NOT use npm commands
- When adding packages, use `yarn add` instead of manually editing package.json
- This ensures the latest version is installed automatically

## Git Operations

NEVER perform git commit, push, or other git operations without explicit user permission.

- **ALWAYS check current branch** before making changes with `git branch` or `git status`
  - If the branch is different from expected, ask the user which branch to use
  - User may be working on multiple branches in parallel
- **Create a feature branch** before starting implementation work
  - Commit after each meaningful change (e.g., schema done → commit, utility functions done → commit)
  - This makes PRs easier to review and enables incremental progress tracking
- Always ask before running: `git commit`, `git push`, `git merge`, `git rebase`, etc.
- Read-only operations like `git status`, `git diff`, `git log` are OK
- NEVER use `git add .` or `git add <directory>` - always add files individually
- There may be work-in-progress files that should not be committed
- NEVER delete untracked files - they may be temporary work files or intentionally kept outside version control
- Commit message prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- NEVER push directly to main. Always create a feature branch and open a PR
- When asked to 'create a PR' or 'PR、マージ', this means CREATE a pull request, not merge it
- Always confirm the correct default/target branch before creating PRs
- When merging PRs, use merge commit (`--merge`). Do NOT use squash merge
- NEVER use `git rebase`
- NEVER use `git push --force`

## Change Scope Rules

- Only make changes that were explicitly requested. Do NOT autonomously add features, tools, packages, or content
- If you think something additional should be added, ASK first
- Do not over-elaborate PR comments, commit messages, or documentation — keep them concise unless asked otherwise

## Bug Fix / Feature Request Workflow

When asked to fix a bug or implement a new feature:

1. Discuss and clarify requirements with the user
2. Once the scope is clear, create a GitHub issue summarizing the task
3. Create a plan file under the repository's `plans/` directory (e.g., `plans/fix-xxx.md` or `plans/feat-xxx.md`)
4. Implement based on the plan
5. When creating the PR, include the user's original request in the PR description as a "User Prompt" section — clean up formatting but preserve the original wording without adding content. If there are multiple requests, list each as a bullet point

## Debugging Approach

- When a test or command fails, diagnose the ROOT CAUSE before attempting fixes
- Do NOT try quick-fix approaches (hardcoding values, JSON workarounds) — properly investigate the issue first
- Check git history/diffs when investigating regressions
- When user reports a failing command, understand what they're asking before jumping to debug

## Code Quality

After completing a significant implementation, check for duplicate code with existing codebase and refactor to eliminate redundancy. Prioritize readability.

After making code changes, always run:

1. `yarn format` - Format code with Prettier
2. `yarn lint` - Check for linting errors
3. `yarn build` - Verify build succeeds

## Documentation Maintenance

When implementing or modifying CLI commands or other user-facing features:

- **Always check README.md** after changes and update it to reflect the correct specification
- Ensure command examples, options, and usage instructions are accurate
- Update CLAUDE.md/AGENTS.md if they contain relevant CLI documentation
- When specs are added or changed, always check the repository's README.md and docs/ for affected documentation, update them accordingly, and include the updates in the same commit
- When creating web documentation, generate proper web components (Vue/Astro) NOT plain markdown files, unless explicitly asked for markdown
- When documenting APIs or tools, VERIFY the actual implementation before writing — do not guess API names or parameters

## New Project Setup

When creating a new project, use `/init-project`.

## Coding Style

### Philosophy: Code for Human Comprehension

Human context and memory are limited. Always write code with this in mind:

- **Compact functions**: Split into small, focused functions that humans can fully comprehend at a glance
- **Clear naming**: Function and variable names should tell a story; a beginner should understand the flow
- **Minimal scope**: Keep variable scope as small as possible to reduce cognitive load
- **Readable flow**: Code should read like a narrative - the sequence of function calls and variable assignments should make the intent obvious

### Rules

- Keep functions under 20 lines; split into smaller functions if needed
- Prefer `const` over `let`; never use `var`
- Prefer functional approaches (`forEach`, `map`, `filter`, `reduce`) over `for` loops
- Prefer `async/await` over `.then()` chains
- Use explicit type definitions; avoid `any`
- No magic numbers; use named constants
- Include units in variable names when applicable (e.g., `timeout_ms`, `distance_km`)
- Follow DRY principle (Don't Repeat Yourself)
- **Error handling**: Always add try/catch for operations that can fail
  - Network requests (fetch, API calls) must include timeout handling with AbortController
  - Provide meaningful error messages with context (URL, file path, etc.)

## Vue.js

- Use Composition API (not Options API)
- Always use relative paths for imports (not alias paths like `@/`)
- Use `emit` instead of passing functions as props
- Prefer `ref` over `reactive`
- Never use `v-html` (security risk)
- Use vue-i18n for text; never hardcode strings in templates (use `$t()`)

## Testing

- Use Node.js native `node:test` and `node:assert` for testing
- Mock external APIs (tests should run without API keys)
- For unit tests, cover the following patterns:
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
- For CI integration tests (CLI execution, program output), use **golden tests**:
  - Store the expected correct output (text, files) in git as golden files
  - Compare actual output against golden files in CI
  - Update golden files explicitly when output intentionally changes
- Test file organization:
  - Place tests in `test/` directory at repository root
  - File naming: `test_xxx.ts` (e.g., `test_parser.ts`, `test_utils.ts`)
  - For large repositories, split into subdirectories (e.g., `test/api/`, `test/utils/`)
- Add `test` script to package.json and run tests in CI

### CI / Cross-Platform Compatibility

CI should work on **Linux, Windows, and macOS** whenever possible.

- Use `node:path` with `path.join()` / `path.resolve()` instead of hardcoded `/` or `\\` separators
- Use `node:url` (`fileURLToPath`, `pathToFileURL`) for file URL conversions
- Avoid shell-specific syntax in npm scripts (no `rm -rf`, `cp`, `&&` chaining that differs across shells); use cross-platform alternatives:
  - `rimraf` instead of `rm -rf`
  - `shx` or `cpy-cli` instead of `cp` / `mv`
  - Or use Node.js scripts for complex build steps
- Do not rely on case-sensitive file systems (macOS/Windows are case-insensitive by default)
- Use `node:os` for platform-specific logic when unavoidable
- In GitHub Actions, include all three runners in the matrix:
  ```yaml
  strategy:
    matrix:
      os: [ubuntu-latest, windows-latest, macos-latest]
  ```

## npm Package Release

When releasing an npm package, use `/publish`.

## Web Design with Playwright

When modifying web design (CSS, HTML, layouts):

1. Use `browser_navigate` to open the page
2. Use `browser_snapshot` to get DOM structure (preferred for understanding layout)
3. Use `browser_take_screenshot` to capture visual state
4. Make code changes
5. Refresh and verify with screenshot

Useful tools:
- `browser_resize` - Test responsive design at different breakpoints
- `browser_evaluate` - Inspect computed styles via JavaScript

## TypeScript Best Practices

- Avoid `as` type casts; use type guards instead (e.g., `const isXxx = (x: unknown): x is Type => { ... }`)
- Use existing utility functions from libraries (e.g., `isObject` from graphai) instead of writing your own
- Use `z.infer<typeof schema>` to derive types from Zod schemas; don't define duplicate local types
- When building strings with `const`, use array + `push()` + `join()` pattern instead of `let` + `+=`
- Separate pure data transformation functions into their own files for reusability and testability
- Use descriptive format names (e.g., "object format" vs "text format") instead of "new/legacy"
- When migrating or upgrading packages, verify the correct API signatures for the TARGET version — do not assume old APIs still work

## Continuous Learning

When learning something new during a session that should be remembered:
1. Confirm with the user before adding to this file
2. Add the learning to the appropriate section (or create a new section if needed)
3. Keep entries concise and actionable

## Automation Proposals

When the same instruction or pattern is given 2+ times in a session:
1. Recognize the repetition and propose automation
2. Choose the most appropriate method:
   - **CLAUDE.md**: For rules, guidelines, or workflows (e.g., release process, coding standards)
   - **Skill/Command**: For executable actions that can be parameterized (e.g., `/release`, `/deploy`)
   - **Script**: For complex multi-step operations that benefit from scripting
3. Explain the trade-offs and let the user decide
4. After implementation, confirm it works as expected
