---
description: Dual-reviewer loop on a GitHub PR. User pastes a PR URL or number; you invoke Codex to review and post findings, then YOU critically evaluate each finding (necessity, side effects, missing related issues), apply valid fixes, re-request Codex, and repeat until both reviewers agree. Throughout the loop you monitor CI, sync main, and resolve conflicts. Merge only when both reviewers are OK and CI is green.
---

# Codex Cross-Review

Two-reviewer convergence loop. Codex raises findings; you (Claude Code) evaluate them with full context, apply real fixes, and request a follow-up review. Both reviewers must sign off before merge.

## Inputs

The user supplies a PR URL (`https://github.com/<owner>/<repo>/pull/<N>`) or just a number in the current repo context. Parse out `<owner>/<repo>` and `<N>`. If only a number, infer owner/repo from `gh repo view --json nameWithOwner`.

## Setup (once)

1. `date` — capture current ISO time so you can filter "new" comments per iteration.
2. `gh pr view <N> --json state,headRefName,baseRefName,mergeable,isDraft,statusCheckRollup` — confirm OPEN & not draft. If not, stop and report.
3. `gh pr checkout <N>` — check out the branch locally.
4. Resolve the default branch: `DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)`. Then record `BASE_SHA = git merge-base "origin/$DEFAULT_BRANCH" HEAD` so you can detect new default-branch commits during the loop.
5. Read `CLAUDE.md` in the repo root (if present) for project-specific rules — they apply to every fix you commit.
6. Ensure `codex` CLI is installed (`command -v codex`). If missing, stop and tell the user to install `@openai/codex`.

## The Loop (ends on two consecutive clean rounds — see step G for the cap)

Keep a per-iteration state file at `/tmp/codex-cross-review-<N>/iteration-<k>.json` summarising what Codex said, what you decided, and what you changed. Helps post-mortem if the loop spins.

### Iteration step A — request Codex review

Run Codex non-interactively with an explicit verdict contract:

```bash
codex exec --sandbox workspace-write \
  "Review PR #<N> at https://github.com/<owner>/<repo>/pull/<N>.

   Use the gh CLI to inspect the diff, then post inline review comments on specific lines via:
     gh api repos/<owner>/<repo>/pulls/<N>/comments
   or top-level comments via:
     gh pr comment <N>

   Focus on: correctness, edge cases, security (XSS / SSRF / path traversal / prompt injection),
   accessibility, i18n lockstep (if this repo has multi-locale dicts), tests coverage for the
   happy path + boundary cases, and consistency with the rest of the codebase.

   At the END of your work, ALWAYS post ONE final top-level comment that starts with a
   verdict marker on its own line:
     - 'CODEX VERDICT: LGTM' if you have no outstanding concerns
     - 'CODEX VERDICT: CHANGES REQUESTED' followed by a bulleted summary of remaining issues

   Do not apply any fixes yourself — only review and post findings."
```

Wait for `codex exec` to finish. If it errors out, record and stop the loop (ask user to rerun manually).

Then post the exchange to the PR — see *Every Codex exchange goes on the PR* below. Do it now, not at the end of the loop: an exchange you did not post while you had it is one you will reconstruct from memory later.

### Iteration step B — fetch what Codex posted *this iteration*

Filter by author and the timestamp captured at the start of the iteration:

```bash
gh api "repos/<owner>/<repo>/pulls/<N>/comments" --paginate \
  --jq "[.[] | select(.user.login | test(\"codex\"; \"i\")) | select(.created_at > \"$ITER_START\")]" \
  > /tmp/codex-cross-review-<N>/inline-<k>.json

gh api "repos/<owner>/<repo>/issues/<N>/comments" --paginate \
  --jq "[.[] | select(.user.login | test(\"codex\"; \"i\")) | select(.created_at > \"$ITER_START\")]" \
  > /tmp/codex-cross-review-<N>/top-<k>.json
```

