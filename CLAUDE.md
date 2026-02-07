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

## Debugging Approach

- When a test or command fails, diagnose the ROOT CAUSE before attempting fixes
- Do NOT try quick-fix approaches (hardcoding values, JSON workarounds) — properly investigate the issue first
- Check git history/diffs when investigating regressions
- When user reports a failing command, understand what they're asking before jumping to debug

## Code Quality

After making code changes, always run:

1. `yarn format` - Format code with Prettier
2. `yarn lint` - Check for linting errors
3. `yarn build` - Verify build succeeds

## Documentation Maintenance

When implementing or modifying CLI commands or other user-facing features:

- **Always check README.md** after changes and update it to reflect the correct specification
- Ensure command examples, options, and usage instructions are accurate
- Update CLAUDE.md/AGENTS.md if they contain relevant CLI documentation
- When creating web documentation, generate proper web components (Vue/Astro) NOT plain markdown files, unless explicitly asked for markdown
- When documenting APIs or tools, VERIFY the actual implementation before writing — do not guess API names or parameters

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

### TypeScript ESM Configuration

For TypeScript projects, use ESM with `.js` extensions in imports:

**tsconfig.json** (key settings):
```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "declaration": true,
    "outDir": "./lib",
    "strict": true
  }
}
```

**package.json** (key settings):
```json
{
  "type": "module",
  "main": "lib/index.js",
  "types": "lib/index.d.ts",
  "exports": {
    ".": {
      "types": "./lib/index.d.ts",
      "default": "./lib/index.js"
    }
  }
}
```

**Import statements** - Always use `.js` extension (even for `.ts` files):
```typescript
// Correct
export * from "./actions/index.js";
import { foo } from "./utils/helper.js";

// Wrong - will fail at runtime
export * from "./actions/index";
import { foo } from "./utils/helper.ts";
```

This configuration ensures:
- TypeScript compiles to ESM
- Node.js resolves modules correctly at runtime
- Package consumers get proper type definitions

### Node/Browser Dual Export

When implementing library-like packages, always consider whether functions are **pure** (no Node.js-specific dependencies). Export browser-compatible code separately so it can run in both Node.js and browser environments.

**Design principles**:
- Pure functions (data transformation, validation, parsing) should be exported for browser use
- Code depending on Node.js APIs (`fs`, `path`, `child_process`, etc.) stays Node-only
- Separate entry points into `index.node.ts` and `index.browser.ts`

Extend the basic package.json above: change `main` to `lib/index.node.js` and add a `./browser` export:
```json
"exports": {
  ".":        { "types": "./lib/index.node.d.ts",    "default": "./lib/index.node.js" },
  "./browser": { "types": "./lib/index.browser.d.ts", "default": "./lib/index.browser.js" }
}
```

**File structure example**:
```
src/
  index.node.ts      # Node entry (re-exports everything)
  index.browser.ts   # Browser entry (re-exports pure functions only)
  utils/
    parser.ts         # Pure function - available from browser
    transform.ts      # Pure function - available from browser
  node/
    file_io.ts        # fs-dependent - Node only
    cli.ts            # Node only
```

Reference: `mulmocast-cli/package.json` exports pattern

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
7. **Create GitHub release with highlights**:
   - First, check merged PRs since last tag to identify key features:
     ```bash
     gh release view <previous-tag>  # See what PRs were included
     ```
   - Create release with highlights prepended to auto-generated notes:
     ```bash
     gh release create "1.0.0" --generate-notes --title "1.0.0" --notes "$(cat <<'EOF'
     ## ✨ Highlights

     - **Feature Name**: Brief description of the feature

     📖 **Sample**: [Example Link](https://github.com/org/repo/blob/main/path/to/sample.json)

     ---

     EOF
     )"
     ```
   - The `--notes` content is prepended to `--generate-notes` output
   - Include links to docs/samples when available
   - For code examples, use fenced code blocks in the notes

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
