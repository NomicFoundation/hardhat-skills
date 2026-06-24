# Phase 1 — Project Setup

Phase 1 of the V2→V3 migration. Follow [migration-conventions.md](../migration-conventions.md) first (rules, package-manager detection, gates, optional parallel execution).

**Prerequisites:** Node.js v22.13.0+ (blocking — stop and tell the user if lower); a Hardhat V2 project with `hardhat` in `package.json`.

Run these three steps in order. Each ends with a gate — run its checks in order, fix and re-run from the first on any failure.

## Step 1 — Environment file (domain: `env`)

Create `.env` from an example template if one exists, so the variables the V3 config reads via `configVariable()` (RPC URLs, keys) are available when needed. Hardhat does **not** auto-load `.env` and you must **not** add `import "dotenv/config"` to the config — load `.env` at the shell (`set -a; source .env; set +a`) when running commands that resolve values. Reference: [env-file-migration.md](env-file-migration.md).

- [ ] `.env` created from `.env.example`/`.sample`/`.template`, or skipped (no template, or `.env` already present)

## Step 2 — tsconfig.json (domain: `tsconfig`)

Update `tsconfig.json` compiler options for ESM + V3. Reference: [tsconfig-migration.md](tsconfig-migration.md).

- [ ] `tsconfig.json` updated, or skipped (no file)

## Step 3 — package.json (domain: `package`)

Convert `package.json` to ESM, swap V2 dependencies for V3 equivalents, update scripts. Reference: [package-json-migration.md](package-json-migration.md).

- [ ] `package.json` converted, dependencies installed
- [ ] `npx hardhat --version` runs and prints a `3.x` version (e.g. `3.9.0`)
