---
description: Dual-reviewer loop on a GitHub PR, tuned to converge in as FEW rounds as possible. User pastes a PR URL or number; you invoke Codex under a one-pass completeness contract — every finding this round, with severity, blast radius and the smallest resolving change — then YOU evaluate the whole batch, settle disagreements with Codex inside the same round, apply every accepted fix in one push, and re-request. Exit is ONE clean round on a head nothing was pushed to; a trivial PR takes two self-review passes and no Codex round at all. The tier sets how DEEP a round is, never how many rounds you owe. Any real finding costs one more round, which is why the loop's whole job is to make each round complete. Setup proves Codex's sandbox can actually run tests, bind a socket and reach GitHub before round 1, because a reviewer that cannot run something reports no findings. Every request carries a COMPACT ledger of settled findings so Codex does not re-litigate them, round 5 folds a whole-PR direction check into the same call rather than spending a round on it, and the third finding against one symbol stops the case-by-case fixing and re-shapes the rule instead. Dispatch each Codex round the moment you push — NEVER wait for CI, since the two answer different questions and serialising them spends the longer twice; run rounds for several PRs concurrently in the background. Throughout the loop you monitor CI, sync main, and resolve conflicts. The loop can also conclude the PR should be CLOSED rather than merged. You report to the user once at the end, not every round. Merge only when both reviewers are OK and CI is green.
---

# Codex Cross-Review

Two-reviewer convergence loop. Codex raises findings; you (Claude Code) evaluate them with full context, apply real fixes, and request a follow-up review. Both reviewers must sign off before merge.

## Rounds are the cost — everything below exists to spend fewer of them

A fix pushed in response to round N is reviewed for the first time in round N+1. That single fact sets the arithmetic, and it is not negotiable — an unreviewed fix is not reviewed:

> **Ten findings arriving one per round costs eleven rounds. The same ten arriving in one round costs two.**

So the loop is not slow because it is thorough. It is slow when a round is *incomplete* — when a reviewer holds findings back, when a disagreement is deferred to the next round instead of settled in this one, when a fix at one site leaves three identical sites for the next round to find. Each of those turns one round into two, and they compound.

Five devices keep the count down. Each has its own section; this is the list so they read as one design rather than five habits:

1. **A one-pass completeness contract on every Codex call** — every finding this round, with severity, every other site of the same pattern, and the smallest change that resolves it. A held-back finding is a protocol failure and gets named as one. → *Iteration step A*
2. **Disagreements are settled inside the round**, with a second Codex call before you push. A rebuttal parked until the next round is a round spent on an argument. → *Step C-bis*
3. **One batch, one push.** Every accepted fix from the round goes in together. → *Steps C, D, F*
4. **Fix the CLASS, not the site.** A finding names a class of mistake; sweep the whole class this round, or each surviving site comes back as its own finding in its own round. → *Iteration step C*, point 2
5. **The round-5 direction check rides in the same call** as that round's review, instead of costing a round of its own. → *Round 5*
6. **Each round is dispatched the instant the push lands, never after CI** — CI gates the merge, not the review, and waiting turns a 3–8 minute round into a 20-minute one. → *Iteration step F-bis*

And the exit bar is **one clean round**, at every tier. Two consecutive clean rounds was the old bar, dropped by the user's instruction on 2026-08-15. Depth moved into the round instead — which is what the tier now controls. The reason is recorded in *Exit*; do not restore the second round without reading it.

What this is NOT licence for: a shallower round, a skipped check, an unverified finding, or merging on a head Codex has not seen. Fewer rounds is bought by making each round complete, never by making it lighter.

### And three failure modes that make a round WORTHLESS rather than merely extra

Measured over the 91 loops with a ledger under `/tmp/codex-cross-review-*` (2026-08-21): **mean 7.4 rounds, median 5, longest 57**, and **62% of all 1,640 findings were raised in round 3 or later**. The long tail is not caused by hard PRs. It is caused by rounds that ran without being able to find anything:

7. **The reviewer's sandbox is broken and nobody checked.** → *Setup step 8*
8. **The same rule is patched case by case** until someone thinks to invert it. → *The third finding on one symbol*
9. **A claim exists in four places and only one gets fixed.** → *The claims sweep*

Two more make every round slower without making any of them wrong: the ledger pasted verbatim until it outweighs the diff (→ *The prompt carries a COMPACT ledger*), and one comment per Codex call until GitHub's rate limit blocks the loop outright (→ *ONE comment per round*).

