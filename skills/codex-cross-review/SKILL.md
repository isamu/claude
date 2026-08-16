---
description: Dual-reviewer loop on a GitHub PR. User pastes a PR URL or number; you invoke Codex to review and post findings, then YOU critically evaluate each finding (necessity, side effects, missing related issues), apply valid fixes, re-request Codex, and repeat. There is no iteration cap; the exit bar is set by the PR's tier — a trivial PR takes two self-review passes and no Codex round, a simple one takes a single clean round on an unchanged head, a complex one takes two consecutive clean rounds. Any real finding promotes the tier and resets the count. The loop stops early only for a decision that is genuinely the user's. Every request carries a ledger of settled findings so Codex does not re-litigate them, and every fifth round both reviewers re-read the whole PR to check the discussion is still aimed at the right thing. Throughout the loop you monitor CI, sync main, and resolve conflicts. The loop can also conclude the PR should be CLOSED rather than merged. Merge only when both reviewers are OK and CI is green.
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
7. Read the whole diff (`git diff "$BASE_SHA"...HEAD`) and **tier the PR** — see the next section. Say the tier and the reason to the user before round 1.

## Tier the PR — the exit bar is set by the change, not by the protocol

A five-line changelog edit and a rewrite of the auth middleware do not deserve the same number of rounds. Tier the PR once at setup, from the diff you just read, and state the reason out loud. The tier decides only **how much review the change has to survive** — never how carefully any single round is done.

| Tier | What it is | Reviewers | Exit bar |
|------|------------|-----------|----------|
| **T — trivial** | no change to shipped behaviour at all | you, twice | two self-review passes, no Codex round |
| **S — simple** | one bounded concern; the diff shows the whole behaviour change | Codex + you | ONE clean round, on a head nothing was pushed to |
| **C — complex** | everything else | Codex + you | TWO consecutive clean rounds, the second on an unchanged head |

**C is the default; T and S are exceptions you have to argue for.** If the argument for "this is trivial" takes more than a sentence, it is not trivial — write the tier down as the sentence you would defend, and if you cannot write that sentence, the tier is C.

### What forces tier C regardless of diff size

- auth, permissions, secrets, payments, crypto, PII
- destructive or irreversible operations: deletes, data migrations, publish / release, infra teardown
- concurrency, async ordering, retry / replay, caching and invalidation
- anything every request or every caller passes through — middleware, a route entry point, a handler signature, a shared helper with many call sites
- a public API, schema, wire format, or on-disk format change
- **any claim of behaviour preservation** — "refactor", "no functional change", "same output". Those are proved by running old and new side by side, never by reading (`/refactor-safely`), and a diff that looks small is exactly how they slip through
- a bug fix whose root cause is not proven, or a bug you could not reproduce
- generated or vendored files, where the diff is not the source of truth

### What qualifies as tier S

All of these, not most:

- one concern, in one or two source files, and the behaviour change is visible in the diff itself
- the callers are countable and you counted them
- the new behaviour has a test that goes red when you break it — existing or added here
- nothing from the tier-C list
- every claim in the PR body is checkable against the diff alone

### What qualifies as tier T

No shipped behaviour changes at all:

- docs, comments, README, CHANGELOG
- a version bump
- test-only additions that do not touch production code
- formatting produced by the repo's own formatter
- a CI or config change that is purely additive and changes no permission, secret, or published artifact

Plus: nothing from the tier-C list, and CI green.

### Promote on contact — the tier only ever moves up

The tier is a prediction about where the bugs are. A real finding is that prediction failing, so it costs you the discount:

- **Any P1 or P2 finding promotes the PR one tier and resets the clean-round count.** T → S at minimum, S → C, and straight to C if the finding lands anywhere in the C list. A tier-T PR that produced a code change was never trivial: promote it and run Codex.
- **The diff growing into the C list promotes it** too, even with no finding — mid-loop fixes are how a two-file PR acquires a middleware change.
- **Nothing demotes.** A PR does not become simple because the last two rounds were quiet; quiet rounds are what the exit bar already counts.

This is the pattern the skill already records in *Reviewing someone else's converged PR*: an LGTM followed by `CHANGES REQUESTED` means the first LGTM was premature. Promotion is that lesson applied before the fact rather than after.

### Tier T — two self-review passes, no Codex

You are both reviewers here, so the two passes have to be genuinely separate — a second read taken immediately after the first sees what the first one decided, not what the diff says.

1. **Pass 1** — read the full diff against the PR title and body. Run the project's checks (step D). Push whatever it produces.
2. Wait for CI green.
3. **Pass 2** — on the head that will merge, with nothing pushed since pass 1. Re-read the diff *whole*, as if it arrived from someone else with no argument attached: does it do what the body says, does the body claim anything the diff does not support, is any file in it by accident.
4. **Any code change in pass 2 means the PR was not tier T.** Promote it, and run the Codex loop from round 1.

