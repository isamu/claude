---
description: GitHub PR review-driven loop. Read whatever the GitHub-side bots (Codex via GitHub Actions, CodeRabbit, Sourcery) posted on the latest commit, evaluate each finding, apply real fixes, push, wait for the bots to re-review the new commit, repeat. Merge once every bot signs off (Codex `LGTM`, CodeRabbit no actionable comments, Sourcery LGTM/skipped) AND CI is green AND the user confirms. Trigger shorthands the user actually types: "comments", "ci & comments", "ci失敗とcomments", "CIとコメント", "loopしてokになるまで" — all mean: infer the PR from the current branch, fetch new bot/CI feedback, triage, fix, loop until green.
---

# GitHub Review Loop

Reactive sibling of `codex-cross-review`. That skill drives a local
`codex exec`; this one reads what the **GitHub-side** review bots
already posted (via GitHub Actions, GitHub Apps), responds, and waits
for them to come back with the next iteration.

The contract assumes at least one bot posts a `CODEX VERDICT: LGTM`
or `CODEX VERDICT: CHANGES REQUESTED` line on every push. If a repo
has neither Codex auto-review nor CodeRabbit nor Sourcery wired up,
this skill has nothing to react to — use `codex-cross-review`
instead.

## Inputs

PR URL (`https://github.com/<owner>/<repo>/pull/<N>`) or number. If
only a number, infer owner/repo from `gh repo view --json
nameWithOwner`. With no arg, infer the PR for the current branch —
open-PR-first, since this loop targets open PRs:
`gh pr list --head <branch> --state open --limit 1 --json number`,
falling back to `--state merged` only if nothing is open.

## Setup (once)

1. `date -u +%Y-%m-%dT%H:%M:%SZ` — capture iteration start time.
2. `gh pr view <N> --json state,headRefName,baseRefName,mergeable,isDraft,statusCheckRollup,headRefOid`
   — confirm OPEN & not draft. Record `headRefOid` → `LAST_PUSHED_SHA`
   so the loop can tell when a new push has produced a fresh CI run.
3. `gh pr checkout <N>` — local working copy.
4. Record `LAST_KNOWN_MAIN = git rev-parse origin/main`.
5. Read repo `CLAUDE.md` if present — every fix has to honour it.
6. Confirm `gh auth status` is logged in. If not, stop and report.

## The Loop (max 5 iterations)

State file `/tmp/gh-review-loop-<N>/iteration-<k>.json` records
findings + decisions per iteration so the run is auditable.

### Iteration step A — wait for the bots to post their reviews

After every push, the bots run asynchronously. Don't fetch comments
prematurely — you'll see stale findings from the previous commit.

Wait for the Codex review workflow. Don't hardcode its name — it
varies per repo. Discover it once via `gh workflow list` and store
it in a variable:

```bash
# Discover the Codex review workflow name (once per run)
CODEX_WORKFLOW=$(gh workflow list --json name \
  --jq '.[] | select(.name | test("codex"; "i")) | .name' | head -1)

# Get the run ID for this PR's latest codex review job
RUN_ID=$(gh run list --workflow "$CODEX_WORKFLOW" --branch "$HEAD_REF" \
  --limit 1 --json databaseId,headSha --jq \
  ".[] | select(.headSha == \"$LAST_PUSHED_SHA\") | .databaseId")

# Block until it's done; cancel-in-progress means the latest one wins
until [ "$(gh run view "$RUN_ID" --json status --jq .status)" = "completed" ]; do
  sleep 12
done
```

CodeRabbit posts via GitHub App, not a workflow run — no exact
"is it done" signal. Use a 30-90 s grace after the Codex job
finishes; CR usually lands within that window. Sourcery is even
more variable but rate-limited frequently.

### Iteration step B — fetch this iteration's review

Filter by author + (created_at > iteration start):

