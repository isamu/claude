# How big a PR should be when AI reviews AI

Referenced from CLAUDE.md → GitHub Issues & PRs. Applies whenever the reviewer is another model —
`/codex-cross-review`, `/code-review`, the GitHub bots — which is now the common case.

## The unit is one intent, not one size

Human review optimises for "can somebody read this in half an hour". A model does not tire at line
600, so that is the wrong constraint. What a model actually needs is a **coherent context boundary**:
every hunk in the diff answerable against the same question.

So the unit is:

> **One feature, one set of invariants, one PR.** Changes that are right or wrong *together* belong
> together. Anything that could be merged or reverted on its own belongs apart.

A schema change plus the service plus the route plus the tests is **one** PR when it is one
end-to-end intent — "add user invitations" — and a reviewer judges it better whole than in four
pieces, because three of the four pieces mean nothing alone.

The PR that wrecks review is not the long one. It is this one:

```
login bug fix + logging improvement + package upgrade + dead code removal + API rename
```

Short, and hostile to review: the reviewer's judgement axis changes at every hunk, and quality drops
on all five.

**Line count is a symptom, never the rule.** A 2000-line generated migration can be fine. A 150-line
change touching auth, caching and concurrency is dangerous. Ask instead: can the reviewer hold one
mental model across the whole diff?

## Size interacts with the review loop, so there is still a ceiling

A convergence loop does not merely review the diff — **it generates new code**. By the third
iteration most findings are about the fixes written in response to earlier iterations, and those land
under time pressure with less scrutiny than the original diff got.

Measured on one secret-scanning PR in this workflow: 5 iterations, 9 findings, and **4 of the 9 were
defects in fixes written during the review** — including a patch that silently failed to apply
because the formatter had reflowed the function since the match was written.

That is the real argument for a ceiling under AI review, and it is not about reading speed: a bigger
PR takes more rounds, more rounds means more mid-loop code, and mid-loop code is the least-reviewed
code in the change. Splitting caps the compounding.

## The contract matters more than the size

A model can write thousands of lines. What decides whether that is safe is not the diff — it is
whether the reviewer was told **what counts as correct**. Give it, in the PR body, before any hunk:

| | |
| --- | --- |
| **What changed** | one paragraph, in intent terms |
| **Why** | the problem, not the patch |
| **Expected behaviour** | what should now be true |
| **What must NOT change** | the invariants this is not allowed to break |
| **Deliberate decisions** | choices that look wrong without the reason — say the reason |
| **Evidence** | what was run, and what it showed |

CLAUDE.md already mandates **Summary** and **Items to Confirm / Review** at the top of every PR. That
is this contract's first half; the rest of the table is the other half.

**Name the deliberate decisions explicitly.** In practice this is what stops a reviewer re-finding a
choice you already made on purpose and arguing with it for two rounds. A PR that diverged from its
own reference implementation on an exit code said so in the first line of Items to Confirm — and the
review engaged with the reason instead of reporting the divergence as a bug.

**Evidence beats claims.** "355 tests pass" says almost nothing; a suite that has never gone red
proves nothing about the fix. "Reverting this fix fails 4 tests, and the real linter goes from 2
errors back to 0" is a contract term a reviewer can check.

## Ask before opening, and ask for the revert test

Before opening a PR, ask yourself:

> Would splitting this produce independently testable and reviewable units? If yes, propose the split
> first.

The implementer is the worst judge of this, having just built one coherent mental model of the whole
thing — everything looks connected from inside. So do not ask "does this feel coherent". Ask the
mechanical question:

> **Which parts of this could be reverted on their own without breaking the rest?**

Every answer is a PR that should have been separate. This catches the mixed PR above, which *feels*
like "a bit of tidying" from inside and is five unrelated intents from outside.

## Give the two models different jobs

Prompting them identically wastes the second one. The reviewer's brief is not "check this over":

> Assume the implementation may be plausible but wrong. Review against the stated intent, the
> invariants, the tests, and the surrounding code. Do not optimise for style. Prioritise correctness,
> regressions, security, and missing cases.

And the corollary that keeps the loop honest: **declining a finding is not an exit.** A rebuttal the
reviewer never sees is a disagreement awarded to yourself. Post the reasoning and run another round
so it can answer — sometimes the rebuttal is the thing that is wrong.
