---
name: hh3-migrate-config
description: Migrate hardhat.config.ts from Hardhat V2 imperative style to V3 declarative format using defineConfig — covers plugin registration, network configuration, custom task migration (builder pattern), solidity profiles, etherscan/verify config, and mocha/test settings. Use when migrating a Hardhat config to V3, converting hardhat.config to declarative format, or dealing with "hardhat 3 config" issues.
---

Phase 2 of the V2→V3 migration. Follow [migration-conventions.md](../migrate-hardhat2-to-hardhat3/references/migration-conventions.md) first.

**Prerequisites:** `package.json` has `"type": "module"` + Hardhat V3 installed; `tsconfig.json` set to a node-family `"module"` (`"nodenext"`, or `"node20"` if TypeScript `>= 5.9`) (Phase 1 done if running the full migration).

## Step — Migrate hardhat.config (domain: `config`)

Rewrite `hardhat.config.ts` to V3 declarative format with `defineConfig`: solidity settings, plugins, networks, custom tasks, verify, and test settings.

References:
- [hardhat-config-migration.md](references/hardhat-config-migration.md) — the step-by-step guide (V2↔V3 mappings, networks, plugins, etherscan, mocha, fuzz/invariant)
- [task-migration.md](references/task-migration.md) — custom task builder pattern; deciding which tasks still need migrating
- [config-example.md](references/config-example.md) — a complete V3 config to use as a template
- [troubleshooting.md](../migrate-hardhat2-to-hardhat3/references/troubleshooting.md) — common load/validation errors

**Gate:**

- [ ] New `hardhat.config.ts` created with `defineConfig`
- [ ] `npx tsc --noEmit` run **unfiltered** against a tsconfig that includes `hardhat.config.ts` (a usage/help banner means tsc never ran → FAILURE; a `tsconfig.json`/compiler-option diagnostic such as `TS6046`/`TS5070` — e.g. `"module": "node20"` on TypeScript `< 5.9` — means the tsconfig is broken and module-resolution checks silently don't run → FAILURE, fix the tsconfig first) and reports no error referencing `hardhat.config.ts` — see [hardhat-config-migration.md](references/hardhat-config-migration.md) §8a (do **not** rely on a filtered `| grep`, which false-passes when tsc never ran)
- [ ] `npx hardhat --help` runs without errors
