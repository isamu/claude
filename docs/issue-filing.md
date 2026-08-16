# Filing a GitHub issue that is worth someone's day

Referenced from CLAUDE.md → GitHub Issues & PRs. Read before filing an issue for a bug or a feature.
The `/issue-draft` skill does this interactively; these are the rules it enforces, and they apply
just as much when filing by hand.

## Before filing

- MUST first check the **20 most recent issues** (`gh issue list --limit 20 --state all`) and say
  whether anything overlaps. A duplicate splits the discussion, and a near-duplicate is usually a
  sign the real problem is bigger than the one being filed.
- MUST trace the defect through the actual code before filing, not from the shape of the code. An
  issue argued from shape is how a fix gets built for a case production cannot reach (orion #1582:
  two "races" derived from a loop's guard order, both unreachable at the sole call site — the fix
  PR was closed and the issue withdrawn).
- MUST grep for the **producer** of the input, not only the handler. The handler is the easy half
  to find and the useless half to reason from. (orion #1634: the issue quoted the exact handler and
  diagnosed it correctly — an id assigned inside a state updater — while nothing ever sent the
  frame that reaches it. The body opened "`thinking_start` opens the bubble", *assuming* the frame
  arrives. Correct fix, unreachable path.)
- If reachability cannot be established, **MUST NOT file** — record the observation in the PR that
  surfaced it, with what was checked and what could not be. An unreachable issue costs someone a
  day; a paragraph in a PR costs nothing and keeps the finding.

## In the body

- MUST say whether the symptom was **observed or inferred**, and MUST name what could not be
  established. Silence reads as "checked and fine".
- **Regression or long-standing gap** is cheap to settle and changes who fixes it and how urgently:
  `git log -S'<the string that would have to exist>' -- <path>`, then read that commit's
  **message**. A deliberate deletion usually says so, and that sentence is the difference between
  "someone broke this" and "this never worked" (orion #1662: the obvious suspect had removed dead
  branches and said so in its own message).
- When the remedy turns on a **number** — a threshold, a limit, a timeout — MUST measure both
  populations it separates **including their worst observed readings**, and MUST read the code
  being fixed before proposing the fix. Means mislead: healthy 1.92–1.99 against broken 3.84–3.91
  says "move the line to 3.5", while a healthy reading of 3.57 already recorded in the repo says no
  line fits at all. A remedy that only makes the failing path slower is not a remedy.

## Partial-scope PRs MUST NOT use closing keywords

When a PR addresses only PART of an issue (one phase, one of several targets), MUST NOT write
`Closes #N` / `Fixes #N` / `Resolves #N` in the PR body. Qualifying the sentence does not help —
GitHub parses the keyword and ignores the prose around it, so `Closes #2583 の google 側` still
closes the whole issue on merge. Use a bare `#N` reference or `refs #N`, and close the issue by hand
once every phase has shipped.

If it happens anyway: reopen the issue immediately and post a comment naming what shipped (with the
merge commit) and what remains — a silently-closed issue is how the remaining phases get lost.
(mulmoclaude #2583: the google-side PR closed an issue whose spotify half was untouched.)
