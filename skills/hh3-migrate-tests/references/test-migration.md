# Test Suite Fix-Up (Hardhat V3)

Fix `.ts` test failures caused by the V2→V3 migration. File migration (ESM syntax, V3 API calls) is already done — your job is to get the test suite green.

**Do not edit already-ESM `.js` test files — only fix `.ts` test files.** (Any CommonJS `.js` test files were converted to ESM or renamed to `.cjs` in Phase 4.)

Read these first:
- [esm-migration.md](../../hh3-migrate-source-files/references/esm-migration.md) — ESM patterns in test files
- [hardhat-api-migration.md](../../hh3-migrate-source-files/references/hardhat-api-migration.md) — V2→V3 API mapping for test code
- [troubleshooting.md](../../hardhat-v2-to-v3-migration/references/troubleshooting.md) — common errors and fixes

## Test command

Always run individual files with `npx hardhat test {filePath}`. Don't use `package.json` scripts — they may carry flags that mask or change behavior.

## Runner & config changes (V2 → V3)

- `hardhat compile` → `hardhat build` (prefer `build`; `compile` still works as an alias, so existing invocations don't break)
- `--parallel` flag removed — to keep parallel execution, set `test: { mocha: { parallel: true } }` in `hardhat.config.ts` instead (parallel testing is still supported)
- Tracing via verbosity levels: `-vvv` (level 3, traces for failing txs), `-vvvv` (level 4, traces for all txs ≈ old `--trace`), `-vvvvv` (level 5 ≈ old `--fulltrace`)
- `--gas-stats` is the built-in replacement for the `hardhat-gas-reporter` plugin; the plugin's `gasReporter` config and its `REPORT_GAS` env var no longer apply
- Foundry-style Solidity tests run natively: `hardhat test solidity`
- `hardhat.config.ts` uses `defineConfig()` + explicit `plugins` array (no side-effect imports)

## Common error patterns (diagnostic checklist)

**ESM syntax** — missing `.js` extension on relative imports; `require()` in ESM; `__dirname`/`__filename` not converted to `import.meta.dirname`/`.filename`; missing `type` keyword on type-only imports (`verbatimModuleSyntax`); directory imports not resolved to `/index.js`.

**V2 API not updated** — `hre.ethers` instead of `connection.ethers` (via `hre.network.create()`); `hre.network.provider.send()` instead of `connection.provider.request()`; `loadFixture`/`time`/`mine`/`setBalance` imported from `@nomicfoundation/hardhat-network-helpers` instead of `networkHelpers`; missing `await` on `hre.network.create()`.

**Chai matcher signatures** — `.to.be.reverted` → `.to.revert(ethers)`; `.to.not.be.reverted` → `.to.not.revert(ethers)`; `.changeEtherBalance(s)` / `.changeTokenBalance(s)` now take `ethers` as first arg; `.revertedWithoutReason()` → `.revertedWithoutReason(ethers)`. (`.emit` is **unchanged** — still `.emit(contract, eventName)`, no `ethers` arg.)

**Config / package** — missing plugin in `hardhat.config.ts`; missing dependency in `package.json`; mocha timeout too low; tests hang on startup → likely a mocha root-hooks file with HRE dependencies statically imported in the config (see [hardhat-config-migration.md](../../hh3-migrate-config/references/hardhat-config-migration.md) § "Warning: Mocha root hooks with HRE dependencies").

**Test-specific** — business logic dependent on V2 behavior; fixtures needing restructuring; failures that don't match the patterns above.

## Workflow

1. **Iterate** — for each file in order: run `npx hardhat test {filePath}`. If it passes, move on. If it fails, read it, fix all errors (use the checklist above), re-run (max ~5 attempts). If still stuck, escalate to the user. If a fix needs a change in the `config` or `package` domain, make it there (in sub-agent mode, hand it off).
2. **Verify all** — once you've been through every file in step 1, re-run all files in your list; a later fix may have broken an earlier one. Restart this sweep from the top until everything passes in one clean run.

**Dependency fixes:** if a test fails because a shared helper wasn't migrated properly, fix the helper first, verify it compiles (`npx tsc --noEmit --pretty 2>&1 | grep {helper}`), then return to the test.

## Validation gate (HARD STOP)

Do not declare success until:

- [ ] `npx tsc --noEmit` passes with zero errors (no regressions from your fixes)
- [ ] Every file in your list passes `npx hardhat test {filePath}` with zero errors
- [ ] `npx hardhat test solidity` still passes (regression check — test-helper/config fixes can break Solidity tests)

If stuck on a file (same error after ~5 attempts, or fixes introduce new errors), stop and tell the user: the exact error output, which files you tried, and what you attempted.
