# Phase 3 — Solidity Tests

Phase 3 of the V2→V3 migration. Follow [migration-conventions.md](../migration-conventions.md) first.

**Prerequisites:** Phases 1–2 done (ESM, V3 installed, `hardhat.config.ts` on `defineConfig`).

## Step — Migrate Solidity tests (domain: `solidity-test`)

If the project has no `.t.sol` / Foundry-style test contracts, skip this phase and report it skipped.

Otherwise follow [solidity-test-migration.md](solidity-test-migration.md). It covers the full Foundry→HH3 test migration (via the dry-run skill it depends on), false-positive test-suite detection, `allowInternalExpectRevert`, and fuzz/invariant run reduction during migration. If a fix needs config changes (e.g. adding directories to `paths.sources.solidity`), make them in the `config` domain — directly, or by handoff in sub-agent mode.

**Gate:**

- [ ] `npx hardhat test solidity` passes with zero errors (or phase skipped)