Those three are why a loop reaches round 20. They are not caught by making a round complete — a complete round over a broken sandbox is still blind.

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
8. **Prove Codex's sandbox works, before round 1.** This is the cheapest step on this page and the one whose absence has cost the most rounds:

   **Two flags this repo's Codex needs, both learned the hard way on 2026-08-21:**

   - **`--model gpt-5.5`.** The configured default in `~/.codex/config.toml` was
     `gpt-5.6-luna`, which the installed CLI (v0.137.0) answers with
     `400 invalid_request_error: requires a newer version of Codex`. `codex exec`
     produces NO output and exits 0-ish noise. Pass the model explicitly rather than
     inheriting a config a CLI upgrade can invalidate.
   - **`-c sandbox_workspace_write.network_access=true`.** `--sandbox workspace-write`
     alone gives Codex NO network. Measured on a real worktree: vitest exit 1, socket
     bind exit 1, `curl https://api.github.com` status 000, `gh pr view` "error
     connecting". Every one of the documented failure modes below at once — and the
     round would still have ended with a verdict Codex could not post. With the flag:
     all four exit 0.

   Together they are the difference between a review and a silent no-op. Put both in
   EVERY call in the loop, not just the pre-flight.

   ```bash
   codex exec --model gpt-5.5 --sandbox workspace-write \
     -c 'sandbox_workspace_write.network_access=true' \
     "Do not review anything. Run these and report each command's EXIT CODE verbatim:
        1. <EVERY suite the review will lean on — ONE LINE EACH, see below>
        2. <the repo's typecheck command>
        3. node -e 'require(\"net\").createServer().listen(0,()=>process.exit(0))'
        4. curl -sS -o /dev/null -w '%{http_code}' https://api.github.com
      Then say, in one line each: WHICH OF THE SUITES in 1 you could run, can you BIND
      a socket, can you REACH api.github.com, and are file deletions permitted
      (try: touch /tmp/x && rm -f /tmp/x)."
   ```

   **Item 1 is PLURAL, and writing it in the singular is how half a review goes blind.** A repo
   with a backend suite and a frontend suite has two, and they fail INDEPENDENTLY — different
   runners, different configs, different directories, and one of them green tells you nothing
   about the other. Name each one in the probe, and hand Codex the invocation that WORKS rather
   than the one in the README. In a worktree with symlinked `node_modules` that means
   `--configLoader runner` on every vitest line, in the probe itself:

   ```
   1a. cd backend && npx vitest run --configLoader runner <one backend test file>
   1b. npx vitest run --configLoader runner --dir src <one frontend test directory>
   ```

   Write the four answers into the ledger as row 0. Every one of these has failed in a real loop here, and each failure was invisible for many rounds because a reviewer that cannot run something reports *no findings*, which is indistinguishable from clean:

   - `node_modules/.vite-temp` unwritable through a symlink → **Codex could not run vitest for EIGHTEEN rounds** on one PR. Eighteen rounds of test-related verdicts meant nothing.
   - could not bind a socket → **47 supertest files / 523 tests "failed"** on environment, not content. A red suite that is not about the code is worse than no suite: it gets fixed.
   - `api.github.com` unreachable → an entire round's inline comments **and its verdict** never posted. The round looked like it never happened.
   - `rm -f` refused by the sandbox → its mutation sweep's restore failed and the retry **mutated on top of a mutation**. Every count after that was measuring the harness.

   **What a failing answer changes:** nothing is skipped, but a claim Codex cannot execute is recorded as UNVERIFIED, and you run that check yourself. Say so in the round's triage comment. If the sandbox cannot be repaired, Codex is a reader of the diff and not a runner of it, and every verdict in the loop carries that qualification.

   **Why that flag belongs IN the probe and not in a note beside it.** With `node_modules`
   symlinked into a worktree, vitest's bundled-config write path cannot write
   `node_modules/.vite-temp`, and the run dies at Vite startup with `EPERM` — before a single
   test is collected. `--configLoader runner` does not write there and works.

   This paragraph already existed, and was paid for again on 2026-09-21 (#3915): the backend
   command was probed and the frontend command was not, so every page-side axis came back as a
   READER's verdict for two rounds. Codex said so honestly each time, which is the only reason
   it was recoverable. Round 3's prompt carried the flag and it ran the frontend suite at once.

   **The flag was never the missing knowledge — applying it, to every suite, in the probe, was.**
   A rule you have read and not executed costs exactly what not having it costs.

   **`rm` can stay blocked even with the network flag** — it is refused by policy, not by the
   sandbox mode. That is survivable, and the round has to say so out loud: **tell Codex not to
   run a mutation sweep**, because its restore will fail and every count it takes afterwards is
   measuring the harness. Offer instead to run any mutation it names.

   **RE-SYNC `main` INTO THE BRANCH BEFORE THE ROUND, not just before the first one.** A round
   reads the checkout, so a branch whose last `main` merge predates the PRs its own text describes
   will produce findings that are TRUE of the branch and FALSE of `main` — and they read like the
   reviewer being wrong. Measured on 2026-09-22: a documentation PR was rewritten to say three
   defects had closed, and the round raised three MAJORs saying they had not. Both were right; the
   branch was **thirty commits** behind, having merged `main` before two of the three landed. The
   round was not wasted, but the fix was a merge rather than an edit.

   So the pre-round checklist is: `git rev-list --count origin/<branch>..origin/main` is **0**, or
   the round is reviewing a world that does not exist. State the head's sha in the prompt so the
   reviewer can say which tree it read.

   **ONE WORKING TREE, so two backgrounded rounds on DIFFERENT BRANCHES cannot overlap.** Codex
   reads the checkout, not a snapshot of the diff. Dispatching a round for PR A and then checking
   out PR B's branch — or dispatching a second round that checks out its own branch — leaves the
   first reading whichever tree won, and it reports the fixes as unmade. Measured on 2026-09-22:
   a round on a fix branch was dispatched while the tree sat on a docs branch, and it would have
   raised every P1 again against code that no longer had the defect; it was killed and re-run.
   The same collision silently invalidated a 1,442-file vitest run in the same session — nine
   files "failed" because `git checkout` moved the tree under them.

   So: **one round in flight per working tree.** Run rounds for different PRs sequentially, or give
   each its own git worktree. And never `git checkout` while a round or a suite is running.

   **A `codex exec` that takes longer than the harness's per-call ceiling must run in the
   background** — and every backgrounded call needs `< /dev/null` and a `timeout`:

   ```bash
   timeout 2400 codex exec --model gpt-5.5 --sandbox workspace-write \
     -c 'sandbox_workspace_write.network_access=true' "$(cat prompt.txt)" \
     < /dev/null > raw.txt 2> err.txt
   ```

   Both were paid for on 2026-08-21. A review round on a real PR can exceed ten minutes, and
   killed at the foreground ceiling it writes a **zero-byte transcript and posts nothing** —
   indistinguishable from a round that found nothing. Backgrounded without `< /dev/null` it is
   worse: `codex` prints *"Reading additional input from stdin..."* to stderr and **waits
   forever** on a pipe that never closes. It sat at 0.0% CPU for thirty minutes, produced no
   output, and would never have sent the notification the loop was waiting on. The `timeout` is
   what turns that class of hang into an exit code you get told about.

   **`err.txt` is where the hang announces itself**, so read it when a round is silent for
   longer than a round should take. `raw.txt` being empty says nothing — codex writes its
   stdout at the end.

## Tier the PR — the tier sets the DEPTH of a round, not how many you owe

A five-line changelog edit and a rewrite of the auth middleware do not deserve the same review. They deserve the same *number of rounds* — one clean one — and a very different round. Tier the PR once at setup, from the diff you just read, and state the reason out loud.

| Tier | What it is | Reviewers | Exit bar | What the tier buys |
|------|------------|-----------|----------|--------------------|
| **T — trivial** | no change to shipped behaviour at all | you, twice | two self-review passes, no Codex round | nothing to review |
| **S — simple** | one bounded concern; the diff shows the whole behaviour change | Codex + you | ONE clean round, on a head nothing was pushed to | the default axes |
| **C — complex** | everything else | Codex + you | ONE clean round, on a head nothing was pushed to | **every C-list axis named in the prompt and answered per axis**, plus the behaviour-preservation proof where one is claimed |

**The count is the same at S and C; the round is not.** A tier-C round names the risk axes explicitly and demands a per-axis answer, so "no findings" becomes a statement about concurrency, auth and blast radius rather than a statement about whatever Codex happened to look at. That is the trade this skill makes: one deep round instead of two shallow ones. Reading a shallow round twice does not find what a deep round finds once — the nine-round loop that ended this bar spent rounds 5-9 on defects the loop itself had introduced.

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

The tier is a prediction about where the bugs are. A real finding is that prediction failing, so it costs you the depth you discounted:

- **Any P1 or P2 finding promotes the PR one tier.** T → S at minimum, S → C, and straight to C if the finding lands anywhere in the C list. A tier-T PR that produced a code change was never trivial: promote it and run Codex.
- **The diff growing into the C list promotes it** too, even with no finding — mid-loop fixes are how a two-file PR acquires a middleware change.
- **Nothing demotes.** A PR does not become simple because the last round was quiet.

Promotion changes the next round's prompt, not the number of rounds owed: from the promoting round on, the C-list axes are named and answered per axis. The clean-round count resets for the ordinary reason — a fix was pushed, so the head that will merge has not been reviewed yet — and not as a penalty.

This is the pattern the skill already records in *Reviewing someone else's converged PR*: an LGTM followed by `CHANGES REQUESTED` means the first LGTM was premature. Promotion is that lesson applied before the fact rather than after — and it is the reason one clean round is enough, because the round that grants it is a round the tier has already deepened.

### Tier T — two self-review passes, no Codex

You are both reviewers here, so the two passes have to be genuinely separate — a second read taken immediately after the first sees what the first one decided, not what the diff says.

1. **Pass 1** — read the full diff against the PR title and body. Run the project's checks (step D). Push whatever it produces.
2. Wait for CI green.
3. **Pass 2** — on the head that will merge, with nothing pushed since pass 1. Re-read the diff *whole*, as if it arrived from someone else with no argument attached: does it do what the body says, does the body claim anything the diff does not support, is any file in it by accident.
4. **Any code change in pass 2 means the PR was not tier T.** Promote it, and run the Codex loop from round 1.

Post one comment recording it: the tier, the one-sentence reason, what each pass looked at, and that no Codex round ran. The record has to say *why* the second reviewer is missing — otherwise a reader six months out cannot tell a tiered-out PR from one where the protocol was skipped.

## The Loop (ends on ONE clean round — no iteration cap)

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

**Row 0 is the sandbox pre-flight** (*Setup step 8*): what Codex can and cannot run. Every later row that leans on something it could not execute inherits that qualification.

#### The prompt carries a COMPACT ledger, not this file

The evidence column is what makes a row worth keeping and what makes it enormous. Measured: ledgers reach **80 KB by round 30**, and the median at eight rounds or more is **13 KB against 4 KB below that** — pasted verbatim into every prompt, on top of the diff it is supposed to be helping someone read. A prompt where settled history outweighs the change is a prompt that produces findings about settled history.

So keep two forms:

- **`ledger.md`** — the full record, evidence and all. On disk, quoted in the PR, never truncated.
- **What goes in the prompt** — one line per row: `#n | iter | finding | disposition | ≤120 chars of why`. Trim the evidence, keep the verdict and the reason.

Two rows always go in FULL, because they are the ones a compaction turns back into findings: anything `REJECTED` (its evidence is what stops the repeat) and anything `REOPENED`. And once the compact form passes ~4 KB, say so in the prompt — *"rows 1-30 are compacted; ask for any row in full and I will paste it"* — rather than letting it grow. Codex asking for one row costs nothing; a 40 KB preamble costs every round after it.

### Iteration step A — request Codex review

Run Codex non-interactively under the **one-pass completeness contract**. The contract is the round-reducing device: a reviewer that may hold findings back turns one round into as many rounds as it has findings, and no amount of care on your side can recover that.

```bash
codex exec --sandbox workspace-write \
  "Review PR #<N> at https://github.com/<owner>/<repo>/pull/<N>.

   ── THIS IS THE ONLY ROUND YOU ARE GUARANTEED ────────────────────────────────
   Every finding you hold back costs a full extra round: a fix is reviewed for the
   first time in the round AFTER it is pushed, so a finding raised late is not a
   later finding, it is a repeat of this entire exchange. Read the WHOLE diff
   before you post anything. Do not stop at the first problem. Do not save
   anything for 'a follow-up review'.

   Use the gh CLI to inspect the diff, then post inline review comments on specific lines via:
     gh api repos/<owner>/<repo>/pulls/<N>/comments
   or top-level comments via:
     gh pr comment <N>

   ── AXES — answer EVERY line, even when the answer is 'no findings' ──────────
   <the axis list — the default six, plus one line per C-list axis this PR touches>
     correctness and edge cases
     security (XSS / SSRF / path traversal / injection / secrets)
     tests: does a test go red if the new behaviour breaks
     API / schema / wire / on-disk compatibility
     consistency with the rest of the codebase
     accessibility and i18n lockstep (if this repo has multi-locale dicts)

   An axis with no findings is a RESULT and has to be stated. 'No findings' on an
   axis you did not look at is the one answer that makes this loop longer, because
   it is indistinguishable from a clean axis until round three.

   ── EACH FINDING CARRIES FOUR THINGS ─────────────────────────────────────────
   1. SEVERITY — P1 blocker / P2 should fix / P3 nit / N note.

      **N is for a finding whose only remedy is editing PROSE** — a comment, a
      docblock sentence, a plan file, the PR description. It does NOT block the
      verdict and does NOT make the round CHANGES REQUESTED. List N findings under
      the verdict; they are collected and fixed once, at the end, when the code has
      stopped moving.

      **A prose finding is P3 or higher, not N, when believing the sentence would
      lead someone to make a WRONG CHANGE.** "This is measured before the first
      await" when it is measured after; "the guard covers X" when it does not.
      Those are code findings wearing prose, and they are worth a round.

      Merely behind is N: a count that no longer matches, an enumeration the code
      outgrew, a name that moved, a figure from an earlier commit. **Counts are the
      recurring instance** — say so rather than filing each one.
   2. EVERY SITE. If the same mistake appears elsewhere in the repo, list every
      occurrence NOW — grep for it. A finding reported at one site and fixed at one
      site comes back next round as 'the same problem over here', which is the
      single most common way this loop doubles in length.
   3. THE SMALLEST CHANGE THAT RESOLVES IT — concretely enough to apply. A finding
      whose remedy is ambiguous costs a clarification round.
   4. WHAT WOULD CHANGE YOUR MIND — the fact that, if true, makes this a
      non-issue. This is what lets a disagreement be settled in this round instead
      of the next one.

   ── ALREADY SETTLED — do not raise these again in the same form ──────────────
   <paste ledger.md verbatim here>

   That table is context so you do not spend this round re-deriving what earlier rounds
   already decided. It is NOT a list of closed topics: if a resolution is wrong, or a FIXED
   row's fix does not do what it claims, say so and mark it 'REOPENING #<row>' with the
   specific evidence. Silence on a bad fix is worse than a repeat finding.
   If a finding is a variant of one already in the table, say which row it varies and
   what is different about it.

   <LATE-FINDING NOTE — include only when applicable, see below>

   ── END WITH ONE TOP-LEVEL COMMENT, IN THIS ORDER ────────────────────────────
     Line 1, a verdict marker on its own line:
       'CODEX VERDICT: LGTM' if you have no outstanding concerns ABOVE severity N
       'CODEX VERDICT: CHANGES REQUESTED' followed by a bulleted summary of remaining issues
     N findings never change that line. An LGTM with a list of N notes under it is
     the normal shape of a converged round, not a contradiction.
     Then the axis table: one row per axis above, findings or 'none'.
     Then, on its own line:
       'FINDINGS COMPLETE: I read every hunk of the diff and this is every finding I have.'
     Omit that last line if it is not true, and say which part of the diff you did
     not reach and why. An honest partial round is recoverable in one more round; a
     partial round claiming completeness is discovered three rounds later.

   Do not apply any fixes yourself — only review and post findings."
```

**Build the axis list from the tier.** Tier S gets the default six. Tier C adds one line per C-list item the PR actually touches — *concurrency and async ordering*, *auth boundary*, *retry / replay idempotence*, *cache invalidation*, *the behaviour-preservation proof*, and so on. Naming them is the whole of what tier C buys, and a named axis cannot be quietly skipped the way "review this PR" can.

**Name late findings in the next prompt.** If a finding arrives in round *k* about code that was unchanged since round *k-1*, it should have arrived a round earlier, and saying so is what stops it happening again:

```text
   ── LATE FINDING ────────────────────────────────────────────────────────────
   Ledger row #<n>, raised in round <k>, is about <file>:<line>, which has not
   changed since round <k-1>. It was reviewable then and would have cost no extra
   round. This round, before you look for anything new, re-read the parts of the
   diff you did not reach last time.
```

This is not a scolding; it is the only feedback channel the contract has. Without it the contract is a request repeated identically every round, and a request nothing checks is a request.

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

#### ONE comment per round — the flood breaks the thing it was protecting

The rule above is right and its failure mode is arithmetic. Measured on the PRs this loop has run: **371 comments on one PR, 239 on another, 217 of the first PR's being `## Codex exchange` bodies** — because a round makes several Codex calls and each was posted on its own. Then the predictable happened:

> On PR #2126, **~60 comments from the loop tripped GitHub's secondary rate limit, and ALL comment creation was blocked from ~03:45Z.** Codex could not post its round-11 LGTM or its round-12 verdict; both had to be relayed from stdout by hand. A **1,033,163-byte** checkpoint transcript was going up **in 22 parts**.

A record that cannot be written is not a record, and a thread nobody can read is not one either — the next round's reviewer pays for the volume too. So:

- **One comment per ROUND, not per call.** The round's step-A exchange, its step C-bis exchange and any follow-up go in one comment, each under its own `<details>`. Still verbatim, still nothing edited out but secrets.
- **Never split one transcript across comments.** A reply in 22 parts is not more complete than a reply in one; it is the same content, unreadable, at 22× the rate-limit cost. If it does not fit, post the verdict and the findings verbatim, keep the full stdout in the state dir, and **say plainly in the comment that the remainder was not posted, how big it was, and where it is.** An honest gap is recoverable; a silent one is not.
- **Back off, do not retry harder.** A 403/422 on comment creation is the secondary limit. Stop posting, finish the round, and post the backlog as one comment when it clears. Retrying is what turns a slow limit into a blocked one.
- **Budget it out loud.** Past ~40 comments on one PR from this loop, say so in the round report and switch to verdict-plus-findings only. That threshold is the observed one, not a documented GitHub number — treat it as a smell, not a constant.

- **Verbatim, not summarised.** A summary of what Codex said is *your reading* of it, and your reading is the thing the second reviewer exists to check. Paraphrase and judgement belong in your triage comment, which is a separate comment.
- **Every invocation**, including the ones that produce nothing quotable. A round where Codex answered "no change needed" is evidence about the loop; its absence reads as a round that never ran.
- **Check that Codex actually posted, and relay it when it did not.** It is instructed to post a verdict comment and it does not always manage — a network refusal, a rate limit, or simply not doing it. Across one session it posted round 1 of one PR and none of rounds 2-5, and skipped the FINAL LGTM on another. **Look in BOTH places.** It lands a verdict sometimes as an issue comment and sometimes as a PR *review*, and a check against only the first reports "it never posted" when it did:

```bash
gh api "repos/<o>/<r>/issues/<N>/comments" --jq '[.[]|select(.body|startswith("CODEX VERDICT"))]|length'
gh api "repos/<o>/<r>/pulls/<N>/reviews"   --jq '[.[]|select(.body|startswith("CODEX VERDICT"))]|length'
```

It also cannot submit a formal *request changes* review on a PR opened by the same account — GitHub refuses reviewing your own PR — so on those it falls back to a comment and says so. An unposted verdict is not a missing round; it is a round whose only copy is your stdout, so paste it verbatim and say that is what happened.
- **Follow-up questions especially.** The rounds where you ask Codex to judge your rebuttal are the most valuable half of the record and the easiest to lose, because those runs usually end with "do not apply any fixes and do not post" — so Codex posts nothing itself and the entire exchange exists only in your scrollback.
- **Strip secrets, and say what you stripped.** Tokens, credential paths, anything out of `.env`. Nothing else gets edited out — "this part was noise" is a judgement call the record should not be making.
- Wrap long output in `<details>` so the thread stays readable. Never truncate to make it fit; a shortened reply is a summary wearing a transcript's clothes.

### Iteration step C — YOU evaluate each finding

**This is not a passive apply step.** For every finding from Codex:

1. **Necessity** — Is this a real bug, a style preference, or a false positive? **Reproduce it as a runnable mutation before accepting**, not in your head: apply the exact change the finding describes and watch what the suite says. Reasoning about which cases are safe is the step that fails — in one loop it produced "these two array methods are harmless" (they carried the same alias as the four being fixed) and "`length` returns a number so it cannot mutate" (`length = 0` truncates the array). Both took seconds to disprove by running and neither was going to be disproved by thinking harder. A mental repro is enough only for a finding you are about to REJECT and can point at the code that refutes it.
2. **Blast radius — fix the CLASS this round, not the site.** If Codex flags issue X at one site, search the codebase for the same pattern and fix every occurrence in this batch. A single-site fix that leaves three other copies broken is worse than the original finding, and it is also the most reliable way to spend three more rounds: each surviving copy arrives as a fresh finding in a round of its own. Codex was asked to list every site; that list is a starting point, not the answer — run the search yourself, because the sites it cannot name are the ones that cost the rounds.
3. **Side effects** — Will the suggested fix break callers? Violate a convention? Regress a test? Check before applying.
4. **Gaps Codex missed** — Look at the diff yourself with fresh eyes. Codex's review is your starting point, not your ceiling.
5. **Categorise**: MUST-FIX / VALID-NIT / FALSE-POSITIVE / DEFER-TO-FOLLOWUP.
6. **Write it into the ledger** — one row per finding, before you move to the next one. The ledger is written here, while you still have the evidence in hand; reconstructing it at the top of the next round is how rows become "false positive" with no reason attached, which is what brings the finding back.

Apply MUST-FIX + VALID-NIT in this iteration, **all of them, in one batch** — one round's findings produce one push, never one push per finding.

### Iteration step C-bis — settle disagreements INSIDE this round

For everything you are NOT fixing — FALSE-POSITIVE and DEFER — do not push the rebuttal and wait for the next verdict. **That costs a round per disagreement, and it is the second-largest source of round inflation after trickled findings.** Put the rebuttals in front of Codex now, before you push, in one call:

```bash
codex exec --sandbox workspace-write \
  "Follow-up on PR #<N>, round <k>. Do NOT re-review the PR and do NOT post anything.

   I am declining these findings from your review this round. For each one, say
   ACCEPTED (my reasoning holds) or DISPUTED (with the specific evidence that
   refutes it). Answer only these:

   <one block per declined finding: the finding, my reasoning, and the file:line
    or command output that supports it>

   Reply as plain text. If you DISPUTE any of them, say which single change would
   settle it."
```

Then:

- **ACCEPTED** → the rebuttal is settled. It goes in the ledger as `REJECTED` with Codex's agreement recorded, and it cannot come back as a finding next round.
- **DISPUTED** → you have the counter-evidence *now*, in the same round, and can either fix it in this batch or answer it in this batch. Either way it does not become next round's finding.

This call **does not count as a round** — it produces no verdict and reviews no head. It is the same follow-up the skill has always recommended; what is new is doing it *before* the push rather than after, which is the entire saving. Each disagreement settled here is one that does not come back as next round's finding.

Post the exchange to the PR like any other (*Every Codex exchange goes on the PR*), and post ONE triage comment per round covering every finding and its disposition — not one comment per finding. A reader wants the round's decisions in one place, and so does the next round's Codex.

**Verify the premise, not just the conclusion.** A finding arrives as *claim + reason*, and the two fail independently. Codex can be right that a change is wrong for a reason that is not true — "deleting this breaks historical pricing" was sound advice about a table nothing prices from. Trace the reason to the code before writing it into a commit message or a comment: whatever you accept as a premise becomes the justification a future reader inherits. If the conclusion survives but the reason does not, say so and give the real one.

### The loop converges on YOUR additions, not the original diff

By iteration 3 most findings are about code, comments, tests, and docs **you wrote while responding to earlier iterations**. That is the loop working, and it is also its main hazard: review-driven changes get written under time pressure, land without the scrutiny the original diff got, and carry a false authority because a reviewer asked for them.

Hold everything you add mid-loop to the *same* bar as the original diff:

- **A fix justified by a symptom it cannot affect is not a fix.** Before claiming a change resolves something, confirm the code path from the change to the symptom exists. Two lists with similar names, two tables with similar contents — check which one the surface actually reads.
- **Search for existing coverage before writing new coverage.** Duplicating a weaker version of a check that already exists is worse than adding nothing, because it reads as coverage.
- **State the property your new test asserts, then confirm the implementation agrees.** A test whose comment describes the opposite of what the code does still passes, and the comment is what the next person believes.
- **A new test can encode the wrong rule and pass.** A check written against one configuration can forbid what a second supported configuration requires. Ask which configurations the thing under test serves, and whether the assertion holds in each.
- **Break-verify every fix, including the ones a reviewer asked for** — and confirm the break actually broke something. A patch that silently failed to apply produces a green run that looks like a test gap.
- **A mutation that stops the file COMPILING is not a mutation the test survived.** Inserting a raw backtick into a template literal to test a regex made the suite fail to *load*; the grep counted `Failed Tests` and the run had produced `Failed Suites`, so it read as "the guard has a hole". It did not — checked directly in node, the matcher was 8 for 8. **Read the runner's summary line, not a grep for the word you expected**, and when a mutation must sit inside a string, test the matcher in isolation instead.
- **A guard that asserts `false` everywhere passes just as well when it matches nothing.** `expect(hint.includes(bad)).toBe(false)` is green when `bad` never appears AND when the matcher is broken. So pin the MATCHER in both directions: the shapes it must catch, and the near-misses it must not. The near-miss half is the one that fails when a guard has quietly stopped guarding.
- **The mutation harness edits production code, so check the restore rather than assuming it.** A script that loses its backup — a wrong working directory is the usual way — fails in both directions: a patch that went nowhere reports green and reads as "the test does not catch this", and mutations that accumulate across cases report red and read as "this case is covered". One loop hit that three times. Verify the tree is clean between cases (`git status`, or count the mutation's own marker back to zero), and use absolute paths.

#### The third finding on one symbol STOPS the fixing

This is a gate, not advice, because the advice already existed and loops walked straight past it. Measured over the ledgers: **10 loops have a single symbol named in four or more separate findings** — `curProjDir` in nine, `loadedModuleEdge` in seven, `server.ts` in four across a twenty-round loop whose last eleven rounds were `(req,res,next)=>next()`, then `Router().get(h)`, then `app.use([],h)`, then `app.use("",h)`, then `Reflect.get(app,"use").call(…)`, then `false && h`, then `0 || h`. Each was a real finding. Each fix was correct. The loop was still wrong, because a language always has one more way to say a thing than anyone will list.

**So: when a third finding lands on the same symbol, rule, or function in one loop, stop fixing cases.** Do not apply the third fix. Instead, in that round:

1. Say out loud that the rule enumerates BAD forms, and that this is the third.
2. Re-express it as what is **PERMITTED**, and report everything else. State in the test that it deliberately rejects some safe code, because it does and the next reader deserves to know it was a choice.
3. Put the inverted rule in front of Codex in the same round (*step C-bis*) and ask it for a form that gets past the new one — that question is answerable in one call and it is the whole remaining risk.

The saving is not one round. In the twenty-round loop above it was eleven, and the inversion also closed doorways nobody had listed.

**When the same class of finding keeps arriving, change the shape of the rule, not the number of cases.** A reviewer who finds a new way past your check every round is telling you the check enumerates BAD forms — and a language always has one more way to say a thing than anyone will list. Enumerate what is PERMITTED instead, and report everything else. In one loop that inversion ended four rounds of ban-this-then-ban-that: the rule went from "these array methods are forbidden" to "the batch may appear in these three positions", which deleted two checks and simultaneously closed doorways nobody had thought of (computed access, `.bind`, aliasing). It also fails closed, so the next unimagined form arrives as a red test rather than a silent hole — at the cost of rejecting some safe code, which is the trade you want here and should say out loud in the test.

When reporting, attribute honestly: if an iteration's finding was a defect in your own earlier fix, say so. It tells the human which commits to read hardest.

#### Do not ASK for prose findings, and do not pay a round for one

The claims sweep below is about YOUR prose, swept once. This is about the reviewer's, and the two
pull in opposite directions: a brief that names a "claims" axis every round, and praises the
reviewer for having found false prose before, gets more of it — including the kind that changes
nothing.

Measured on one eight-PR series: **25 rounds, two of them spent entirely on a count in a docblock
and then the same count in a plan file.** Both were filed P3, both turned the round into CHANGES
REQUESTED, and neither changed a line of code. In the same series a prose finding *was* worth its
round — a sentence saying a value was measured at request arrival when it was measured after an
await, which the reviewer itself pointed out "a future extraction can trust and move the measurement
to the wrong side of". That is the line: **would believing this sentence cause a wrong change?**

So:

- severity **N** exists for prose-only findings and does not block the verdict (see *EACH FINDING
  CARRIES FOUR THINGS*). Collect them; fix them once, at the end.
- do not put a standing "claims" axis in a tier-S brief at all. In tier C, word it as *"report a
  sentence that would cause a WRONG CHANGE if believed; file anything merely behind as N"*.
- never tell the reviewer it has found false prose before. It is a request, not a compliment, and it
  is answered.
- **when a count is the finding, fix the class rather than the instance** — delete the count. A
  number in a sentence is checked by a person; a number in an assertion is checked by running. In
  that same series the first count was corrected, and the next round found the second one.

What this does NOT touch: a qualification attached to a "none". Those were real holes six times out
of eight PRs in that series, each reproduced GREEN before being accepted, and they are the most
valuable thing the reviewer produces. Reproduce every one.

#### The claims sweep — every surface at once, or it comes back

**A claim lives in up to four places, and the loop only ever fixes the one it was shown.** Measured over the ledgers: **128 findings across 41 of 88 PRs** are a comment, a test docblock, a PR body or a plan file saying something the code does not do — **the single largest preventable class here**, and 70% of them arrive in round 3 or later, which is exactly where a round is most expensive. Two shapes that cost the most:

- **A false sentence with a second copy.** One loop fixed a docblock, then found the same sentence in the test file next door — a separate round for a copy-paste.
- **A number corrected in one surface only.** One round found **five stale claims in the PR description and four more in the plan file**, all made false by the loop's own earlier rounds. Another spent a round on a plan file still saying "Twenty-eight rounds" after the description had been corrected.

So whenever you touch a claim, sweep all four surfaces in the same batch:

| surface | how to find it |
|---|---|
| the code comment / docblock | the file you are editing |
| the test docblock and test names | `grep` the claim's distinctive phrase across `test/` |
| the PR body | `gh pr view <N> --json body` |
| the plan file, if the repo uses one | `plans/*.md` for this change |

And **the trigger is any change to a measurement, not just a change to prose**: a count that moved, a grid that widened, a mutation re-run. Re-grep the old number as a literal — `grep -rF "84 combinations"` — because the sentence carrying it is rarely where you were looking.

##### A phrase sweep fails at whatever boundary the tool works within

Four sweeps in one session each returned all-clear and each was wrong, at a different boundary:

| the sweep looked for | what it could not see |
|---|---|
| `toLowerCase() === 'true'` | the same expression with **double quotes** |
| `"deploy is a separate optional step"` | the sentence built from **two concatenated string literals**, so no LINE holds it |
| `"the other ~20 readings"` | a comment **wrapped** so the number and its noun are on different lines |
| the phrase that DENIED a gate | seven surfaces that agreed a gate existed and were wrong about **what it holds** |

The practical rules, in the order they cost the most:

- **Search for the NUMBER, never for the sentence.** A number is one token; it cannot span a
  line, a quote style, or a concatenation. List every count in the diff, then read each one.
- **Read the assembled value, not the source.** A tool description built with `+` is a string
  only at runtime, so a test over the REGISTRY sees it and a grep over the file cannot. That is
  what finally stopped the recurrence above, and it is the reason to prefer a test to a sweep
  for anything agent-facing.
- **Fixing a claim exposes the next claim.** Once every surface agreed a gate existed, the next
  round found they disagreed about WHICH operation it held. A sweep for the old wrongness is
  blind to the wrongness the fix reveals — so after a claims sweep lands, ask what the corrected
  sentences now assert, and check that.

**Prefer a claim that cannot go stale.** Three rounds in one loop went to three different wrong sentences in one docblock, each a universal negative ("no field is examined"), each falsified by one more shape. The fix that ended it was to enumerate what happens to each of the seven fields, one line each, checkable against the code beneath. A negative quantified over everything unsaid has no last case; a list does.

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

**And run what CI DOES — the whole list, not the ones you remember.** A loop reached LGTM and CI went red on the next push, on a `typecheck:tests` job that had been in the list all along. Locally only `tsc --noEmit` had run, and in that repo it reads a config whose `include` is `src/**` and has never looked at the test tree. The project's own CLAUDE.md said so.

That gap is worse than it sounds, because **vitest transpiles rather than typechecks**: the wrong call ran, passed, and the test was green while checking half of what its name claimed — `f()` on a function whose argument selects *which* of two things to check. A green suite is not evidence that the test called what it meant to. Derive the local gate list from `gh pr checks`, and treat any typecheck job that covers tests as non-optional.

**A red CI on the head you just declared clean is a finding, and it may not be yours.** Read it before assuming either way: on that same PR one of the two failures came from `main` — a guard and a file that violates it merged from different PRs, so their combination was red on main itself. Check whether the offending file is byte-identical to `origin/main`'s (`git diff origin/main HEAD -- <path>`) before spending a round on it, and say plainly in the PR when you fixed someone else's breakage to get your own branch green.

**A red check is not automatically yours.** Read the failure before chasing it: a timing assertion in a file the PR does not touch, failing at 3.06 against a threshold of 3, is a loaded runner. Say so on the PR — an unexplained red tick on an approved security PR teaches everyone that this job's failures are ignorable, right up to the one that is not.

**Establish a pre-existing-red baseline ONCE, in ledger row 0.** Measured: **24 findings across 16 PRs** are somebody re-deriving that a red check or a flagged alert predates the branch, and one loop did it twice for the same eight CodeQL alerts, nine rounds apart. Record it once, with what makes it checkable rather than a memory:

```
row 0b | CodeQL: alerts #297-301 (2026-07-21), #371-372 (08-09), #870 (08-16) are open on
         refs/heads/main and pre-date BASE_SHA. Files: uiProjectService.ts, projectContainment.ts
         — neither in this diff. Re-check ONLY if the diff grows into either file.
```

Then a later round answers it by lookup. **The one thing that must be re-checked rather than inherited is the condition**: if `main` moved a lot, or the diff now touches a file the baseline named, re-derive it and say you did — one loop re-verified for exactly that reason and was right to.

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

### Iteration step F-bis — dispatch the next Codex round IMMEDIATELY. Do NOT wait for CI.

**Push, then call Codex in the same breath.** CI and the review answer different questions and
share no dependency, so running them in series spends the longer of the two twice.

Measured on this repo: `test (22.x)` runs **17–21 minutes** and `typecheck (22.x)` **14**, and
`e2e` is only scheduled after those. A Codex round is **3–8 minutes**. Serialised, every round
costs a CI cycle before the review even starts; in parallel the review is usually back *before*
CI, so its findings are in hand when the checks land and the next push carries both.

- **Dispatch Codex the moment the push lands.** Do not wait for a single check.
- **Run them in the BACKGROUND, and run several PRs' rounds at once** — each `codex exec` is
  independent. Three concurrent reviews on three PRs is normal and correct.
- Always `< /dev/null` and `timeout` (see *Setup step 8*) — a backgrounded call without them
  hangs on stdin forever and notifies nobody.
- **CI still gates the MERGE.** Parallel dispatch changes when you *learn* things, never what
  may ship: the exit bar is unchanged, and `Merge only when both reviewers are OK and CI is
  green` still holds.

**The failure this prevents is not slowness, it is a finding arriving too late to be cheap.**
On 2026-09-15, three reviews were dispatched against heads whose CI was still running. One
returned a P2 that **two prior reviewers — a Claude round and the coordinator — had both
missed**: a guard asserting `readFileSync(...).includes(seed)`, which a rename had updated
correctly and which was therefore satisfied **by a comment**. Codex found it by replacing the
production seed with `false` while leaving the expression in a comment and watching the guard
stay green. Had that round waited on CI, the finding would have landed after the PR was queued
for approval.

**When CI is the thing you are waiting on, say so and keep working.** A red check is a finding
like any other — fold it into the next round rather than letting it stall the loop. And read a
check by its conclusion, not by the run being over: a queued job is not a passing one.

### Iteration step G — decide whether to loop

Parse the Codex verdict marker from step B:

An iteration is **clean** only when BOTH reviewers pass on the same commit: Codex posted `CODEX VERDICT: LGTM`, and your own evaluation of that same head has no MUST-FIX left. Anything else is not clean.

**An `LGTM` with no `FINDINGS COMPLETE` line is not clean either.** It is a verdict on an unknown fraction of the diff, and accepting one is how a round-three finding becomes a round-seven finding. Do not spend a round on this: re-ask in a follow-up call — *"which parts of the diff did you not reach, and is this every finding you have?"* — the same non-round shape as step C-bis. Only the answer counts.

Before counting the round, apply *Promote on contact*: a P1 or P2 finding this round deepens the next round's prompt. It does not add a round on its own; the push does.

- **Clean, and it was on a head you pushed nothing to** → exit (see *Exit* below).
- **Clean, but you pushed commits this round** → not an exit. This is the one round nobody can save: the fix has never been reviewed. Push nothing, re-run, get the verdict on the unchanged head.
- **Not clean** → next iteration. This holds whether you applied the finding or declined it: a rebuttal you never put back in front of Codex is a disagreement you awarded to yourself. Post the reasoning, push, and run the next round so it can answer — it may accept, or it may show the rebuttal is the thing that is wrong.
- `CODEX VERDICT: LGTM` but you found real issues it missed → **not clean**. Fix them, push, loop.
- No verdict marker → treat as `CHANGES REQUESTED`, note the protocol failure, retry with a reminder.
- **The next iteration is a multiple of 5** → append the checkpoint block to that round's step-A prompt (*Every fifth round*). It is not a separate call and not a separate round.
- **The round established the PR should not exist** → stop the loop and take the close path (*Closing the PR is a legitimate outcome*). This outranks a pending verdict: an LGTM on a change that should not ship is not a reason to merge it.

### Exit: ONE clean round on the head that will merge

**The bar is one clean round, at tier S and tier C alike.** Tier T takes no Codex round at all, two self-review passes instead (see *Tier T* above).

Three things have to be true of that round, and they are what a verdict actually means:

- **It covers the head that will merge.** A fix pushed in response to round N is reviewed for the first time in round N+1, so a clean round you pushed commits in never ends the loop. The round that fixed things is not the round that ends it; the quiet round after it is.
- **Both reviewers, same commit** — Codex `LGTM`, and your own evaluation of that head with no MUST-FIX left.
- **The round was complete** — the axis table is answered line by line and `FINDINGS COMPLETE` is there.

The bar used to be two consecutive clean rounds at tier C. **The user dropped it on 2026-08-15**, mid-loop on PR #1796, and the reason they gave is the argument against restoring it: that loop ran five rounds and found three real defects, of which **two were inside guard tests written during the loop itself**. The original change was read five times and drew no finding. Past the point where the diff has been read properly, an extra round mostly reviews the loop's own output — and the cheap way to stop finding bugs in review-driven code is to write less of it, not to review it once more.

That is a judgement about one repository's PRs, not a proof. If a project wants the second round back, it belongs in that project's `CLAUDE.md`, which overrides this file.

So the depth went into the round instead: the completeness contract, the axis table, the class-wide fix, and the in-round rebuttal. **A second clean round is not banned — it is what you get for free whenever a round finds anything at all**, since the fix has to be reviewed. It is simply no longer owed on a round that found nothing.

### There is no cap

**The loop runs until one clean round lands on the head that will merge, however many rounds that takes.** Nothing in this skill sets a round budget, and none of the round-reducing devices above is a reason to stop early — they exist to make rounds unnecessary, never to skip a necessary one. A one-line diff that cannot converge in five rounds is not a small change — it is a disagreement about something the diff does not show, and it promoted itself out of tier S at the first finding anyway. Stopping there hides exactly the thing worth finding.

What the rounds actually produce is the argument. In one loop, rounds 1–4 each found a real defect and three of them were defects in fixes made *during* the loop: an equivalence fix that was not equivalent, a flag set on the wrong event so the bug it fixed came back by another door, a rule patched at three exits when a fourth existed, and a comment claiming an invariant the code did not have. A cap of five would have merged the fifth.

Read that list again for what it says about round count, because it cuts both ways. Three of the four were defects the loop **introduced**, and the fourth — a rule patched at three exits when a fourth existed — is exactly what *fix the CLASS, not the site* now catches in round 1. The way to have run that loop in three rounds was never to review less; it was to fix the class the first time and to write less review-driven code. That is the whole thesis: **rounds are removed at the source, not at the exit.**

**Do not stop mid-loop to ask the user whether to continue.** Asking is how a two-reviewer protocol quietly becomes a one-reviewer one. "This is taking a while", "we are at round 12", and "the remaining findings look minor" are not reasons to stop — they are the loop working.

The loop ends early for exactly these, and nothing else:

1. **A decision that is genuinely the user's** — see *Stopping for the user* below.
2. **`codex exec` errored out** and does not recover on a retry. Record it and hand back.
3. **A semantic merge conflict** that step E says to escalate.

If the loop passes ~10 rounds without a clean one, say so and keep going — but read the pattern out loud first. Findings that keep arriving in the same shape mean the fix enumerates bad cases instead of stating the rule (see the note above on inverting to what is PERMITTED); findings that arrive in new shapes each round mean the change is bigger than the PR admits and may want splitting — or closing.

### Stopping for the user

The exception to "do not stop" is narrow: a question the code cannot settle, where proceeding either way would be guessing at the user's intent. Concretely:

- **The loop concludes the PR should be closed** (see below). Closing someone's work is the user's call, so you bring the recommendation and the evidence, not the action.
- **The merge go-ahead**, as it always was.
- **A finding whose resolution is a product or policy decision** — which of two valid behaviours is wanted, whether a breaking change is acceptable, whether a deferred item blocks the release. Post the options and what each costs; do not pick one and call it converged.
- **A conflict whose correct resolution is not derivable** from either side's history.

Everything else — including a finding you and Codex flatly disagree on — is resolved inside the loop by putting the rebuttal back in front of Codex, not by escalating. When you do stop, say which of these it is, and what specifically you need decided.

### Every fifth round, both reviewers re-read the whole PR — in the SAME call

Findings compound. By round 5 the conversation is mostly about the loop's own output, and a loop can converge beautifully onto the wrong thing — a rule refined for four rounds that should not exist, a test suite grown around a behaviour the PR was never supposed to have. Round-by-round review cannot see this, because each round only ever looks at the delta.

So at iterations **5, 10, 15, …**, both sides also look at the PR *whole*. **This rides inside round 5's normal call rather than costing a round of its own** — the checkpoint questions are appended to the step-A prompt and answered in the same top-level comment, after the axis table. A PR that has reached round 5 is not one to spend an extra round on asking whether it should exist; asking is cheap, and a separate call is the expensive way to ask.

**Your half** — re-read the full diff (`git diff "$BASE_SHA"...HEAD`) against the PR's stated goal, as if seeing it for the first time:

- Does the diff still do what the PR title and body say? If the body now describes a different change, the body is stale or the PR has drifted — name which.
- How much of the diff is original work versus loop-driven additions? A PR that is now majority review-response is a PR whose centre of gravity moved.
- Is anything in it there only because a reviewer asked, and no longer justified on its own? Removing it is a legitimate outcome.
- Would you approve this diff if it arrived fresh today, with none of the argument attached?

**Codex's half** — appended to round <k>'s step-A prompt, after the axis list and before the verdict instructions:

```text
   ── CHECKPOINT — round <k>, in addition to the review above ──────────────────
   This is round <k> of a review loop. As well as this round's findings, read the
   PR whole — title, body, full diff, and the ledger — and answer:

   1. Does the diff still accomplish what the PR says it does?
   2. Has the review conversation drifted onto something other than the PR's purpose?
   3. Is any part of the diff there only because a reviewer asked for it, without
      standing on its own merit?
   4. Should this PR be split, redirected, or CLOSED rather than converged? Say so
      plainly if yes, with the reason.
   5. What is the single largest remaining risk in this change?

   Put these answers in the same final comment, under a heading
   'CODEX CHECKPOINT: round <k>', after the axis table. Still give a verdict marker.
```

Post both halves to the PR. Because it rides in the normal call, the round **still produces a verdict and still counts** — a clean round 5 with a checkpoint that finds no drift is an exit like any other. Only what the checkpoint *finds* can cost a round: if it changes direction (scope cut, split, close), the clean-round count starts over, because what the earlier verdicts approved is no longer what will merge. (A PR that reached round 5 is tier C by then in all but name; if it is still labelled S, that label is stale — promote it, which deepens the axis list from here on.)

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
- Tiering changes how DEEP a round is, never how many rounds you owe — the bar is one clean round at every tier that runs Codex at all. There is no lighter review, no skipped local checks, and no unverified finding at tier S or T.
- Fewer rounds is bought by making a round COMPLETE, never by making it shallower. Every device in *Rounds are the cost* removes work that would have been repeated; none of them removes work that would have been done. If a shortcut would make the loop shorter by leaving something unreviewed, it is not one of them.
- A round that produced a push always owes another round. That one is not negotiable and no batching removes it — an unreviewed fix is unreviewed however good the reason for it was.
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

## Reporting to the user — once, at the end

**The loop is not a conversation.** A status message per round makes an eight-round loop eight messages the user has to read and cannot act on, and each one invites the "shall I stop?" the protocol does not want. The record already exists in three durable places — the PR comments, `ledger.md`, and the per-iteration state files — so a running commentary adds nothing but turns.

So: **write the per-round line to the state file, and say nothing to the user** until one of these:

1. **The loop ends** — the full report below.
2. **One of the three early-exit conditions fires** (user decision, `codex exec` dead, semantic conflict). One message, saying which and what you need.
3. **The shape of the work changes** — the tier is promoted into the C list, the checkpoint recommends splitting or closing, or CI has been red for reasons outside the PR. One line, no question attached.

Nothing else. Not "round 4 done", not "still going", not "this is taking a while". If the user asks where it is, give the state-file line for the current round and carry on.

The per-round line, written to `/tmp/codex-cross-review-<N>/iteration-<k>.json` rather than to the user:

- Round `<k>`, tier `<T/S/C>`
- Codex verdict: `LGTM` / `CHANGES REQUESTED (<N> issues)`, and whether `FINDINGS COMPLETE` was present
- Findings by severity, and how many were fixed / rejected / deferred
- Whether anything was pushed (which is what decides if another round is owed)
- CI status at this moment

Final report on merge: PR number, merge commit SHA, tier it exited at (and any promotion, with what caused it), total rounds and what each one found, notable disagreements if any. **If the loop ran more than three rounds, say what made it long** — trickled findings, a class fixed one site at a time, a disagreement that took two rounds to settle. That sentence is the only feedback this skill gets about whether its round-reducing devices are working.

Final report on a close recommendation: which reason applies, the findings that established it, what replaces the PR, and Codex's judgement of the recommendation.