Locate the verdict marker in the top-level comments. The line starting `CODEX VERDICT:` is the machine-readable signal.

### Every Codex exchange goes on the PR

Codex posts its own findings, so it is easy to assume the record is already there. It is not. **Your half of the conversation is invisible** — the prompt you sent, the constraints you put on it, the follow-up questions, and the answers you got back when you pushed back on a finding. What lands on the PR is a verdict with no record of what was asked to produce it, and the rest lives in a terminal that is gone by morning. A reviewer six months out cannot tell whether an `LGTM` answered "review this diff" or "confirm my four rebuttals are right".

So: **after every `codex exec`, post that exchange to the PR before doing anything else.** One comment per invocation:

````bash
gh pr comment <N> --body-file - <<'EOF'
## Codex exchange — iteration <k>

**Prompt sent:**

```text
<the prompt, verbatim>
```

<details><summary>Codex reply (verbatim)</summary>

```text
<the full stdout, verbatim>
```

</details>
EOF
````

- **Verbatim, not summarised.** A summary of what Codex said is *your reading* of it, and your reading is the thing the second reviewer exists to check. Paraphrase and judgement belong in your triage comment, which is a separate comment.
- **Every invocation**, including the ones that produce nothing quotable. A round where Codex answered "no change needed" is evidence about the loop; its absence reads as a round that never ran.
- **Follow-up questions especially.** The rounds where you ask Codex to judge your rebuttal are the most valuable half of the record and the easiest to lose, because those runs usually end with "do not apply any fixes and do not post" — so Codex posts nothing itself and the entire exchange exists only in your scrollback.
- **Strip secrets, and say what you stripped.** Tokens, credential paths, anything out of `.env`. Nothing else gets edited out — "this part was noise" is a judgement call the record should not be making.
- Wrap long output in `<details>` so the thread stays readable. Never truncate to make it fit; a shortened reply is a summary wearing a transcript's clothes.

### Iteration step C — YOU evaluate each finding

**This is not a passive apply step.** For every finding from Codex:

1. **Necessity** — Is this a real bug, a style preference, or a false positive? **Reproduce it as a runnable mutation before accepting**, not in your head: apply the exact change the finding describes and watch what the suite says. Reasoning about which cases are safe is the step that fails — in one loop it produced "these two array methods are harmless" (they carried the same alias as the four being fixed) and "`length` returns a number so it cannot mutate" (`length = 0` truncates the array). Both took seconds to disprove by running and neither was going to be disproved by thinking harder. A mental repro is enough only for a finding you are about to REJECT and can point at the code that refutes it.
2. **Blast radius** — If Codex flags issue X at one site, search the codebase for the same pattern elsewhere. A single-site fix that leaves three other copies broken is worse than the original finding.
3. **Side effects** — Will the suggested fix break callers? Violate a convention? Regress a test? Check before applying.
4. **Gaps Codex missed** — Look at the diff yourself with fresh eyes. Codex's review is your starting point, not your ceiling.
5. **Categorise**: MUST-FIX / VALID-NIT / FALSE-POSITIVE / DEFER-TO-FOLLOWUP.

Apply MUST-FIX + VALID-NIT in this iteration. For FALSE-POSITIVE and DEFER cases, post a top-level reply explaining why you're not fixing them — Codex can then factor that into the next verdict.

**Verify the premise, not just the conclusion.** A finding arrives as *claim + reason*, and the two fail independently. Codex can be right that a change is wrong for a reason that is not true — "deleting this breaks historical pricing" was sound advice about a table nothing prices from. Trace the reason to the code before writing it into a commit message or a comment: whatever you accept as a premise becomes the justification a future reader inherits. If the conclusion survives but the reason does not, say so and give the real one.

### The loop converges on YOUR additions, not the original diff

By iteration 3 most findings are about code, comments, tests, and docs **you wrote while responding to earlier iterations**. That is the loop working, and it is also its main hazard: review-driven changes get written under time pressure, land without the scrutiny the original diff got, and carry a false authority because a reviewer asked for them.

