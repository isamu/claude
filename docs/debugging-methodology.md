# Debugging methodology

Referenced from CLAUDE.md → Debugging Approach. Read before a non-trivial bug hunt (a symptom with several possible causes, a bug that repeats across call sites, a retry mechanism, a per-branch repro, or judging another repo's state).

## Is the report real?

Settle this BEFORE designing anything, in this order, and record the answer:

1. Does it **reproduce**? A repro you can run is the only cheap evidence — everything below is what you fall back to when there isn't one.
2. If it does not reproduce, is the line **reachable**? Grep the **producer** of the input, not only the handler, and name the caller that supplies it. A correct diagnosis of an unreachable path still costs a full PR and a reviewer's day (cases → [`issue-filing.md`](issue-filing.md)).
3. Is the behaviour actually **wrong**, or only different from what the report expected? A deliberate asymmetry, a documented limitation, a stale expectation, and a test asserting yesterday's contract all look exactly like bugs from the outside.

- Findings phrased as **shape** — "possible race", "may be null", "unbounded loop" — are claims about how the code looks, not about what production does. Bot reviews and your own skim both produce these in volume. MUST convert each into a concrete chain (entry point → caller → input value) or classify it as not established.
- A failing test proves a disagreement between test and code, not that the **code** is the wrong half. MUST check the test's own assumption before editing source.
- When the issue is real but the reported CAUSE is wrong, MUST say both. Fixing the reported cause closes the report and leaves the bug live.
- "Not reproducible" / "unreachable" is a **result**, and MUST be reported with what was checked and what could not be — never dropped silently, or the next person re-runs the same investigation from zero.

## Shaping the fix

- Before writing it, MUST answer: which layer **owns** this rule, what other inputs reach the same branch, and what would have to be true for this to be the only affected site. The call-site sweep below is the mechanical half of that question; this is the design half.
- Prefer moving the rule into **one pure function** over patching N call sites: the fix then has a name, a signature, and a place its tests can reach without booting the app. Purity boundaries and dependency injection → [`testing.md`](testing.md).
- MUST pin the originally reported input **verbatim** as one of the cases, so the report stays answerable months later.
- The abnormal side gets the attention, but the side that breaks silently is the **normal** one — a fix almost always adds a rejection or a normalisation. MUST test that previously-valid inputs still pass, not only that the bad input is now caught.
- MUST cover the abnormal side with inputs the caller can actually produce (a truncated file, an empty array from an API that "always" returns one, `undefined` from an optional field), not only types that could never arrive.

## Bug-hunt rules

- When one error symptom has MULTIPLE root causes (a "bug family"), MUST map every case into a matrix (symptom × layer × platform / layout / config) and fix + regression-test EACH case — never patch only the one that happened to reproduce, leaving the siblings live. Keep the matrix as a committed doc so the cases can't silently regress.
- A bug found at ONE call site is a report about a RULE, not about that line. NEVER fix only the site that surfaced. MUST first sweep the whole codebase for every place the same rule is decided — grep the operation, not the symptom (path joins, separator handling, encoding/escaping, timezone/date math, ID normalisation, comparison + sort keys, unit conversion, retry/timeout policy, permission checks) — then classify each hit as: (a) same bug → fix, (b) same rule already hand-rolled correctly → candidate to absorb, (c) deliberately different → leave, with the reason recorded. MUST then ask explicitly: **can this rule become ONE shared function?** If yes, extract it and route the call sites through it instead of repeating the fix. Two or more hand-rolled copies of the rule is proof the helper was already needed.
  - MUST report the sweep with counts before choosing scope (N sites found / M actually broken / K must-not-touch), so widening vs. staging the change is the user's call, not an assumption.
  - The classification matters as much as the fix: sites that look identical but are host-internal (containment checks, comparisons against a host-shaped value) MUST be left alone — "normalise everything the grep matched" trades one bug for another.
- When adding a retry / auto-recovery / replay mechanism, MUST adversarially review it (a dedicated review pass or sub-agent) for the classic failure modes BEFORE shipping: double-execution of side effects on replay, abort/cancel handling during any wait, and over-broad error matching that triggers false-positive retries.
- When a bug reproduces DETERMINISTICALLY on one branch/commit but not another (e.g. works on `main`, breaks on a feature branch), the cause is in the code path that branch changed — NOT the environment. MUST bisect to the differing code path and REPRODUCE the actual failure in isolation (a minimal script hitting that path) BEFORE proposing environmental explanations (build/Vite cache, `node_modules`, package versions). Do not offer cache/reinstall theories for a deterministic per-branch repro.
- Before judging ANOTHER repository's state (is this implemented? does this API exist?), MUST `git fetch` and read against `origin/<default-branch>` — never the local working copy, which may sit on a stale branch dozens of commits behind. "grep found nothing" means "not in the commit I am looking at", NOT "not implemented".
- MUST NOT take an error string, status label, or log line at face value when it is the only evidence. Get the primary value first (the actual variable, API return, or stored record). A message that lumps distinct states together — e.g. reporting a dismissed permission prompt as "denied" — sends the diagnosis hunting for something that was never there.
