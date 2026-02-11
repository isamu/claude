---
description: Publish an npm package with safety checks, tagging, and GitHub release
---

## npm Package Release

**IMPORTANT**: Confirm with user before each step.

### Pre-publish Checks

Before running `npm publish`, verify:

1. **No `file:` dependencies**: Check that package.json does not contain `"file:"` in dependencies/devDependencies (local path references must not be published). If found, **STOP and do not publish**.
2. **No debug code**: Search for and remove:
   - `console.log`, `console.debug` (except intentional logging)
   - `debugger` statements
3. **TODO/FIXME comments**: If found, ask the user for confirmation. Proceed with publish only if user approves.

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
     ## Highlights

     - **Feature Name**: Brief description of the feature

     **Sample**: [Example Link](https://github.com/org/repo/blob/main/path/to/sample.json)

     📦 **npm**: [`package-name@1.0.0`](https://www.npmjs.com/package/package-name/v/1.0.0)

     ---

     EOF
     )"
     ```
   - The `--notes` content is prepended to `--generate-notes` output
   - Include links to docs/samples when available
   - For code examples, use fenced code blocks in the notes
   - MUST include the npm package link using the format: `📦 **npm**: [\`package-name@version\`](https://www.npmjs.com/package/package-name/v/version)`
   - Read the package name from package.json `name` field (handle scoped packages like `@scope/name` — the npm URL for scoped packages is `https://www.npmjs.com/package/@scope/name/v/version`)

### Important Rules

- Tag format: `1.0.0` (NOT `v1.0.0`)
- Commit message format: `@package-name@version` (e.g., `@gui-chat-plugin/todo@0.1.1`)
- Use `git pull origin main` for syncing (NEVER `git rebase`)
- Add files individually (NEVER `git add -A` or `git add .`)
