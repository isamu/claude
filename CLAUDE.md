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