```bash
mkdir -p /tmp/gh-review-loop-<N>
# `github-actions[bot]` posts all sorts of unrelated CI comments, so it
# only counts as a reviewer when the body carries the `CODEX VERDICT:`
# marker. If your repo's Codex workflow posts inline findings as
# github-actions WITHOUT the marker, widen the inline filter for that repo.
gh api "repos/$OWNER_REPO/issues/$N/comments" --paginate \
  --jq "[.[] | select((.user.login | test(\"codex|coderabbit|sourcery\"; \"i\")) \
         or ((.user.login | test(\"github-actions\"; \"i\")) and (.body | contains(\"CODEX VERDICT:\")))) \
       | select(.created_at > \"$ITER_START\") \
       | {user: .user.login, app: .performed_via_github_app.slug, created: .created_at, body}]" \
  > /tmp/gh-review-loop-<N>/top-<k>.json

gh api "repos/$OWNER_REPO/pulls/$N/comments" --paginate \
  --jq "[.[] | select((.user.login | test(\"codex|coderabbit|sourcery\"; \"i\")) \
         or ((.user.login | test(\"github-actions\"; \"i\")) and (.body | contains(\"CODEX VERDICT:\")))) \
       | select(.created_at > \"$ITER_START\") \
       | {user: .user.login, app: .performed_via_github_app.slug, created: .created_at, path, line, body}]" \
  > /tmp/gh-review-loop-<N>/inline-<k>.json

gh api "repos/$OWNER_REPO/pulls/$N/reviews" --paginate \
  --jq "[.[] | select(.user.login | test(\"coderabbit|sourcery\"; \"i\")) \
       | select(.submitted_at > \"$ITER_START\") \
       | {user: .user.login, state, submitted_at, body}]" \
  > /tmp/gh-review-loop-<N>/reviews-<k>.json
```

The verdict marker for the loop's machine signal: a top-level comment
that **starts** with `CODEX VERDICT: LGTM` or `CODEX VERDICT: CHANGES REQUESTED`.
Find the most-recent one in `top-<k>.json`.

Author identification trap: `codex` may post as either
- `github-actions[bot]` (when the workflow uses `GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}` → standard CI auth),
- the actual user via `chatgpt-codex-connector` GitHub App (when codex CLI uses its own auth via `~/.codex/auth.json`).

If you see a `chatgpt-codex-connector`-signed comment dated BEFORE
the latest push, it's probably from a manual `codex exec` session, not
the CI run. Don't treat it as the latest verdict.

### Iteration step C — triage and skip boilerplate

Drop:
- CodeRabbit walkthrough / sequence-diagram / poem / "Estimated effort" sections.
- CodeRabbit "rate limit exceeded" notices — treat as "no review this iteration".
- Sourcery rate-limit messages — same.
- Any bot's marketing/CTA section.

For each real finding, decide:
- **MUST-FIX** — bug, security, type hole, data loss, regression.
- **VALID-NIT** — small quality / readability win, no downside.
- **FALSE-POSITIVE** — bot misread the diff or repo conventions.
- **DEFER-TO-FOLLOWUP** — real but out of this PR's scope; record as
  a follow-up issue.

Apply MUST-FIX + VALID-NIT immediately. For FALSE-POSITIVE / DEFER,
post a short reply on the PR explaining why so the next iteration
doesn't re-flag.

**Don't trust the bots blindly.** Every finding must pass your own
review against the actual code. Bots are starting points, not
ceilings — note any issues you observe that the bots missed and fix
those too, attributed honestly in the commit message ("observed
during Claude review, not flagged by Codex").

### Iteration step D — local checks

After any code change, run the project's mandated checks (derived
from `CLAUDE.md` or conventional defaults):

- `yarn format` (Prettier)
- `yarn lint`
- `yarn typecheck`
- `yarn build`
- `yarn test` (skip e2e in-loop; CI catches e2e regressions)

If any check fails, fix and retry before pushing. Do not push a red
build into the loop — it confuses the bots' review thread.

### Iteration step E — sync main

```bash
git fetch origin main
NEW_MAIN=$(git rev-parse origin/main)
if [ "$NEW_MAIN" != "$LAST_KNOWN_MAIN" ]; then
  git merge origin/main --no-edit
  # Resolve conflicts: prefer your edits in files you touched this
  # iteration, but re-apply main-side logic. Pause for the user if
  # the conflict is semantic and unclear.
  LAST_KNOWN_MAIN=$NEW_MAIN
  # Re-run local checks after the merge — main may have changed contracts.
fi
```

### Iteration step F — commit + push

- `git add` only intentionally-touched files. Never `git add -A`.
- Commit message: `fix: address CR/Codex review iter-<k>` — list
  accepted findings in the body, attribute by reviewer. Use the
  environment's standard Co-Authored-By trailer for the current model.
- `git push` (no force).
- Capture the new HEAD SHA → `LAST_PUSHED_SHA` so step A of the
  next iteration knows which CI run to wait on.

### Iteration step G — decide whether to loop

Inspect this iteration's signals:

| Signal | Action |
|---|---|
| Codex `CODEX VERDICT: LGTM` + no other actionable bot comments + no issues YOU spotted | exit loop, proceed to merge |
| Codex `CODEX VERDICT: LGTM` + you found real issues | fix them anyway, push, next iteration (Codex will re-verify) |
| Codex `CODEX VERDICT: CHANGES REQUESTED` | next iteration |
| CodeRabbit posted MUST-FIX comments only | next iteration |
| No Codex run at all (workflow misconfigured / failed) | escalate to user — don't pretend the silence is approval |
| Iteration 5 reached | surface to user — non-converging review is human territory |

## CI monitoring (continuous, parallel to the loop)

Between every iteration AND continuously while waiting on CI:

```bash
gh pr checks <N> --json name,state,conclusion,link
```

CI failures are part of the loop, not separate from it:

1. Identify the failing job.
2. `gh run view <run-id> --log-failed` to read logs.
3. Reproduce locally if possible.
4. Fix → local checks → commit (`fix: CI <job-name> <reason>`) → push.
5. The push re-triggers Codex auto-review, which becomes the next iteration's input.

Do not ignore "unrelated" failures. A pre-existing main failure that
broke during an unrelated merge IS now your problem if it blocks
your branch's CI.

## Merge (once converged)

1. If the diff touches Vue/UI files, offer `/pr-ui-test` for a
   manual click-through before merging.
2. Verify in chat with the user: "Codex LGTM + my evaluation clear +
   all CI green. Ready to merge?"
3. On confirmation: `gh pr merge <N> --merge --delete-branch`
   (merge commit per project convention; NEVER squash unless the
   user overrides).
4. After merge: `git checkout main && git pull`, confirm
   `mergeCommit.oid`, report final state.
5. If `pr-merge-tidy` skill is available and the PR linked an issue
   or introduced a plan file, suggest running it as a follow-up.

## Safety rules (always)

- Never skip local checks. Never `--no-verify` / `--no-gpg-sign` / `--force`.
- Never apply a bot suggestion blindly. Every fix passes through
  your own review first.
- If you disagree with a bot, SAY SO in a reply comment with
  reasoning — silent disagreement breaks the loop's signal.
- Sync main before every push.
- Never commit secrets. Never include `.env` / credential files.
- If you find issues bots missed, attribute them honestly in the
  commit ("observed during Claude review, not flagged by Codex").
- Respect project-specific rules in `CLAUDE.md` — they override
  these defaults on conflict.
- If the pre-commit hook fails on an UNRELATED main-side breakage,
  fix it inline (smallest possible patch) and call it out clearly
  in the commit message. Don't bypass the hook.

## What "OK to merge" means from Claude's side

Not just "the bots stopped complaining." Claude's own OK requires:

- No MUST-FIX findings remaining (from any bot or from your own reading).
- No related issues in adjacent code that the diff invites but
  doesn't fix (deliberate deferrals must be posted as PR replies
  with rationale, not silent).
- All project checks pass locally.
- Test coverage matches the change shape — new logic has at least
  one test asserting the new behaviour.
- i18n / a11y / security conventions from `CLAUDE.md` are honoured.

Only then is it ready for the user merge confirmation.

## Reporting to the user

After every iteration, give a 3-4 line status:

- Iteration `<k>` of 5
- Bot signals this iteration: `Codex: LGTM/CHANGES (N)`, `CR: clean/N nits`, `Sourcery: LGTM/rate-limited`
- What you changed this iteration (1 line)
- CI status at this moment

Final report on merge: PR number, merge commit SHA, total iterations,
notable disagreements with bot findings if any.

## Novelty-diff review (on request)

When the user asks "review and tell me what CodeRabbit/bots missed"
(e.g. 「code rabbitのコメントにない指摘ある？」):

1. Run an independent review of the full PR diff — don't anchor on
   the bot threads.
2. Dedupe your findings against ALL bot comments (top-level, inline,
   review bodies).
3. Present only the novel findings in chat, ranked by severity.
4. Post them as PR comments only after the user confirms.

## When NOT to use this skill

- The repo doesn't have any review bots wired up → use
  `codex-cross-review` (drives codex CLI locally instead).
- The PR is already merged → use `pr-quality-sweep` to triage
  whatever was missed and open follow-up PRs.
- Many PRs need triage at once → use `pr-quality-sweep` for the
  scan, this skill for each individual PR you decide to chase.
