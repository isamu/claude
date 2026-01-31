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

- Always ask before running: `git commit`, `git push`, `git merge`, `git rebase`, etc.
- Read-only operations like `git status`, `git diff`, `git log` are OK
- NEVER use `git add .` or `git add <directory>` - always add files individually
- There may be work-in-progress files that should not be committed
- NEVER delete untracked files - they may be temporary work files or intentionally kept outside version control
- Commit message prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- NEVER use `git rebase`
- NEVER use `git push --force`
- Commit frequently with meaningful changes (small PRs are easier to review)

## Code Quality

After making code changes, always run:

1. `yarn format` - Format code with Prettier
2. `yarn lint` - Check for linting errors
3. `yarn build` - Verify build succeeds

## New Project Setup

When creating a new package.json, always include these scripts:

- `format` - Prettier
- `lint` - ESLint
- `build` - Build command

Node.js version: **24** or later
ESLint must use **flat config** (eslint.config.js), not legacy .eslintrc
Use **Tailwind CSS v4** (not v3) with `@tailwindcss/vite` plugin

ESLint rules:
- Indent: 2 spaces
- Quotes: double (`"`)
- Semicolons: required
- Line endings: Unix (LF)
- Unused variables: prefix with `__` to ignore

## Coding Style

- Keep functions under 20 lines; split into smaller functions if needed
- Prefer `const` over `let`; never use `var`
- Prefer functional approaches (`forEach`, `map`, `filter`, `reduce`) over `for` loops
- Prefer `async/await` over `.then()` chains
- Use explicit type definitions; avoid `any`
- No magic numbers; use named constants
- Include units in variable names when applicable (e.g., `timeout_ms`, `distance_km`)
- Follow DRY principle (Don't Repeat Yourself)

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
- For logic changes, write unit tests covering various patterns:
  - Normal cases (expected inputs)
  - Edge cases (boundary values, empty inputs)
  - Error cases (invalid/corrupted data, wrong types)
  - Null/undefined handling
- Test file organization:
  - Place tests in `test/` directory at repository root
  - File naming: `test_xxx.ts` (e.g., `test_parser.ts`, `test_utils.ts`)
  - For large repositories, split into subdirectories (e.g., `test/api/`, `test/utils/`)
- Add `test` script to package.json and run tests in CI

## npm Package Release

When releasing a new version of an npm package:

**IMPORTANT**: Confirm with user before each step.

### Steps

1. **Sync with remote**:
   ```bash
   git pull origin main
   ```
2. **Bump version** in package.json (patch/minor/major)
3. **Install and validate**:
   ```bash
   yarn install
   yarn typecheck  # if available
   yarn lint
   yarn build
   ```
4. **Publish to npm**:
   ```bash
   npm publish --access public
   ```
5. **Commit changes** (add files individually, NOT `git add -A`):
   ```bash
   git add package.json yarn.lock
   git commit -m "@package-name@version"
   git push origin main
   ```
6. **Create git tag** (NO "v" prefix):
   ```bash
   git tag "1.0.0"
   git push origin "1.0.0"
   ```
7. **Create GitHub release**:
   ```bash
   gh release create "1.0.0" --generate-notes --title "1.0.0"
   ```

### Important Rules

- Tag format: `1.0.0` (NOT `v1.0.0`)
- Commit message format: `@package-name@version` (e.g., `@gui-chat-plugin/todo@0.1.1`)
- Use `git pull origin main` for syncing (NEVER `git rebase`)
- Add files individually (NEVER `git add -A` or `git add .`)

### Pre-publish Checks

Before running `npm publish`, verify:

1. **No `file:` dependencies**: Check that package.json does not contain `"file:"` in dependencies/devDependencies (local path references must not be published). If found, **STOP and do not publish**.
2. **No debug code**: Search for and remove:
   - `console.log`, `console.debug` (except intentional logging)
   - `debugger` statements
3. **TODO/FIXME comments**: If found, ask the user for confirmation. Proceed with publish only if user approves.

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