Hold everything you add mid-loop to the *same* bar as the original diff:

- **A fix justified by a symptom it cannot affect is not a fix.** Before claiming a change resolves something, confirm the code path from the change to the symptom exists. Two lists with similar names, two tables with similar contents — check which one the surface actually reads.
- **Search for existing coverage before writing new coverage.** Duplicating a weaker version of a check that already exists is worse than adding nothing, because it reads as coverage.
- **State the property your new test asserts, then confirm the implementation agrees.** A test whose comment describes the opposite of what the code does still passes, and the comment is what the next person believes.
- **A new test can encode the wrong rule and pass.** A check written against one configuration can forbid what a second supported configuration requires. Ask which configurations the thing under test serves, and whether the assertion holds in each.
- **Break-verify every fix, including the ones a reviewer asked for** — and confirm the break actually broke something. A patch that silently failed to apply produces a green run that looks like a test gap.
- **The mutation harness edits production code, so check the restore rather than assuming it.** A script that loses its backup — a wrong working directory is the usual way — fails in both directions: a patch that went nowhere reports green and reads as "the test does not catch this", and mutations that accumulate across cases report red and read as "this case is covered". One loop hit that three times. Verify the tree is clean between cases (`git status`, or count the mutation's own marker back to zero), and use absolute paths.

**When the same class of finding keeps arriving, change the shape of the rule, not the number of cases.** A reviewer who finds a new way past your check every round is telling you the check enumerates BAD forms — and a language always has one more way to say a thing than anyone will list. Enumerate what is PERMITTED instead, and report everything else. In one loop that inversion ended four rounds of ban-this-then-ban-that: the rule went from "these array methods are forbidden" to "the batch may appear in these three positions", which deleted two checks and simultaneously closed doorways nobody had thought of (computed access, `.bind`, aliasing). It also fails closed, so the next unimagined form arrives as a red test rather than a silent hole — at the cost of rejecting some safe code, which is the trade you want here and should say out loud in the test.

When reporting, attribute honestly: if an iteration's finding was a defect in your own earlier fix, say so. It tells the human which commits to read hardest.

### Iteration step D — local checks

After any code change, run the project's mandated checks (derived from CLAUDE.md or conventional defaults):

- `yarn format` (Prettier / similar)
- `yarn lint`
- `yarn typecheck`
- `yarn build`
- `yarn test` if unit tests exist
- Skip e2e in-loop by default (expensive, often env-dependent); CI will catch e2e regressions

If any check fails, fix and retry before pushing. Do **not** commit a red build into the loop — it breaks the shared state between reviewers.

**Run what CI does not.** Check the job list rather than assuming (`gh pr checks <N>`). A repo whose nine checks are five CodeQL jobs, secret-scan, e2e, test and typecheck runs **no build** — and vitest transforms rather than builds, so nothing catches a production-build failure but you.

**A red check is not automatically yours.** Read the failure before chasing it: a timing assertion in a file the PR does not touch, failing at 3.06 against a threshold of 3, is a loaded runner. Say so on the PR — an unexplained red tick on an approved security PR teaches everyone that this job's failures are ignorable, right up to the one that is not.

### Verify what you pushed, not what you meant to push

`git add` fails silently often enough to bite: an `index.lock` collision with another process, an error swallowed by `2>/dev/null`, and only half the change is committed.

> A `console.warn` fix and its test were staged together. The implementation's `git add` lost the race; the test alone got committed and pushed. That test asserts a warning the pushed code does not emit — CI would have gone red for a reason that reads as a broken test. `git status` had shown `M ` and ` M` side by side in the output and it still went out.

Before committing, look at what is staged. Before pushing, look at what the commit contains (`git show --stat HEAD`). After pushing, if it matters, fetch the ref and confirm the remote has it.

### Iteration step E — sync the default branch (before every push)

```bash
git fetch origin "$DEFAULT_BRANCH"
NEW_MAIN=$(git rev-parse "origin/$DEFAULT_BRANCH")
if [ "$NEW_MAIN" != "$LAST_KNOWN_MAIN" ]; then
  git merge "origin/$DEFAULT_BRANCH" --no-edit
  # Resolve conflicts: if they're in files you edited this iteration,
  # prefer your edits but re-apply the main-side logic. If they're purely
  # structural (e.g., both sides added an import), auto-resolve.
  # If the conflict is semantic and unclear, pause and ask the user.
  LAST_KNOWN_MAIN=$NEW_MAIN
  # Re-run local checks after the merge — main may have changed contracts.
fi
```

### Iteration step F — commit + push

- `git add` only the files you touched intentionally. Never `git add -A`.
- Commit message: `fix: address codex review <iteration-<k>>` with a body listing the findings you accepted, in order. Include the current model's standard `Co-Authored-By` trailer.
- `git push` — normal, no force.

### Iteration step G — decide whether to loop

Parse the Codex verdict marker from step B:

An iteration is **clean** only when BOTH reviewers pass on the same commit: Codex posted `CODEX VERDICT: LGTM`, and your own evaluation of that same head has no MUST-FIX left. Anything else is not clean.

- **Not clean** → next iteration. This holds whether you applied the finding or declined it: a rebuttal you never put back in front of Codex is a disagreement you awarded to yourself. Post the reasoning, push, and run the next round so it can answer — it may accept, or it may show the rebuttal is the thing that is wrong.
- `CODEX VERDICT: LGTM` but you found real issues it missed → **not clean**. Fix them, push, loop.
- No verdict marker → treat as `CHANGES REQUESTED`, note the protocol failure, retry with a reminder.

### Exit: two clean iterations in a row

**One clean round is not convergence — it is a round that happened to find nothing.** A fix pushed in response to round N is reviewed for the first time in round N+1, and by then most findings are about the loop's own work rather than the original diff. So the loop ends when **two consecutive iterations are clean, with the second reviewing a head the first has already seen** — no new commit between them, or if there was one, it starts the count again.

Concretely: a clean round with commits pushed in it does not count as the first of the two. Push nothing, re-run, and get a second clean verdict on the unchanged head.

### The cap depends on the change

- **Simple changes** (one file, a rename, a config value, a docs edit): stop at **5** and surface the stalemate. Two reviewers who cannot agree in five rounds on something small are disagreeing about something other than the code.
- **Complex changes** (a behaviour change on a hot path, anything touching a route handler or shared runtime, a diff whose blast radius you could not enumerate at the start): **no cap.** Run until two consecutive clean rounds, however many that takes.

The reason to remove the cap is what the rounds actually produce. In one loop, rounds 1–4 each found a real defect and three of them were defects in fixes made *during* the loop: an equivalence fix that was not equivalent, a flag set on the wrong event so the bug it fixed came back by another door, a rule patched at three exits when a fourth existed, and a comment claiming an invariant the code did not have. Stopping at five would have merged the fifth.

**Do not stop mid-loop to ask the user whether to continue.** Asking is how a two-reviewer protocol quietly becomes a one-reviewer one. The only things that end it early are a `codex exec` that errors out, and a conflict step E says to escalate.

If a no-cap loop passes ~10 rounds without two clean in a row, say so and keep going — but read the pattern out loud first. Findings that keep arriving in the same shape mean the fix enumerates bad cases instead of stating the rule (see the note above on inverting to what is PERMITTED); findings that arrive in new shapes each round mean the change is bigger than the PR admits and may want splitting.

## CI monitoring (continuous, parallel to the loop)

Between every iteration — and continuously while waiting for CI after the loop ends — run:

```bash
gh pr checks <N> --json name,state,conclusion,link
```

**Handling CI failures:**

1. Identify the failing job.
2. Fetch logs: `gh run view <run-id> --log-failed`.
3. Reproduce locally if possible; otherwise read the log carefully.
4. Fix, run local checks, commit (use commit message `fix: CI <job-name> <short-reason>`), push.
5. Sync main first (step E).

**Handling merge conflicts from new main commits:** same as step E; this isn't rare — mainline typically moves during a review cycle.

Do not ignore a failing CI just because it looks unrelated to your changes — a pre-existing failure that you happen to be merging into IS your problem now. Escalate to the user only if the failure is clearly outside your scope (e.g., infrastructure / secrets).

## Merge (once both reviewers are OK AND CI is green)

1. Confirm with the user: "Codex LGTM + my evaluation clear + all CI checks green. Ready to merge?"
2. On user confirmation: `gh pr merge <N> --merge` (merge commit per project convention; NEVER squash unless the user overrides).
3. After merge: delete the local branch, confirm the mergeCommit SHA, and report the final state.

## Safety rules (always)

- Never skip local checks. Never `--no-verify` / `--no-gpg-sign` / `--force`.
- Never apply a Codex suggestion blindly. Every fix must pass your own review.
- If you disagree with Codex, SAY SO in a reply comment with reasoning — silent disagreement breaks the convergence protocol.
- Every `codex exec` round-trip is posted to the PR verbatim, prompt included. The loop's reasoning must outlive the terminal it ran in.
- Always sync main before pushing. Stale branches create artificial conflicts and confuse both reviewers.
- Never commit secrets. Never include `.env` or credential files in the diff.
- If you find issues Codex missed, do NOT pretend they came from Codex. Attribute them honestly in your commit message ("observed during Claude review, not flagged by Codex").
- Respect project-specific rules in `CLAUDE.md` — they override these defaults on conflict.

## What "OK" means from Claude's side

Not "Codex had nothing to say." Claude's own OK requires:

- No MUST-FIX findings remaining (either from Codex or your own reading).
- No obvious related issues in adjacent code that the diff invites but doesn't fix (if you deliberately deferred any, you posted a comment saying so with a rationale).
- All project checks pass locally.
- Test coverage matches the change shape — new logic has at least one test asserting the new behaviour, **break-verified**: mutate what it covers and watch it go red. A test written mid-loop is the most likely one to pass for a reason you did not intend.
- Every claim the PR body and the code comments make is one the code supports. Claims are where loop-driven changes rot: a body promising "the reasoning survives a reconnect" while the reattach parser drops those frames, a comment asserting "every exit writes a terminal frame" when three do not, a plan listing a test that was never written. Re-read them against the diff at the end, not at the start.
- i18n / a11y / security conventions from `CLAUDE.md` are honoured.

Only then do you wait on CI and ask the user for the merge go-ahead.

### Reviewing someone else's converged PR

An `LGTM` is a claim, and its history says how much to trust it. Read the verdict sequence before the diff:

- **An LGTM followed by several `CHANGES REQUESTED`** means the first one was premature. Whatever produced that pattern is still in the change — in one PR, an LGTM at round 3 was followed by six more rounds, ending with a P1 org-boundary finding at round 9 and an LGTM seven minutes later.
- **A guard applied endpoint by endpoint is an enumeration, not a boundary.** If the tests name the same endpoints the guard names, nothing goes red when a new one is added without it. Suggest walking the router and enumerating what is *permitted* to skip.
- **Verify the boundary yourself** rather than taking the final verdict, especially when the last finding was severe and the sign-off arrived quickly.

### After the loop ends

An approval and a green tick are stamped on a **commit**, not on a PR. If anything is pushed afterwards — including a one-line doc fix — say so plainly when reporting, because the approval no longer covers what will merge.

## Reporting to the user

After every iteration, give the user a 3-4 line status:

- Iteration `<k>` of 5
- Codex verdict: `LGTM` / `CHANGES REQUESTED (<N> issues)`
- What you changed this iteration (1-line summary)
- CI status at this moment

Final report on merge: PR number, merge commit SHA, total iterations, notable disagreements if any.
