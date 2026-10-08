# TypeScript tooling traps

Referenced from CLAUDE.md → TypeScript. Both entries below cost most of a session's
investigation time before the cause was found, and neither is visible to `yarn typecheck`
— that is what makes them worth writing down.

The shape they share: **the tool reports something that cannot be true, and the instinct is to
doubt the report.** In both cases the report was honest and the environment was lying.

---

## Type-aware lint rules reporting nonsense means a duplicate TypeScript, not bad code

**Symptom.** A type-aware rule fires on code that obviously satisfies it. The case that found
this was `@typescript-eslint/only-throw-error` on:

```ts
throw new Error("plain");
```

`Expected an error object to be thrown.` A plain `new Error` is the definition of what that rule
wants. Sibling rules misbehave at the same time: `no-unsafe-*` reporting "a type that cannot be
resolved", `sonarjs/null-dereference` firing on values that cannot be null.

**Cause.** Type identity in TypeScript is **per compiler instance**. `typescript-eslint` builds
the program with one copy; `ts-api-utils` (which the rules use for type predicates) resolves
another. A check compiled against instance B does not recognise the `Error` from instance A, so
every type-identity question answers wrongly.

The duplicate arrives through an ordinary version range. `eslint-plugin-sonarjs` requiring
`typescript ">=5 <6.1.0"` while the project pins a lower major is enough: the package manager
gives the plugin its own nested copy rather than deduplicating.

**Diagnosis — one command, before doubting the rule:**

```sh
find node_modules -maxdepth 4 -path "*/node_modules/typescript/package.json"
```

Any output is the answer. Print the versions of what it finds and compare against the root.

**Fix.** Unify TypeScript across the repo so the nested copies collapse — which is what the
"one TypeScript version everywhere" rule in CLAUDE.md exists to prevent. Confirm with the same
`find` afterwards, and re-run the probe that was failing; it should fall silent.

- MUST run the `find` above before adjusting a rule, adding a disable, or "fixing" the code a
  type-aware rule complains about, whenever the complaint does not match what the code plainly says.
- NEVER downgrade or disable a rule that is reporting nonsense. The report is a symptom of a
  broken program, and silencing it leaves every *other* type-aware rule in the same repo
  silently unreliable.

---

## `skipLibCheck: true` hides a dependency's broken types and turns its exports into `any`

**Symptom.** None. That is the problem. `yarn typecheck` passes, `yarn build` passes, the editor
shows no error, and calls into the dependency are checked against nothing.

**Cause.** A package ships `"type": "module"` with **extensionless relative imports** in its
`.d.ts` files. Under `moduleResolution: "node16"` / `"nodenext"` those do not resolve (`TS2834`).
`skipLibCheck: true` — which nearly every project sets, for good reasons — suppresses the error,
and the unresolved re-exports degrade to `any` instead of failing.

Everything downstream inherits it. A class imported this way accepts any value; its methods
return `any`; and `any` propagates through the calling code invisibly.

**Diagnosis — assert something that must be rejected:**

```ts
import { SomeExport } from "the-dependency";
export const shouldBeAnError: SomeExport = 42;
```

If that compiles, the type is dead. Confirm the cause with `tsc --skipLibCheck false` and read the
errors reported inside `node_modules`.

**Beware the vacuous probe.** A probe that *uses* the suspect type rather than *constraining* it
proves nothing — `type X = ReturnType<typeof v.method>` is `any` when the type is dead, and every
assertion against `any` passes. The probe must be a value the type is obliged to reject.

**Fix.** Declare the surface actually used, in a local `.d.ts`, with signatures copied from the
package's own declarations:

```ts
declare module "the-dependency" {
  export class SomeExport { /* only what this repo calls */ }
}
```

- MUST note in the file why it exists and that it should be deleted once upstream ships types that
  resolve — a hand-maintained declaration silently drifts, and unlike the `any` it replaced, a
  wrong signature here is *trusted*.
- SHOULD report it upstream. The package is broken for every NodeNext consumer, not just this repo.
- MUST re-run the rejecting probe after the fix and confirm it now errors. A declaration that is
  not picked up fails exactly as silently as the problem it was meant to solve.