Post one comment recording it: the tier, the one-sentence reason, what each pass looked at, and that no Codex round ran. The record has to say *why* the second reviewer is missing — otherwise a reader six months out cannot tell a tiered-out PR from one where the protocol was skipped.

## The Loop (ends at the tier's exit bar — no iteration cap)

Keep a per-iteration state file at `/tmp/codex-cross-review-<N>/iteration-<k>.json` summarising what Codex said, what you decided, and what you changed. Helps post-mortem if the loop spins.

### The ledger — one cumulative file, rewritten every iteration

Also keep `/tmp/codex-cross-review-<N>/ledger.md`: every finding raised so far and what happened to it. This file is what you paste into the next Codex prompt (step A), and it is the only thing standing between round 8 and a fourth re-run of an argument settled in round 2.

One row per finding, appended as they are disposed of:

```markdown
| # | iter | finding (one line) | disposition | why / where |
|---|------|--------------------|-------------|-------------|
| 1 | 1 | `parseRange` accepts a reversed span | FIXED | commit a1b2c3d, test `test_range.ts:42` |
| 2 | 1 | suggests memoising `buildIndex` | REJECTED | called once per boot; measured 0.4 ms |
| 3 | 2 | i18n key missing in `ja` | FIXED | commit d4e5f6a, all 8 locales |
| 4 | 2 | wants a retry around the fetch | DEFERRED | needs a backoff policy decision — issue #123 |
```

Dispositions: `FIXED` / `REJECTED` / `DEFERRED` / `REOPENED`. A `REJECTED` row must carry the evidence that refuted it, not just the verdict — "false positive" tells the next round nothing and invites the finding back.

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

   ALREADY SETTLED — do not raise these again in the same form:
   <paste ledger.md verbatim here>

   That table is context so you do not spend this round re-deriving what earlier rounds
   already decided. It is NOT a list of closed topics: if a resolution is wrong, or a FIXED
   row's fix does not do what it claims, say so and mark it 'REOPENING #<row>' with the
   specific evidence. Silence on a bad fix is worse than a repeat finding.

   REPORT EVERYTHING IN ONE PASS. Read the whole diff before you post, and post every
   finding you have in this round — do not hold some back for a later round, and do not
   stop at the first problem you find. Give each finding a severity (P1 blocker / P2 should
   fix / P3 nit) so the response can be ordered. If a finding is a variant of one already in
   the table, say which row it varies and what is different about it.

   At the END of your work, ALWAYS post ONE final top-level comment that starts with a
   verdict marker on its own line:
     - 'CODEX VERDICT: LGTM' if you have no outstanding concerns
     - 'CODEX VERDICT: CHANGES REQUESTED' followed by a bulleted summary of remaining issues

   Do not apply any fixes yourself — only review and post findings."
