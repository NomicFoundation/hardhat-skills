---
name: hh3-migrate-solidity-tests
description: Migrate Solidity test files (.t.sol) to Hardhat V3 test conventions — covers Foundry-style test migration, false positive test detection (non-test contracts with test/invariant prefixes), and fuzz/invariant run configuration. Use when migrating Solidity tests to Hardhat 3, fixing "hardhat test solidity" failures, or dealing with false positive test suite detection.
---

Phase 3 of the V2→V3 migration. Follow [migration-conventions.md](../migrate-hardhat2-to-hardhat3/references/migration-conventions.md) first.

**Prerequisites:** Phases 1–2 done (ESM, V3 installed, `hardhat.config.ts` on `defineConfig`).

## Step — Migrate Solidity tests (domain: `solidity-test`)

If the project has no `.t.sol` / Foundry-style test contracts, skip this phase and report it skipped.

Otherwise follow [solidity-test-migration.md](references/solidity-test-migration.md). It covers the full Foundry→HH3 test migration (via the dry-run skill it depends on), false-positive test-suite detection, `allowInternalExpectRevert`, and fuzz/invariant run reduction during migration. If a fix needs config changes (e.g. adding directories to `paths.sources.solidity`), make them in the `config` domain — directly, or by handoff in sub-agent mode.

**Gate:**

- [ ] `npx hardhat test solidity` passes with zero errors (or phase skipped)
