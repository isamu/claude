# Working one backlog issue in parallel

The rules are in [`CLAUDE.md`](../CLAUDE.md) → *Working one backlog issue in parallel*. This is what
they cost.

The setting throughout is a real one: `tne-ai/orion` #1802, a `sonarjs/cognitive-complexity` backlog
of 65 entries in the ≤20 band, worked as one-function-per-PR tranches, four to five agents at a time,
each in its own git worktree, each producing a PR that then went through a Codex review loop before
merging. Over a day it took the band from 65 to 32 and the repo total from 163 to 131. Everything
below went wrong at least once along the way.

## The coordination failures

### A merged PR is invisible to every check you were running

Each tranche opened with a collision check: `gh pr list --state open`, then `gh pr diff <n> --name-only`
on the plausible ones. One tranche picked `backend/src/ktap/extract.ts`, worked its casts, and found
at the end that a PR had **merged an hour earlier** taking that whole area from 50 casts to 0. It was
not in the open list, because it was closed. It was not in the working tree, because the branch had
been cut before the merge.

The missing command is `git log origin/main --oneline -20 -- <file>`. It costs nothing and answers
the question the open-PR list cannot.

### "Stop and report" needs a scope, or an agent has to invent one

A later tranche found an open PR editing the same file — the last 120 lines, ~400 lines from anything
it was touching. Its brief said "if an open PR touches this file, stop and report", so it had to
decide alone, mid-flight, whether that instruction was aimed at duplicated work or at any overlap at
all. It chose to proceed and flagged the judgement, which was right, but nothing in the record said
who had arrived first or what either party had claimed.

Hence the claim: a comment on the issue, before the work, naming the **function or line range**. Two
tranches at opposite ends of one file is fine. Two in one function is not. Only a specific claim can
distinguish them, and the claim is also what a reviewer reads to see whether the two PRs need
ordering.

## The reachability failures

### `knip` answers a different question than the one you are asking

The gate after the first orphan (`clawBinary.ts` — never imported in the repository's history, and an
extraction was written for it before anyone checked) was `yarn knip --include files` before
extracting. It does not report:

- **`FileViewer.tsx`**, whose only remaining reference is a *type-only* import in `App.tsx`. A type
  import keeps a file "used" at file granularity while the component is never rendered — the ref that
  import typed had been a permanent `null` since the commit that removed the last `<FileViewer />`, so
  three `.refresh()` call sites had been no-ops for two months.
- **`folderTree.ts`, `fileViewerAction.ts`, `FolderPickerModal.tsx`**, whose only readers are their own
  `*.test.ts(x)` — and `knip.jsonc` declares test files as *entry points*, correctly, because a helper
  only a test uses is not dead. The consequence is that a module whose last non-test reader
  disappeared can never be reported.

So knip answers *"is this file imported"*. The question is *"can this code be reached"*. For a `.tsx`
target the difference is one grep: `grep -rn "<ComponentName"`.

Two tranches had already extracted tested rules out of that unrendered component before anyone
noticed. Both are now importable only from their own test files.

## The measurement failures

### A leftover `node_modules` symlink makes the repo lint twice

Agents symlink the primary checkout's `node_modules` into their worktree rather than installing —
correct, and much faster. If the symlink survives the agent, the *next* `npx eslint .` from the repo
root walks `.claude/worktrees/agent-*/` too and lints two or three whole copies of the repository.

Observed twice. The first time it reported 5,130 errors and a nonsense rule breakdown; the second
time it reported the backlog as **zero remaining**, which is exactly the shape of a result nobody
questions. Remove the symlinks in the agent, and pass `--ignore-pattern '.claude/**'` when measuring
from the root.

### A stale shared Prisma client is not your change

Several worktrees share one `node_modules`, hence one generated Prisma client. When `main` adds a
model, every worktree's `tsc` reports errors in files nobody touched. Three separate tranches hit
this; the correct handling — which one of them worked out and the others then copied — is:

- do **not** run `prisma generate`, because it mutates state the other tranches are using;
- confirm the failure reproduces with your own change reverted;
- say so in the report, and note that CI regenerates the client before typechecking.

### A timeout at load average 120 is the load

Five agents plus three Codex reviews took the machine to a load average of 120 on a laptop. Under
that, `vitest`'s 10s hook timeout and 15s test timeout fire on tests that have nothing to do with any
change in flight — and they fire on a *different* file each run, which is the tell. Re-running the
same three files standalone with `--testTimeout=120000 --hookTimeout=120000` turned 3 failures into
37 passes.

This is why the load check is a rule rather than a courtesy: every agent added past the machine's
capacity does not merely go slower, it makes the *existing* agents produce failures that then have to
be re-run and re-judged, one at a time, by hand.

### A concurrent sweep can put a mutation in your file

A mutation sweep edits production code dozens of times and restores it after each. `git status` cannot
distinguish your mutation from a sibling's. One agent found a guard it had reasoned was redundant
scoring 603 differences, and a `diff` against a saved copy showed a mutation **it had never applied**.

The fix is cheap and belongs in every sweep: assert the file matches a pristine copy **before**
applying each mutation, not only after restoring it. A stray mutation left in the tree reads as *more*
coverage — the next case scores against an accumulating edit — which is the direction that does not
announce itself.

## The verification failures

These are not specific to parallelism, but parallelism is where they surfaced, because five agents
running the same method produce five samples of its weak points.

### A harness over a private function can score the rule against itself

A tranche comparing a module-private collector reported 0 differences three times over. The collector
was not exported, so the harness had *reimplemented* it — and was comparing that reimplementation with
itself. Only a second harness entering at the caller's exported boundary produced counts that moved.

If the function under comparison is private, enter at the caller as well. A zero from a harness that
cannot reach the code is indistinguishable from a zero that means equivalence.

### Removing a cast changes behaviour, and query parameters are where it shows

`?status=a&status=b` arrives at Express as an **array**. A `status as string` cast passed that array
straight through — to Prisma, to `parseInt`, to whatever came next. Replacing the cast with a `typeof`
guard answers `undefined` instead, so a request that used to 500 now succeeds *unfiltered*, and a
`?limit=5&limit=6` that used to mean 5 now means the default.

There is no version of "remove this cast" that preserves the old answer, because the cast is what
produces it. The resolutions that are honest: keep the cast and say why, or make the change and **put
it in the PR title**, since the body is not what survives being read from a PR list. What is not
honest is a title that says refactor over a diff that changes an API.

### For genuinely dead code, "restore it and watch a test fail" cannot be satisfied

A reviewer asked a deletion to be verified the way the method verifies removals: put the branch back,
watch something go red. The deleted branch was `if (ref.current) { … }` where `.current` was
permanently `null`. Restoring it restores a statement that does nothing — no test can separate the two
versions, and if one could, the branch would have been reachable and the deletion wrong.

What *can* be pinned is the invariant the deletion leans on. There, a no-op `() => {}` had to stay in
`App.tsx` because a listener's effect opens `if (!onFileWritten) return` — the prop is a switch, and
deleting the empty function as dead weight would silently switch the listener off. Two tests, one
mutation, and the thing a future cleanup would break is now guarded.

## The pattern worth naming

Across roughly twenty tranches, the single most common review finding was **not a defect in the code**.
It was a defect in the *sentence explaining why the code is safe*: an equivalence claimed for "every
shape" that held only for the shapes one caller can produce; a `[object Object]` that nothing
stringified; a "the only place this is observable" that was not observable there at all.

The comment justifying a change is the least-tested artefact in a PR, and it is what the next reader
believes. Check it against the code as carefully as the code.
