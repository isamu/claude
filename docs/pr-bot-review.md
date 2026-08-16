# Triaging PR bot reviews

Referenced from CLAUDE.md → GitHub Issues & PRs. The `/gh-review-loop` skill runs this loop: it
reads what all the GitHub-side bots posted on the latest commit (including the inline threads that
`gh pr view` omits), applies real fixes, pushes, and waits for re-review. These are the principles
it enforces, and they MUST hold when triaging by hand too.

Triage **every** bot reviewer, not just the first one — CodeRabbit (`coderabbitai[bot]`), Sourcery
(`sourcery-ai[bot]`), Codex, plus project-specific reviewers like Socket Security.

- MUST NOT blindly apply suggestions — verify each against the actual codebase; bots disagree, so
  pick the right answer rather than satisfying both mechanically.
- Classify each comment: actionable fix (apply + add tests), valid nitpick (fix if cheap, else note
  as intentional), false positive / outdated (verify and skip with reason), rate-limited (note;
  re-check later).
- MUST commit fixes as `fix: address <bot-name> review comments` (name the specific bot), batched
  into one commit when possible.
- MUST post a follow-up PR comment summarizing what was addressed vs. deliberately skipped, so the
  human reviewer doesn't re-walk the bot threads.

## Don't block on bots — and for docs-only PRs, don't wait at all

- **NEVER wait for CodeRabbit.** It finishes long after the other checks, and its findings on an
  already-finished PR are advisory. Triage whatever it has already posted; never hold a merge for it
  to show up. This applies to every PR, docs or code.
- **A documentation-only PR MUST NOT wait for CI or for any bot** before merging. No source file
  changed, so lint / build / typecheck / tests have nothing to say that reading the diff did not.
  "Docs-only" means the diff touches **only** `docs/`, `*.md`, or images — anything under `src/` /
  `server/` / `common/` / `test/`, or `package.json`, or a workflow, is NOT docs-only however small.
- A bot comment that lands anyway and is **actually correct** SHOULD still be fixed — merging fast is
  not a reason to knowingly ship a false statement. Fix it, push, merge; do not re-wait.
- These two cases override "wait until every bot signs off and CI is green". Everywhere else the loop
  still applies.
