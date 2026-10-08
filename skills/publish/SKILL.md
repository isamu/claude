---
description: Publish an npm package with safety checks, tagging, and GitHub release. Use whenever the user asks to release/publish an npm package (also "mainでpublish & release", "publishしてrelease").
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

0. **Preflight**: confirm the current branch is `main`
   (`git branch --show-current`) and the working tree is clean
   (`git status`). If either fails, stop and ask the user.
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
4. **Commit and push the version bump** (add files individually, NOT `git add -A`).
   This push to main is the sanctioned release exception — get explicit
   user confirmation before pushing:
   ```bash
   git add package.json yarn.lock
   git commit -m "@package-name@version"
   git push origin main
   ```
5. **Create git tag**. Format: `@scope/name@X.Y.Z` when the package.json
   `name` is scoped, plain `X.Y.Z` otherwise. NEVER a `v` prefix (that's
   for app releases):
   ```bash
   git tag "@scope/name@1.0.0"   # scoped package
   # git tag "1.0.0"             # unscoped package
   git push origin "<tag>"
   ```
6. **Publish to npm** — 🔴 **hand this command to the user; do NOT run it yourself**:
   ```bash
   npm publish --access public --registry https://registry.npmjs.org/
   ```
   > **`npm publish` needs interactive authentication (OTP / 2FA) every time**, so it
   > cannot complete inside a tool call — it either hangs waiting for a prompt nobody
   > can answer, or fails. **Print the command, say which directory to run it in, and
   > wait for the user to report the result.** Continue to step 7 only after they
   > confirm it succeeded.

   > MUST pass `--registry https://registry.npmjs.org/`. The environment's
   > default registry may be a private mirror (e.g. an internal proxy), so
   > omitting it can publish to the wrong registry or fail auth. Always
   > pin the public registry explicitly on `npm publish`.

   > Commit + tag come BEFORE publish on purpose: `npm publish` is the
   > irreversible step, so the published tarball must correspond to a
   > tagged commit. If publish fails, fix and re-run it — the tag already
   > points at the right code. Never publish from an uncommitted tree.
7. **Create GitHub release** (notes in English):
   - Review **every** merged PR since the last tag — not just the headline
     ones — and write detailed notes covering all of them:
     ```bash
     git log <previous-tag>..HEAD --merges --oneline
     gh pr view <number>   # read each merged PR in the range for scope/rationale
     ```
   - Write all release notes in **English** (title, highlights, body),
     regardless of the repo's PR/commit language.
   - Decide the Latest flag by looking, not by assuming — see the rule below:
     ```bash
     # `^v[0-9]` and --limit 1000 are both load-bearing; see the rule below for why.
     gh release list --limit 1000 --json tagName -q '.[].tagName' | grep -qE '^v[0-9]' \
       && LATEST_FLAG=--latest=false || LATEST_FLAG=--latest
     ```
   - Create release with highlights prepended to auto-generated notes,
     using the tag from step 5 (scoped example shown; use plain `X.Y.Z`
     for unscoped packages):
     ```bash
     gh release create "@scope/name@1.0.0" "$LATEST_FLAG" --generate-notes --title "@scope/name@1.0.0" --notes "$(cat <<'EOF'
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
8. **Record in the changelog**: prepend an entry to `docs/ChangeLog.md`
   (create the file if it does not exist) — in English, newest-first,
   containing the version, the release date (from the `date` command, never
   guessed), and the same detailed per-PR summary as the GitHub release.
   Commit it individually: `git add docs/ChangeLog.md`.
9. **Post-release issue follow-up**:
   - From step 7's merge log, find issues the released PRs fixed
     (auto-close keywords in the PR bodies).
   - Comment on each with the released version and thanks to the reporter.
   - Close any that are resolved but still open.

### Monorepo / multi-package repos

When the repo publishes multiple packages
(「これに関連するパッケージを全部publishしてrelease」):

1. Determine the affected packages since each package's last release tag:
   `git log <pkg-last-tag>..HEAD -- <pkg-dir>`.
2. Run the full flow per affected package, in dependency order
   (dependencies before dependents).
3. Bump inter-package dependency ranges before publishing dependents.

### Important Rules

- Release notes are written in **English** and MUST cover every merged PR since the last tag in detail — not only headline features.
- Every release is also recorded in `docs/ChangeLog.md` (English, newest-first) with the same detail as the GitHub release.
- Tag format: `@scope/name@X.Y.Z` when the package.json `name` is scoped, plain `X.Y.Z` when not (NEVER `v1.0.0` — that's for app releases). The tag, release title, and commit message must all use the actual package name.
- Commit message format: `@package-name@version` (e.g., `@gui-chat-plugin/todo@0.1.1`)
- Use `git pull origin main` for syncing (NEVER `git rebase`)
- Add files individually (NEVER `git add -A` or `git add .`)
- **The Latest flag depends on whether the repo also ships an app release** — check, don't assume:
  ```bash
  gh release list --limit 1000 --json tagName -q '.[].tagName' | grep -qE '^v[0-9]' \
    && LATEST_FLAG=--latest=false || LATEST_FLAG=--latest
  ```
  - **An app release exists** → **MUST use `--latest=false`**. Only `/release-app` creates latest releases there, and a package release must not take Latest from `v0.2.0`.
  - **None anywhere** → package releases are the only releases, so **use `--latest`**. `--latest=false` has nothing to protect and instead strands Latest on whichever old version last claimed it: mulmoterminal sat on `4.1.0` while `4.1.1` and `4.2.0` shipped past it, so the front page advertised a version nobody was running.
  - When correcting a release published with the wrong flag: `gh release edit <tag> --latest`.

  **Both details in that command are load-bearing, and getting either wrong inverts the answer:**
  - **`^v[0-9]`, not `^v`.** A package whose name begins with `v` produces tags like graphai's `vanilla.2.0.9`. `^v` reads those as app releases and pins `--latest=false` on a repo that has no app release at all — graphai has 0 real matches and 25 false ones.
  - **`--limit 1000`, not the default or 100.** `gh release list` returns newest-first, so in a repo publishing many packages the app releases sit far down the list. mulmoclaude has 39 `v*` releases and none of them are in the newest 100 — at `--limit 100` the check answers "no app release" for **the one repo the rule exists to protect**, which is the exact failure it was written to prevent.

  Verified against four repos: mulmoterminal, mulmocast-cli and graphai → `--latest`; mulmoclaude → `--latest=false`.
- NEVER push feature work directly to main — always create a PR. Sole exception: this flow's version-bump commit + tag (steps 4-5), pushed to main only after explicit user confirmation.
