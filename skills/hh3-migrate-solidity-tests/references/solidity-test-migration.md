# Solidity Test Migration (V2 → V3)

Skip this entire phase if the project has no Solidity tests — look for `.t.sol` files or Foundry-style test contracts. If none exist, report it skipped.

## Required dependency: the dry-run Foundry→HH3 skill

This phase reuses the Foundry→Hardhat 3 migration procedure documented in the [dry-run-migrate-foundry-to-hh3](../../dry-run-migrate-foundry-to-hh3/SKILL.md) skill. **Read it in full before modifying any file**, then follow it end-to-end for the Solidity test migration. If that file is missing, stop and ask the user to provide it.

## Completion gate (HARD STOP)

Run this before declaring success:

```bash
npx hardhat test solidity
```

If it fails, you are not done — fix the errors and re-run until it passes with zero errors. Do not suggest a commit or declare success until it exits cleanly.

## Reduce fuzz/invariant runs during migration

Solidity fuzz/invariant tests are slow and noisy while you iterate on compilation and logic fixes. Temporarily lower their iteration counts, then restore production values once the migration is stable. The settings and snippet live in the config guide — see [hardhat-config-migration.md](../../hh3-migrate-config/references/hardhat-config-migration.md) § "Reduced fuzz/invariant runs (MANDATORY)".

## Common V2 → V3 issues

### 1. Non-test contracts detected as test suites (false positives)

**Symptom:** Harness or helper contracts under `test/` fail with `constructor()` errors or produce unexpected test entries.

**Root cause:** HH3's Solidity test runner treats a **deployable** contract under the test directory as a test suite if its ABI has **any** function whose name starts with `test` or `invariant` (see `isTestSuiteArtifact` in the `solidity-test` builtin plugin, `node_modules/hardhat/dist/src/internal/builtin-plugins/solidity-test/helpers.js`). Contracts with no bytecode (abstract contracts, interfaces) are skipped. Common offenders: harness contracts with `testing_*` helpers, mocks with `testMode()` getters, any non-test contract with a function coincidentally starting with `test`.

**Detect:**

```bash
grep -rn "function test\|function invariant" test/ --include="*.sol" | grep -v "\.t\.sol"
```

**Fix:** Rename the offending functions to avoid the `test`/`invariant` prefix (`harness_`, `exposed_`, `mock_`, …):

```solidity
// Before (triggers false positive)
function testing_setBaseVersion(uint256 v) external { ... }
// After
function harness_setBaseVersion(uint256 v) external { ... }
```

Then update every caller of the renamed functions — Solidity, TS, and JS:

```bash
grep -rn "testing_setBaseVersion" test/ --include="*.sol" --include="*.ts" --include="*.js"
```

### 2. `allowInternalExpectRevert` — don't wrap internal calls

**Symptom:** A test uses `vm.expectRevert` before an internal function call or a direct `revert()`, and the revert isn't caught (test fails unexpectedly).

**Root cause:** In Foundry, `vm.expectRevert` by default only catches reverts on the **next external call**; it ignores reverts from internal functions or local logic.

**Fix — do NOT refactor tests into external wrappers or harness contracts.** A `config`-domain change — enable `allowInternalExpectRevert` under `test.solidity` in `hardhat.config.ts`:

```typescript
test: {
  solidity: {
    allowInternalExpectRevert: true,
  },
},
```

This preserves the original test intent and keeps the migration minimal.

### 3. Contract size limit (mocha/ethers deployment path — NOT the Solidity test runner)

**Symptom:** `Transaction reverted: trying to deploy a contract whose code is too large` when deploying harness/mock contracts (e.g. `*__Harness`, `*__Mock`) **from TypeScript/mocha tests, scripts, or Ignition** on the `edr-simulated` network.

**Does not apply to `npx hardhat test solidity`:** the HH3 Solidity test runner does **not** enforce the EIP-170 24KB limit, and `allowUnlimitedContractSize` has **no effect** on it (the option does not exist on the Solidity-test runner config, and the `test solidity` task reads only `hardfork` from the network config). So this symptom never surfaces from this phase's gate — it belongs to the mocha/ethers path (Phase 5) and to deploy scripts. It is noted here only because the offending harness/mock contracts usually live under `test/`.

**Root cause:** V3's default EDR network sets `allowUnlimitedContractSize: false`; harness contracts often exceed the EIP-170 24KB limit when deployed via the EDR provider backing the mocha/ethers connection.

**Fix:** A `config`-domain change — add `allowUnlimitedContractSize: true` to the default network in `hardhat.config.ts` (in sub-agent mode, hand this off to the config domain). See [hardhat-config-migration.md](../../hh3-migrate-config/references/hardhat-config-migration.md) § "Default network: `allowUnlimitedContractSize`" for the exact snippet.