```

Wait for `codex exec` to finish. If it errors out, record and stop the loop (ask user to rerun manually).

**Send the ledger every round, including round 1** (where it is empty — say so explicitly, `_no findings yet_`). A prompt whose shape changes between rounds makes the verdicts harder to compare.

**A finding that repeats despite the ledger is a signal, not noise.** It means either the row's one-liner does not describe what Codex actually meant, or the resolution genuinely did not hold. Re-read your own row before dismissing the repeat — the cheapest explanation is that you summarised the finding into something you had already fixed.

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
6. **Write it into the ledger** — one row per finding, before you move to the next one. The ledger is written here, while you still have the evidence in hand; reconstructing it at the top of the next round is how rows become "false positive" with no reason attached, which is what brings the finding back.

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

Before counting the round, apply *Promote on contact*: a P1 or P2 finding this round moves the tier up and zeroes the count, so a round can raise the exit bar at the same time as it fails to meet it.

- **Clean, and the tier's exit bar is now met** → exit (see *Exit* below).
- **Clean, but you pushed commits this round** → not an exit at any tier. Push nothing, re-run, and get the verdict again on the unchanged head.
- **Not clean** → next iteration. This holds whether you applied the finding or declined it: a rebuttal you never put back in front of Codex is a disagreement you awarded to yourself. Post the reasoning, push, and run the next round so it can answer — it may accept, or it may show the rebuttal is the thing that is wrong.
- `CODEX VERDICT: LGTM` but you found real issues it missed → **not clean**. Fix them, push, loop.
- No verdict marker → treat as `CHANGES REQUESTED`, note the protocol failure, retry with a reminder.
- **The next iteration is a multiple of 5** → run the checkpoint first (*Every fifth round*), then step A.
- **The round established the PR should not exist** → stop the loop and take the close path (*Closing the PR is a legitimate outcome*). This outranks a pending verdict: an LGTM on a change that should not ship is not a reason to merge it.

### Exit: clean rounds on the head that will merge

Two requirements hold at every tier, because they are what a verdict actually means:

- **The clean verdict must cover the head that will merge.** A fix pushed in response to round N is reviewed for the first time in round N+1, so a clean round you pushed commits in never satisfies the bar. The round that fixed things is not the round that ends the loop; the quiet round after it is.
- **Clean means both reviewers on the same commit** — Codex `LGTM` and your own evaluation of that head with no MUST-FIX left.

How many such rounds you need is the tier's only job:

- **Tier S — one.** A single clean round on a head nothing was pushed to ends the loop.
- **Tier C — two in a row.** One clean round is not convergence, it is a round that happened to find nothing; and by round 3 most findings are about the loop's own work rather than the original diff. Two consecutive clean iterations, the second reviewing a head the first has already seen — a commit between them starts the count again.
- **Tier T — no Codex rounds at all**, two self-review passes instead (see *Tier T* above).

### There is no cap

**The loop runs until the tier's exit bar is met, however many rounds that takes.** The tier sets how many clean rounds you need; it never sets a round budget and is never a reason to stop early. A one-line diff that cannot converge in five rounds is not a small change — it is a disagreement about something the diff does not show, and it promoted itself out of tier S at the first finding anyway. Stopping there hides exactly the thing worth finding.

What the rounds actually produce is the argument. In one loop, rounds 1–4 each found a real defect and three of them were defects in fixes made *during* the loop: an equivalence fix that was not equivalent, a flag set on the wrong event so the bug it fixed came back by another door, a rule patched at three exits when a fourth existed, and a comment claiming an invariant the code did not have. A cap of five would have merged the fifth.

**Do not stop mid-loop to ask the user whether to continue.** Asking is how a two-reviewer protocol quietly becomes a one-reviewer one. "This is taking a while", "we are at round 12", and "the remaining findings look minor" are not reasons to stop — they are the loop working.

The loop ends early for exactly these, and nothing else:

1. **A decision that is genuinely the user's** — see *Stopping for the user* below.
2. **`codex exec` errored out** and does not recover on a retry. Record it and hand back.
3. **A semantic merge conflict** that step E says to escalate.

If the loop passes ~10 rounds without meeting the exit bar, say so and keep going — but read the pattern out loud first. Findings that keep arriving in the same shape mean the fix enumerates bad cases instead of stating the rule (see the note above on inverting to what is PERMITTED); findings that arrive in new shapes each round mean the change is bigger than the PR admits and may want splitting — or closing.

### Stopping for the user

The exception to "do not stop" is narrow: a question the code cannot settle, where proceeding either way would be guessing at the user's intent. Concretely:

- **The loop concludes the PR should be closed** (see below). Closing someone's work is the user's call, so you bring the recommendation and the evidence, not the action.
- **The merge go-ahead**, as it always was.
- **A finding whose resolution is a product or policy decision** — which of two valid behaviours is wanted, whether a breaking change is acceptable, whether a deferred item blocks the release. Post the options and what each costs; do not pick one and call it converged.
- **A conflict whose correct resolution is not derivable** from either side's history.

Everything else — including a finding you and Codex flatly disagree on — is resolved inside the loop by putting the rebuttal back in front of Codex, not by escalating. When you do stop, say which of these it is, and what specifically you need decided.

### Every fifth round, both reviewers re-read the whole PR

Findings compound. By round 5 the conversation is mostly about the loop's own output, and a loop can converge beautifully onto the wrong thing — a rule refined for four rounds that should not exist, a test suite grown around a behaviour the PR was never supposed to have. Round-by-round review cannot see this, because each round only ever looks at the delta.

So at iterations **5, 10, 15, …**, before step A, run a checkpoint. Both sides look at the PR *whole*, not at the latest findings.

**Your half** — re-read the full diff (`git diff "$BASE_SHA"...HEAD`) against the PR's stated goal, as if seeing it for the first time:

- Does the diff still do what the PR title and body say? If the body now describes a different change, the body is stale or the PR has drifted — name which.
- How much of the diff is original work versus loop-driven additions? A PR that is now majority review-response is a PR whose centre of gravity moved.
- Is anything in it there only because a reviewer asked, and no longer justified on its own? Removing it is a legitimate outcome.
- Would you approve this diff if it arrived fresh today, with none of the argument attached?

**Codex's half** — a different prompt from the normal round, aimed at direction rather than defects:

```bash
codex exec --sandbox workspace-write \
  "Checkpoint review of PR #<N> at https://github.com/<owner>/<repo>/pull/<N>.

   This is round <k> of a review loop. Do NOT hunt for new line-level defects this round.
   Read the PR whole — title, body, full diff, and the review history below — and answer:

   1. Does the diff still accomplish what the PR says it does?
   2. Has the review conversation drifted onto something other than the PR's purpose?
   3. Is any part of the diff there only because a reviewer asked for it, without
      standing on its own merit?
   4. Should this PR be split, redirected, or CLOSED rather than converged? Say so plainly
      if yes, with the reason.
   5. What is the single largest remaining risk in this change?

   Review history so far:
   <paste ledger.md verbatim here>

   Post your answer as ONE top-level comment starting with 'CODEX CHECKPOINT: round <k>'.
   Do not apply any fixes and do not post a verdict marker this round."
