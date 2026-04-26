---
description: Spin up a MulmoClaude dev server pointing at a specific PR's branch, generate concrete manual test steps from the PR body + diff, show them in chat, and post the same steps as a PR comment. Use when a PR involves UI changes that need eyeballing before merge.
---

# PR UI Test — start a dev server + generate test steps

## When to use

The user hands you a PR URL or number for a UI change and wants
to manually click through the feature. This skill takes over the
boilerplate: checking out the branch into an isolated worktree,
starting the dev server, and writing the test plan into both
chat and the PR comment thread so a future reviewer can follow
the same steps.

Skip this skill when:

- The PR is server-only / docs-only / tests-only — no UI to click
  (read the diff first; if no `src/**.vue` / `src/**.ts` touched,
  the user is asking for a different skill).
- The PR is already merged / closed.

## Inputs

PR URL (`https://github.com/<owner>/<repo>/pull/<N>`) or just `<N>`
if the cwd is a clone. If only `<N>`, infer owner/repo from
`gh repo view --json nameWithOwner`.

### Default: no argument → infer from current directory

If invoked with no PR number / URL, derive the PR from the
current git context. Common case: the user just pushed a UI
branch and is sitting in its working tree.

```bash
OWNER_REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Prefer an OPEN PR for the current branch (the active iteration);
# only fall back to all-states if nothing's open.
PR_N=$(gh pr list --repo "$OWNER_REPO" --head "$BRANCH" --state open \
       --limit 1 --json number --jq '.[0].number')
if [ -z "$PR_N" ]; then
  PR_N=$(gh pr list --repo "$OWNER_REPO" --head "$BRANCH" --state all \
         --limit 1 --json number --jq '.[0].number')
fi

if [ -z "$PR_N" ]; then
  echo "No PR found for branch '$BRANCH' in $OWNER_REPO." >&2
  echo "Run with an explicit PR number, e.g. /pr-ui-test 805" >&2
  exit 1
fi
```

Surface the inferred PR number to the user before checking out
the worktree — the user can abort if it picked the wrong PR.

If the inferred PR is **already on the current branch's tip**
(no diff between `origin/$BRANCH` and the PR's head), you can
shortcut step 2's worktree setup and run the dev server directly
in the current working tree instead of cloning into
`~/ss/llm/mulmoclaude-pr-<N>`. Confirm with the user before
doing so — the worktree is safer when in doubt.

## Setup

The user's primary clone is at `~/ss/llm/mulmoclaude` (or wherever
their working tree is — confirm with `git rev-parse --show-toplevel`).
Don't touch their main checkout — switching branches under them
clobbers in-progress work. Use a git worktree.

### Steps

1. **Confirm the PR is OPEN.**
   ```bash
   gh pr view <N> --json state,headRefName,title
   ```
   If `state !== OPEN`, stop and tell the user.

2. **Resolve a worktree path.** Convention:
   `${HOME}/ss/llm/mulmoclaude-pr-<N>`. If the dir exists already,
   reuse it (might be from a previous test on the same PR); if
   the worktree exists but points at a different branch, fix:
   ```bash
   git -C ~/ss/llm/mulmoclaude worktree remove ~/ss/llm/mulmoclaude-pr-<N>
   ```
   then re-add. Otherwise:
   ```bash
   git -C ~/ss/llm/mulmoclaude fetch origin
   git -C ~/ss/llm/mulmoclaude worktree add ~/ss/llm/mulmoclaude-pr-<N> origin/<headRefName>
   ```

3. **Install deps in the worktree.**
   ```bash
   cd ~/ss/llm/mulmoclaude-pr-<N>
   yarn install --frozen-lockfile
   ```
   This is the slow step (~30s on a warm cache, ~3min cold).
   Surface a "installing deps…" line so the user knows.

