# Phase 6 — Script Validation

Phase 6 of the V2→V3 migration — interactive. Follow [migration-conventions.md](../migration-conventions.md) first.

**Prerequisites (verify; stop and raise with the user if any fails):** `package.json` on ESM + Hardhat V3; `tsconfig.json` on a node-family `"module"` (`"nodenext"`, or `"node20"` if TypeScript `>= 5.9`); `hardhat.config.ts` on `defineConfig`; `npx tsc --noEmit` passes clean (run unfiltered — a `tsconfig.json`/compiler-option diagnostic such as `TS6046`/`TS5070` is a FAILURE, not "clean").

## Step 1 — Ask which scripts to validate

Ask the user which `package.json` scripts to validate and in what order (e.g. `build`, `test:integration`, `deploy:local`, `lint`). Don't guess — only the user knows which matter. Wait for the answer.

## Step 2 — Validate one at a time

For each script, in order (never in parallel):

1. Run it with the project's package manager.
2. **Passes** → report success, ask for the next.
3. **Fails** → diagnose the error, identify the owning domain(s) via the **Domain table in [migration-conventions.md](../migration-conventions.md)** (it maps each domain to the files it owns), then pull in the matching reference docs below:

   | Domain | References |
   | --- | --- |
   | `config` | [hardhat-config-migration.md](../phase-2-config/hardhat-config-migration.md), [task-migration.md](../phase-2-config/task-migration.md) |
   | `package` | [package-json-migration.md](../phase-1-project-setup/package-json-migration.md) |
   | `tsconfig` | [tsconfig-migration.md](../phase-1-project-setup/tsconfig-migration.md) |
   | `env` | [env-file-migration.md](../phase-1-project-setup/env-file-migration.md) |
   | `esm` | [esm-migration.md](../phase-4-source-files/esm-migration.md), [hardhat-api-migration.md](../phase-4-source-files/hardhat-api-migration.md) |
   | `typechain` | [typechain-migration.md](../phase-4-source-files/typechain-migration.md) |
   | `test` | [test-migration.md](../phase-5-test-fixup/test-migration.md) |
   | `solidity-test` | [solidity-test-migration.md](../phase-3-solidity-tests/solidity-test-migration.md) |
   | any | [troubleshooting.md](../troubleshooting.md) |

   Then re-run the script until it passes. If the failure is ambiguous, diagnose first before fixing. If a fix needs a change in another domain (e.g. `config` or `package`), make it there — or hand it off in sub-agent mode (see conventions).

## Step 3 — Final gate

- [ ] Every script the user listed runs successfully
- [ ] `npx tsc --noEmit` still passes with zero errors
