---
description: Update yarn dependencies with safe, staged upgrade strategy and audit
---

## Yarn Dependency Update

### Phase 1: Safe Upgrades

1. **Update devDependencies (minor)**:
   ```bash
   yarn upgrade --dev --minor
   ```
2. **Update dependencies (patch)**:
   ```bash
   yarn upgrade --patch
   ```
3. Run `yarn lint`, `yarn build`, `yarn test` to verify
4. Commit and create PR — wait for CI to pass before proceeding

### Phase 2: Aggressive devDependencies Upgrade

1. **Update all devDependencies to latest**:
   ```bash
   yarn upgrade --dev --latest
   ```
2. Run `yarn lint`, `yarn build`, `yarn test`
3. If issues occur, revert and report which packages caused the failure
4. If all passes, commit to the same PR

### Phase 3: Security Audit

1. **Run audit**:
   ```bash
   yarn audit
   ```
2. For vulnerabilities, `yarn audit fix` usually does not work — instead:
   - Delete the affected entry from `yarn.lock`
   - Run `yarn install` to re-resolve
   - Run `yarn audit` again to verify the fix persists
3. Commit fixes to the same PR

### Phase 4: Deduplicate

Always run at the end:
```bash
npx yarn-deduplicate --strategy=fewer
yarn install
```
Commit if changes were made.

### Rules

- Branch name: `chore/yarn-update-YYYYMMDD` (e.g., `chore/yarn-update-20260209`) to keep names unique across runs
- Create a single PR for all phases
- Confirm CI passes after Phase 1 before proceeding to Phase 2
- If Phase 2 breaks something, revert and keep only Phase 1 changes
- Always end with deduplication
- Merge the PR once CI passes
