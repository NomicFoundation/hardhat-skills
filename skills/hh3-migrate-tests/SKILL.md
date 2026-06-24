---
name: hh3-migrate-tests
description: Fix test failures after migrating a Hardhat V2 project to V3 — covers ESM import fixes, V2→V3 API replacements in test files, chai matcher signature changes, network helper migration, and common troubleshooting. Use when tests are failing after a Hardhat upgrade, fixing tests after hardhat migration, or "tests failing after upgrade to hardhat 3".
---

Phase 5 of the V2→V3 migration. Follow [migration-conventions.md](../migrate-hardhat2-to-hardhat3/references/migration-conventions.md) first.

**Prerequisites:** Phases 1–2 + source-file migration done (ESM, V3 config, source files on V3 APIs).

References for this phase:
- [test-migration.md](references/test-migration.md) — the fix-up workflow + error-pattern checklist
- [esm-migration.md](../hh3-migrate-source-files/references/esm-migration.md) — ESM patterns in test files
- [hardhat-api-migration.md](../hh3-migrate-source-files/references/hardhat-api-migration.md) — V2→V3 API mapping for test code
- [troubleshooting.md](../migrate-hardhat2-to-hardhat3/references/troubleshooting.md) — common errors

## Step 1 — Collect test files

Find every `.ts` test file (`test/`, `tests/`, and anywhere else tests live). Don't edit or run anything yet — just produce the list.

## Step 2 — Fix tests

Process each file with the workflow in [test-migration.md](references/test-migration.md): run `npx hardhat test {file}`, fix failures using the error-pattern checklist (max ~5 attempts/file, then escalate), move on. After processing all files, re-run every file to catch regressions a later fix may have introduced; repeat until all pass in one clean sweep. Config/package changes go to those domains (directly, or by handoff in sub-agent mode).

**Large suites (sub-agent fan-out):** this per-file run + full re-sweep loop is the dominant context cost on a big test suite — on a large project it can fill the orchestrator's window on its own, which is the main reason to fan out at scale. When the test-file count crosses the scale trigger in [migration-conventions.md](../migrate-hardhat2-to-hardhat3/references/migration-conventions.md) ("Optional: sub-agent fan-out"), split the files into batches of ~10, one sub-agent per batch, then a final agent re-runs every file. Each sub-agent absorbs its own failing-test output and returns only a summary.

**Gate:**

- [ ] `npx tsc --noEmit` still passes with zero errors (no regressions)
- [ ] Every test file passes individually (`npx hardhat test {file}`)
- [ ] `npx hardhat test solidity` still passes (regression check — test-helper/config fixes can break Solidity tests)