```

Post both halves to the PR. A checkpoint round does **not** count toward the tier's exit bar — it produces no verdict — and it does not reset the count either; it sits between rounds. (A PR that reached round 5 is tier C by then in all but name; if it is still labelled S, that label is stale — promote it.) If the checkpoint changes direction (scope cut, split, close), the clean-round count starts over, because what the earlier verdicts approved is no longer what will merge.

### Closing the PR is a legitimate outcome

A review loop is not obliged to end in a merge. Two reviewers arguing in good faith sometimes establish that the change should not exist, and the protocol has to be able to say so — otherwise every PR converges by construction, and the loop's only possible output is approval of whatever it started with.

Recommend closing when the discussion has established one of these, with evidence on the PR:

- **The premise is false.** The bug does not reproduce, the slow path is not hot, the platform behaviour it works around was fixed upstream. A PR fixing something that is not broken has no correct version.
- **The right fix is somewhere else.** The change treats a symptom, and the loop located the cause in another layer. Say where, and file or link the issue that replaces this.
- **It has been superseded.** Mainline moved during the loop and now does this, or does something incompatible that arrived with more context.
- **The cost exceeds the benefit, and the loop measured both.** Not "this is getting complicated" — an actual accounting: what it buys, what it costs to carry, why the balance is negative.
- **It should be split, and nothing is left after splitting.** If every part belongs in its own PR, this one is a container, not a change.

What "recommend" requires — closing is the user's call, so bring it decided, not open:

1. **Post the case on the PR first**, as one comment: what was believed at the start, what the loop established, and which of the reasons above applies. Link the specific findings and commits that got you there.
2. **Ask Codex to judge the close recommendation itself** — the same rebuttal discipline as any other finding. A close argued by one reviewer alone is exactly the kind of unilateral conclusion this skill exists to prevent.
3. **Say what replaces it**: the issue to open, the smaller PR to cut, or nothing at all — and if nothing, say that explicitly, because "closed and forgotten" and "closed because it is already handled" look identical six months out.
4. **Then stop and ask the user**, with the recommendation stated in one line. Never run `gh pr close` on your own judgement.

A close that arrives at round 9 is not a wasted loop. The rounds are what established the premise was false; without them it would have merged.

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

1. Confirm with the user: "Codex LGTM + my evaluation clear + all CI checks green. Ready to merge?" — and name the tier in that line, because the user is approving the tiering as much as the merge. For tier T say it plainly: "no Codex round — tier T (<the one-sentence reason>), two self-review passes clean."
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
- Never `gh pr close` on your own judgement, and never merge one. Both are the user's call; you bring the recommendation with its evidence.
- The tier is stated on the PR before round 1, promoted the moment a real finding lands, and never lowered. A tier decided silently is a shortcut nobody can audit; a tier lowered mid-loop is the loop grading its own homework.
- Tiering changes how many rounds a PR must survive, never how a round is done. There is no lighter review, no skipped local checks, and no unverified finding at tier S or T.
- The ledger is a memory aid, never an authority. A row saying `REJECTED` does not make a repeat finding wrong — it makes it worth re-reading the row.
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

- Iteration `<k>` (no cap — running to tier `<T/S/C>`'s exit bar)
- Codex verdict: `LGTM` / `CHANGES REQUESTED (<N> issues)` / `CHECKPOINT` on rounds 5, 10, …
- What you changed this iteration (1-line summary)
- Clean-round count: `S — 0 of 1` / `C — 1 of 2` — and if the tier was promoted or the count reset, why
- CI status at this moment

This is a status line, not a question. Do not end it with "shall I continue?" — the loop continues unless one of the three early-exit conditions fired, and asking invites a stop the protocol does not want.

Final report on merge: PR number, merge commit SHA, tier it exited at (and any promotion, with what caused it), total iterations, notable disagreements if any.

Final report on a close recommendation: which reason applies, the findings that established it, what replaces the PR, and Codex's judgement of the recommendation.
