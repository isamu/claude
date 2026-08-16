# Reading files: never read a file whole unless you know it is bounded

Referenced from CLAUDE.md → Coding Style. Read before adding any read of a file your own code does
not write.

`fs.readFile(file, "utf8")` **throws past ~512 MB** (V8's maximum string length) no matter what the
file contains. A `catch` around it then reports "nothing here", so **the biggest, most-used data
reads as the emptiest** — and nothing errors. Found in mulmoterminal #998: a 585 MB Claude
transcript made a session show no title, no timeline and **a cost of $0**, across eight separate
read sites.

Before adding a read, MUST answer three questions:

1. **Who writes this file?** Yours, or something outside your control (an agent, a user, another
   process)? The danger is not "a big file" — it is **a file that grows without bound because
   something else is appending to it**. A `tasks.json` your app writes stays small at 100 entries;
   a transcript another tool appends to has no ceiling.
2. **Is there a cap?** A count limit, rotation, or a TTL. If the writer caps what it produces, the
   reader is safe.
3. **If not** → MUST NOT build one string. Pick by what the code actually needs:
   - **the end** (last turn, latest state) → read a bounded tail; measure the window against REAL
     data, since one record can be huge
   - **every record** (totals, counts, a scan) → stream line by line and fold as records arrive
   - **serving it to a client** → a size cap plus an explicit error (413), never a silent empty

Two traps that cost real bugs in #998:

- **A window must not paraphrase the rule it replaces.** Folding "the newest X" is not the same as
  running the original rule on a smaller input — fallbacks and cross-record semantics get lost.
  MUST feed the ORIGINAL function a smaller window, and MUST pin equivalence with a test that
  compares the streamed result against the whole-array result on the same records.
- **A window sized by guess is worse than no fix.** 256 KB looked generous and held nine records of
  that transcript — not one complete turn — so the exception disappeared while the screen stayed
  empty, which is indistinguishable from a new session. MUST measure against real data.
