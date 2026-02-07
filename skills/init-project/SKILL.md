---
description: Set up a new TypeScript project with standard configuration
---

## New Project Setup

When creating a new package.json, always include these scripts:

- `format` - Prettier
- `lint` - ESLint
- `build` - Build command

Node.js version: **24** or later
ESLint must use **flat config** (eslint.config.js), not legacy .eslintrc
Use **Tailwind CSS v4** (not v3) with `@tailwindcss/vite` plugin

ESLint rules:
- Indent: 2 spaces
- Quotes: double (`"`)
- Semicolons: required
- Line endings: Unix (LF)
- Unused variables: prefix with `__` to ignore

### TypeScript ESM Configuration

For TypeScript projects, use ESM with `.js` extensions in imports:

**tsconfig.json** (key settings):
```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "declaration": true,
    "outDir": "./lib",
    "strict": true
  }
}
```

**package.json** (key settings):
```json
{
  "type": "module",
  "main": "lib/index.js",
  "types": "lib/index.d.ts",
  "exports": {
    ".": {
      "types": "./lib/index.d.ts",
      "default": "./lib/index.js"
    }
  }
}
```

**Import statements** - Always use `.js` extension (even for `.ts` files):
```typescript
// Correct
export * from "./actions/index.js";
import { foo } from "./utils/helper.js";

// Wrong - will fail at runtime
export * from "./actions/index";
import { foo } from "./utils/helper.ts";
```

This configuration ensures:
- TypeScript compiles to ESM
- Node.js resolves modules correctly at runtime
- Package consumers get proper type definitions

### Node/Browser Dual Export

When implementing library-like packages, always consider whether functions are **pure** (no Node.js-specific dependencies). Export browser-compatible code separately so it can run in both Node.js and browser environments.

**Design principles**:
- Pure functions (data transformation, validation, parsing) should be exported for browser use
- Code depending on Node.js APIs (`fs`, `path`, `child_process`, etc.) stays Node-only
- Separate entry points into `index.node.ts` and `index.browser.ts`

Extend the basic package.json above: change `main` to `lib/index.node.js` and add a `./browser` export:
```json
"exports": {
  ".":        { "types": "./lib/index.node.d.ts",    "default": "./lib/index.node.js" },
  "./browser": { "types": "./lib/index.browser.d.ts", "default": "./lib/index.browser.js" }
}
```

**File structure example**:
```
src/
  index.node.ts      # Node entry (re-exports everything)
  index.browser.ts   # Browser entry (re-exports pure functions only)
  utils/
    parser.ts         # Pure function - available from browser
    transform.ts      # Pure function - available from browser
  node/
    file_io.ts        # fs-dependent - Node only
    cli.ts            # Node only
```

Reference: `mulmocast-cli/package.json` exports pattern
