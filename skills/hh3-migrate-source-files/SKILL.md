---
name: hh3-migrate-source-files
description: Convert all TypeScript/JavaScript source files from CommonJS to ESM and update Hardhat V2 API calls to their V3 equivalents — covers require-to-import, module.exports-to-export, .js extensions, __dirname/__filename, network helpers, chai matcher changes, TypeChain import verification, and library linking paths. Use when converting a project to ESM, updating hardhat imports, fixing typechain for v3, or migrating hre.ethers usage.
---

Phase 4 of the V2→V3 migration. Follow [migration-conventions.md](../hardhat-v2-to-v3-migration/references/migration-conventions.md) first.

**Prerequisites:** Phases 1–2 done (ESM, V3 installed, `hardhat.config.ts` on `defineConfig`).

References for this phase:
- [esm-migration.md](references/esm-migration.md) — CJS→ESM conversion rules
- [hardhat-api-migration.md](references/hardhat-api-migration.md) — V2→V3 API replacements (connection, network helpers, chai matchers, import paths, library linking)
- [typechain-migration.md](references/typechain-migration.md) — TypeChain import fixes for ESM
- [task-migration.md](../hh3-migrate-config/references/task-migration.md) — for any task files

## Step 1 — Collect and classify files (domain: `esm`/`typechain`, read-only)

Find every `.ts` source file (exclude `node_modules`, the TypeChain output dir, and generated files). For each, note whether it imports from the TypeChain output directory (e.g. `typechain-types/`) — those files need TypeChain import fixes in Step 2. Also list any CJS tool-config `.js` files still using `module.exports` (`.eslintrc.js`, `.prettierrc.js`, `jest.config.js`, `commitlint.config.js`, `lint-staged.config.js`, etc.) — under `"type": "module"` these now parse as ESM and crash, so they need a `.cjs` rename in Step 2 (see [esm-migration.md](references/esm-migration.md) §8). **Also list any CommonJS `.js` _source_ files** (using `require()`/`module.exports` — e.g. `scripts/deploy.js`, `helpers/*.js`); under `"type": "module"` these crash at runtime and must be converted to ESM or renamed to `.cjs` in Step 2 (see [esm-migration.md](references/esm-migration.md) §1). **Also, if `tsconfig.json` has `baseUrl`/`paths`, flag every file using a bare-specifier (path-alias) internal import** (`from "lib"`, `from "lib/units"`, `from "typechain-types"`, … — grep the project's actual `paths` keys); these resolve under V2 but break under HH3 ESM and must be rewritten to `.js`-extension imports in Step 2 — either relative (`./lib/units.js`) or a `.js`-suffixed alias subpath (`lib/units.js`), both of which pass `tsc` **and** the `tsx` runtime (see [esm-migration.md](references/esm-migration.md) §2). In projects that use path aliases this is often the largest single error class. Keep this list to work through; **do not edit any file yet.**

- [ ] Complete `.ts` file list collected, with the TypeChain-importing files identified
- [ ] Bare-specifier / tsconfig path-alias imports identified (if `baseUrl`/`paths` present) for rewrite to `.js`-extension imports
- [ ] CJS tool-config `.js` files needing a `.cjs` rename identified
- [ ] CJS `.js` _source_ files (require()/module.exports) identified for ESM conversion or `.cjs` rename

## Step 2 — Convert files (domain: `esm` + `typechain`)

Process every `.ts` file from Step 1 **in order, one at a time**. For each:

1. **ESM conversion** — `require()`→`import`, `module.exports`→`export`, `.js` extensions, `__dirname`→`import.meta.dirname`, etc.
2. **V2→V3 API updates** — deprecated methods to V3 equivalents.
3. **TypeChain imports** (only files that import from the TypeChain output dir) — add `/index.js` suffix; split barrel imports into direct file imports when a type isn't re-exported.
4. **Verify:** `npx tsc --noEmit --pretty 2>&1 | grep {filename}` — zero errors = pass, then move to the next file. This is the verification for **all** files, including TypeChain-importing ones: a `has no exported member` error is reported against the importing file, so this grep catches it (no scratch file or `tsconfig` edit needed).

If a file fails to compile because an imported dependency hasn't been migrated yet, migrate that dependency first (verify it compiles alone), then return. Apply recursively.

**Handle the CJS `.js` source files** from Step 1: convert each to ESM syntax (preferred — then it follows the same per-file loop above) or rename it to `.cjs` to keep it as CommonJS (see [esm-migration.md](references/esm-migration.md) §1). Update any references to a renamed file in `package.json` and CI scripts.

**Then rename the CJS tool-config files** from Step 1: rename each `module.exports` `.js` config to `.cjs` and update every reference to the old filename in `package.json` and CI scripts. See [esm-migration.md](references/esm-migration.md) §8 for which to rename and which to leave (files already on `export default`, or tools using `.json`/`.yaml` config).

**Large projects (sub-agent fan-out):** independent leaf modules can be split across sub-agents, but **import-coupled files must stay on one agent** because of the recursive dependency-first rule above, and the `npx tsc --noEmit` / `npx hardhat build` gate is a global join point all agents must pass. Fan out only past the scale trigger in [migration-conventions.md](../hardhat-v2-to-v3-migration/references/migration-conventions.md) ("Optional: sub-agent fan-out"); below it, stay single-agent.

**Gate:**

- [ ] `npx hardhat build --force` passes with zero errors
- [ ] `npx tsc --noEmit` passes with zero errors
- [ ] Every collected `.ts` file converted and compiling
- [ ] CJS `.js` source files converted to ESM or renamed to `.cjs` (no `require()`/`module.exports` `.js` source left under `"type": "module"`)
- [ ] CJS tool-config `.js` files renamed to `.cjs`, with `package.json`/CI references updated (no `module.exports` `.js` config left under `"type": "module"`)
- [ ] No `@ts-ignore` / `@ts-expect-error` / `any` casts added to paper over missing types
- [ ] No files manually created/edited inside the TypeChain output folder (all auto-generated)