4. **Check default ports are free.** The server binds 3001 (with
   auto-fallback if `PORT` not explicitly set), Vite binds 5173.
   `vite.config.ts` proxies `/api` → `localhost:3001` *hardcoded*,
   so if you change the server port the frontend can't reach the
   API. Conclusion: assume default ports.
   ```bash
   if lsof -i :5173 > /dev/null 2>&1 || lsof -i :3001 > /dev/null 2>&1; then
     # Surface to user with the conflicting PIDs and ask whether to
     # kill them. Don't kill silently.
     lsof -i :5173 -i :3001
   fi
   ```
   If conflict, ask the user: "既存の dev server を止めますか?" — on
   yes, `pkill -f "npm run dev|tsx server/index.ts|vite"` (be
   explicit; don't kill arbitrary node processes).

5. **Start the dev server in the background.**
   ```bash
   cd ~/ss/llm/mulmoclaude-pr-<N>
   nohup npm run dev > /tmp/mulmoclaude-pr-<N>.log 2>&1 &
   echo $! > /tmp/mulmoclaude-pr-<N>.pid
   ```
   Use the Bash tool with `run_in_background: true` instead of
   `nohup &` if the harness supports it — that way the
   notification surfaces when the process exits unexpectedly.

6. **Wait for Vite to print "ready".** Vite logs
   `VITE v… ready in Nms` once the dev server is serving. Use
   the Monitor tool with an until-loop on the log file:
   ```bash
   until grep -q "ready in" /tmp/mulmoclaude-pr-<N>.log; do sleep 1; done
   echo "READY"
   tail -3 /tmp/mulmoclaude-pr-<N>.log
   ```
   Cap with a 60-second timeout — if the server hasn't printed
   ready by then, dump the log and stop.

7. **Generate the test plan.**

   Read in priority order:

   a. The PR body's `## Test plan` section if present (steps
      already written).
   b. The diff (`gh pr diff <N>`). Spot-check the first 200 lines
      and the file list to figure out what UI surface is touched.
   c. The PR title — gives the feature name.

   Compose a numbered sequence:
   - 1 happy-path action (open the feature, confirm it appears)
   - 1 interaction (click / type / select)
   - 1 edge case the diff invites (empty state, error path, locale
     swap, etc.)

   Keep it tight. ~5 steps total. The user is doing the test —
   they want concrete instructions, not essays.

   Each step: one sentence + which page / which element to find
   it on (`/chat`, `/wiki`, `/files`, …).

8. **Show the user the test plan.**

   Format:

   > ## UI test setup ready
   >
   > - **URL**: http://localhost:5173/
   > - **Worktree**: `~/ss/llm/mulmoclaude-pr-<N>`
   > - **Server PID**: `<pid>` (kill with `kill <pid>` when done)
   >
   > ## Test steps
   >
   > 1. <step>
   > 2. <step>
   > 3. <step>
   > 4. (regression) <step>
   > 5. (cleanup) Stop the server: `kill $(cat /tmp/mulmoclaude-pr-<N>.pid)` and `git -C ~/ss/llm/mulmoclaude worktree remove ~/ss/llm/mulmoclaude-pr-<N>`

9. **Post the same steps as a PR comment.**
   ```bash
   gh pr comment <N> --repo <owner>/<repo> --body "$(<test-plan-here>)"
   ```
   Wrap the body in a hidden HTML comment so a future invocation
   recognises it: `<!-- pr-ui-test:auto-generated -->`. If a prior
   auto-generated comment exists, replace via `gh api PATCH`
   instead of stacking duplicates.

## Don'ts

- Don't switch the user's main checkout to the PR branch. Always
  worktree.
- Don't kill processes you didn't explicitly identify. `pkill`
  patterns must match dev-server names only (`npm run dev`,
  `tsx server/index.ts`, `vite`).
- Don't claim the test passed — you only set it up. The user
  runs the actual click-through.
- Don't post the test plan as a PR comment unless the user
  explicitly asked you to (this skill's contract). For other
  interactive setups, just chat output is enough.

## Cleanup

Skill exits with the dev server still running — the user is in
the middle of testing. They stop it manually:

```bash
kill $(cat /tmp/mulmoclaude-pr-<N>.pid)
git -C ~/ss/llm/mulmoclaude worktree remove ~/ss/llm/mulmoclaude-pr-<N>
rm /tmp/mulmoclaude-pr-<N>.{log,pid}
```

If the user invokes this skill on a second PR while the first is
still running, surface the existing PID and ask whether to swap
or run both (running both requires manually editing
`vite.config.ts`'s proxy port, which is out of scope for v1 —
recommend swap).

## Failure modes

- **PR isn't OPEN**: stop, report state, no setup.
- **`yarn install` fails**: surface the error, leave worktree in
  place so user can debug. Don't try to start the dev server.
- **`npm run dev` exits non-zero before "ready"**: dump the last
  20 lines of the log, ask user.
- **Worktree add fails (branch protected, fetch unreachable)**:
  surface and stop.

## Out of scope (v1)

- Auto-running e2e Playwright suites against the PR (users do
  manual UI tests; e2e is CI's job).
- Database / fixture seeding for the dev server.
- Two PRs at once (port collision; design above).
- Detecting "this PR doesn't need UI testing" — user invokes the
  skill only when they think it does. Trust them.
