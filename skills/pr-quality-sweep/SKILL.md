---
description: Sweep recent PRs in the current repo for unaddressed bot review comments (CodeRabbit + Sourcery) and refactoring needs, then fix them. Default window is 24 hours; accepts an override like "3d" / "72h" / "2 days".
---

## PR Quality Sweep

Audit pull requests updated within a recent time window, triage CodeRabbit findings, and fix everything that is a real issue. Intended to run daily.

### Invocation

The user may pass either a **time window** (sweep mode) or a
**PR number** (single-PR mode). When neither is provided, the
skill defaults to single-PR mode targeted at the PR for the
current branch — see "Default: no argument" below.

- `/pr-quality-sweep` → **single-PR** mode for the PR matching the current branch (or the last merge on the current branch if the head-PR lookup misses)
- `/pr-quality-sweep 805` / `https://github.com/<o>/<r>/pull/805` → single-PR mode for the explicit PR
- `/pr-quality-sweep 3d` / `3 days` / `72h` → sweep mode, 72 hours
- `/pr-quality-sweep 12h` → sweep mode, 12 hours
- `/pr-quality-sweep 24h` → sweep mode, 24 hours (the old default — pass it explicitly when you want it)

Argument parsing:

- Pure integer or PR URL → single-PR mode (skip step 2's listing, jump straight to step 3 with `PR_NUMBERS=<N>`).
- `Nd` / `Nh` / "N days" / "N hours" → sweep mode with that window. Convert to an absolute ISO cutoff using `date -u -v-{N}H +%Y-%m-%dT%H:%M:%SZ` (macOS) or `date -u -d "{N} hours ago" +%Y-%m-%dT%H:%M:%SZ` (Linux).
- Empty → single-PR mode via the inference helper.

### Default: no argument → infer PR from current directory

```bash
OWNER_REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
BRANCH=$(git rev-parse --abbrev-ref HEAD)

HEAD_PR=$(gh pr list --repo "$OWNER_REPO" --head "$BRANCH" --state open \
          --limit 1 --json number --jq '.[0].number')
if [ -z "$HEAD_PR" ]; then
  HEAD_PR=$(gh pr list --repo "$OWNER_REPO" --head "$BRANCH" --state all \
            --limit 1 --json number --jq '.[0].number')
fi

# When already on `main`, the head-PR lookup is empty; fall back
# to the last merge commit on this branch.
LAST_MERGE_PR=$(git log -1 --merges --pretty=%s | grep -oE '#[0-9]+' | head -1 | tr -d '#')

PR_N="${HEAD_PR:-$LAST_MERGE_PR}"

if [ -z "$PR_N" ]; then
  echo "No PR found for branch '$BRANCH' in $OWNER_REPO." >&2
  echo "Run with an explicit PR number (e.g. /pr-quality-sweep 805) or a time window (e.g. /pr-quality-sweep 24h)." >&2
  exit 1
fi
```

Surface the inferred PR number to the user before proceeding.

### Steps

#### 1. Orient

- Confirm current repo with `gh repo view --json nameWithOwner` — all `gh` calls use this implicit context.
- Run `date` to get today's date (never rely on internal knowledge).
- Compute the cutoff timestamp as above.

#### 2. List candidate PRs

```bash
gh pr list --state all --limit 60 --json number,title,state,createdAt,updatedAt,mergedAt,headRefName,baseRefName,author \
  --jq ".[] | select(.updatedAt > \"$CUTOFF\")"
```

Keep only PRs where `updatedAt >= CUTOFF`. Include both OPEN and MERGED / CLOSED — unaddressed comments on merged PRs still deserve follow-up fixes.

If the list is empty, report "No PRs in window" and exit.

#### 3. Collect bot review comments per PR

Two bots are wired up on this repo: **CodeRabbit** (`coderabbitai`) and **Sourcery** (`sourcery-ai`). They flag different classes of issue, so BOTH must be fetched. For each PR, fetch in parallel (one shell pipeline, `&` + `wait`):

```bash
mkdir -p /tmp/pr-quality-sweep
BOT_RE='coderabbit|sourcery'
for pr in $PR_NUMBERS; do
  gh api "repos/$OWNER_REPO/pulls/$pr/comments" --paginate \
    --jq "[.[] | select(.user.login | test(\"$BOT_RE\"; \"i\")) | {id, user: .user.login, path, line, url: .html_url, body: (.body | .[0:1500])}]" \
    > "/tmp/pr-quality-sweep/pr$pr-inline.json" &
  gh api "repos/$OWNER_REPO/pulls/$pr/reviews" --paginate \
    --jq "[.[] | select(.user.login | test(\"$BOT_RE\"; \"i\")) | {id, user: .user.login, state, submitted_at, body: (.body | .[0:2000])}]" \
    > "/tmp/pr-quality-sweep/pr$pr-reviews.json" &
done
wait
```

`pulls/.../comments` returns inline review comments (the most actionable ones — they have `path` and `line`). `pulls/.../reviews` returns top-level review summaries (Sourcery uses these heavily). **Both are needed**; `gh pr view --json comments` alone misses inline threads, and Sourcery's overall-feedback often lives in the review body, not as an inline comment.

#### 4. Filter out boilerplate

Skip:
- **CodeRabbit** walkthrough / summary sections: "Walkthrough", "Sequence Diagram", "Estimated code review effort", "Poem".
- **Sourcery** rate-limit messages: bodies like "Sorry @user, you have reached your weekly rate limit" mean the bot didn't actually review — note the PR as "not yet reviewed by Sourcery" and move on, don't treat it as a finding.
- Marketing / signup CTAs from either bot.

Only triage comments that describe a concrete code issue.

#### 5. Check each comment's current status

For each comment, decide one of:

- **Fixed** — the suggested change is reflected in the current HEAD of the PR branch (for open PRs) or the merged diff (for merged PRs). Verify by reading the file at the commented path / line and comparing to the suggestion.
- **Author-dismissed** — the PR author replied rejecting the suggestion, or the comment was marked outdated after a force-push.
- **Unaddressed** — a real, actionable issue that was not fixed.

For unaddressed items, assign severity:

- **blocker** — bug, security issue, type safety hole, data loss, crash
- **should-fix** — quality / maintainability / clear improvement with no downside
- **nit** — style, naming, micro-optimization, taste-dependent

#### 6. Also evaluate for refactoring needs

Separately from bot comments, scan the diff of each merged PR for things that humans usually catch but the bots might miss:

- Functions over the 20-line / complexity-15 thresholds (`CLAUDE.md` rule)
- Duplicated 3+ line patterns across files (DRY violation)
- Files ballooning past ~500 lines
- New `as` casts, `any`, `console.*` calls outside approved places
- Magic numbers, inline string literals that should be constants

Use `gh pr diff <n>` for merged PRs and read the files directly for open PRs.

#### 7. Report triage

Produce a single markdown report (stdout, not a file unless user asks). Structure:

```
## PR Quality Sweep ({WINDOW})

**Scope**: {N} PRs, {M} bot comments (CodeRabbit + Sourcery; {F} fixed, {D} dismissed, {U} unaddressed). Note any PRs where Sourcery hit its weekly rate limit so the user knows the review is incomplete.

### Blockers
- PR #X — `path/to/file.ts:42` — one-sentence summary → plan: <fix approach>

### Should-fix
- ...

### Nits
(list but don't fix unless user asks)

### Refactoring candidates
- PR #Y — extract `functionX` (32 lines, complexity 18) into smaller helpers
- ...
```

**STOP here and wait for user confirmation before fixing anything.** The report may be long; the user may want to narrow scope ("only blockers", "skip nits", "ignore PR #X, we already discussed it").

#### 8. Fix (after user approval)

For each item to fix:

**Open PRs** — push a new commit to the existing branch:
```bash
git checkout <head_ref>
git pull origin <head_ref>
# make the fix
git add <specific_files>        # NEVER `git add .` or `git add -A`
git commit -m "fix: address CodeRabbit review comment on <path>"
git push
```

**Merged PRs** — open a follow-up fix PR from `main`:
```bash
git checkout main && git pull
git checkout -b fix/cr-followup-pr<N>
# make the fix
git add <specific_files>
git commit -m "fix: address CodeRabbit comments from #<N> (<short topic>)"
git push -u origin fix/cr-followup-pr<N>
gh pr create --title "..." --body "Follow-up to #<N> addressing unaddressed CodeRabbit review comments.

## Items fixed
- file:line — summary
"
```

Batch related fixes into the **same follow-up PR** when they touch the same file or the same concept. One follow-up PR per original merged PR is usually the right granularity.

After each push, run `yarn format && yarn lint && yarn typecheck && yarn build && yarn test` (or whatever scripts the repo has) before the PR is opened. Fix failures before pushing.

#### 9. Summarize what was done

At the end, list:
- What was fixed and where (link to each new commit / PR)
- What was deliberately skipped and why (false positives, nits the user declined)
- Any blockers that need human input (architectural, requires domain knowledge, …)

### Important Rules

- NEVER use `git add .` or `git add -A`
- NEVER push to `main` directly — always a feature branch + PR
- NEVER force-push
- Confirm before merging any PR
- Use `--merge` (not squash) if merging
- If a CodeRabbit finding conflicts with a project-specific convention (see `CLAUDE.md`), prefer the convention and note it in the dismiss reason
- If unsure whether a finding is a real issue, surface it in the report rather than silently skipping
- When fixing fixes: one concern per commit, descriptive message (`fix: …`, `refactor: …`)

### Gotchas

- CodeRabbit sometimes repeats the same finding in both an inline comment and a review summary — dedupe by file+line+gist.
- "Nitpick" comments from CR are often hidden in collapsed `<details>` blocks inside the review body — the `body` field contains them, so grep for `Nitpick` / `nitpick`.
- Comments from a previous push may show `position: null` after a force-push — those are "outdated" and usually safe to skip unless the issue is clearly still present.
- Private repos require `gh auth status` to show an authenticated user; if not, abort with a clear message.
