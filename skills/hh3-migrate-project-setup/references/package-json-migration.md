# Package.json Migration (V2 to V3)

## Prerequisites

Hardhat V3 requires **Node.js v22.13.0 or later** (Hardhat 3.9.0 enforces this minimum at startup). Verify before starting:

```bash
node --version
```

If the version is below v22.13.0, **stop and alert the user**. This is a blocking requirement — the migration cannot proceed until Node.js is upgraded.

### Detect the project's package manager

Use the lockfile to pick the package manager, and use it for **every** install/add/remove command throughout the migration — mixing them creates conflicting lockfiles and corrupts `node_modules` (see [migration-conventions.md](../../hardhat-v2-to-v3-migration/references/migration-conventions.md)): `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm.

## 0. Pre-migration Audit

Before changing anything, create an inventory of the project's Hardhat ecosystem.

### 0a. Catalog all Hardhat-related dependencies

```bash
grep -E "hardhat|@nomicfoundation|@nomiclabs|@openzeppelin|solidity-coverage|solhint|solidity-docgen" package.json
```

Cross-reference each dependency against the Plugin Migration Map below (in section 2) to understand which packages will be removed, replaced, or kept. The table is not exhaustive — anything not listed is handled by the default rule in "Plugins not in the table."

### 0b. Catalog custom tasks and scripts

```bash
grep -rn "task\|subtask\|extendConfig\|extendEnvironment" hardhat.config.*
```

Review the `scripts/` directory for any Hardhat-dependent deploy or utility scripts. These will need updating in the config migration step.

### 0c. Catalog test infrastructure

Identify the test patterns in use — this informs which toolbox to install and what test migration will look like:

- Test runner (Mocha, or other)
- Assertion library (Chai with hardhat-chai-matchers, Waffle)
- Network helpers (`@nomicfoundation/hardhat-network-helpers`)
- Ethers.js version and usage of `hre.ethers`
- Any fixtures or shared test utilities relying on `hre.network.provider`

### 0d. Check for migration blockers

Review https://github.com/NomicFoundation/hardhat/issues/7207 for known blockers. If the project depends on a plugin or feature not yet available in V3, document it and plan a workaround or deferral.

### 0e. Note Hardhat-independent tools

These packages are **not** part of the Hardhat plugin system and require no migration — keep them as-is:

- `solhint`
- `prettier-plugin-solidity`

Do not accidentally remove them during dependency cleanup.

## 1. Clear caches and artifacts

If `node_modules` is already installed, run the Hardhat 2 `clean` task to avoid issues with stale artifacts or caches. Skip this step if the project has no `node_modules` directory.

```bash
npx hardhat clean
```

## 2. Remove all Hardhat V2 dependencies

Remove these categories of packages from `package.json`:

- `hardhat` (the V2 version)
- `solidity-coverage`
- `hardhat-gas-reporter`
- `hardhat-contract-sizer` (no built-in V3 replacement; the community `@solidstate/hardhat-contract-sizer` — the original author's HH3 port, peer `hardhat ^3.8.0` — is a drop-in if sizing is needed, otherwise just remove)
- `hardhat-tracer` (built-in replacement; V3 has a `-v` verbosity flag on the test task)
- Hardhat plugin packages — any name matching `hardhat-*` or `@<scope>/hardhat-*`. This includes third-party scopes (e.g. `@primitivefi/hardhat-dodoc`), not just `@nomicfoundation/hardhat-*` and `@nomiclabs/hardhat-*`.
- `@typechain/*`, `typechain` — Remove. The **mocha-ethers** toolbox bundles `@nomicfoundation/hardhat-typechain` to generate typed artifacts natively. The **viem** toolbox does *not* bundle it — viem projects get typed contracts via `@nomicfoundation/hardhat-viem` instead

> **Note:** Removing `hardhat-deploy`, `@openzeppelin/hardhat-upgrades`, and `hardhat-verify` here is intentional. `hardhat-verify` comes back bundled in the toolbox (step 5); `@openzeppelin/hardhat-upgrades` and `hardhat-deploy` are each handled in the per-plugin notes below.

> **"Bundled by toolbox" means _peer dependency_.** The toolbox declares **no runtime `dependencies`** (the field is absent from its `package.json`); every plugin it "bundles" is a **peerDependency**. **npm 7+ auto-installs peers and hoists them**, so they resolve from the project root. **yarn (Berry) and pnpm leave them unresolvable to the project:** yarn Berry doesn't install undeclared peers at all, and pnpm *does* auto-install them (default `auto-install-peers`) but its isolated layout keeps them in `node_modules/.pnpm`, **not reachable from the project root**. Either way the config can't import any plugin, so yarn/pnpm projects must install these peers explicitly — step 5 shows how to derive and install them.

### Plugin Migration Map

Use this table to decide what to do with each V2 dependency. Packages marked "Remove" should be deleted from `package.json`. Packages with a V3 replacement will be installed in later steps.

| V2 Plugin | V3 Status | Action |
| --- | --- | --- |
| `@nomicfoundation/hardhat-ethers` | Available | Remove — bundled by `hardhat-toolbox-mocha-ethers` |
| `@nomicfoundation/hardhat-chai-matchers` | Replaced | Remove — replaced by `hardhat-ethers-chai-matchers` (bundled by toolbox) |
| `@nomicfoundation/hardhat-network-helpers` | Available | Remove — bundled by toolbox |
| `@nomicfoundation/hardhat-verify` | Available | Remove — bundled by toolbox |
| `@nomicfoundation/hardhat-ignition` | Available | Remove — bundled by toolbox |
| `hardhat-gas-reporter` | Built-in | Remove — use `hardhat test --gas-stats` instead |
| `hardhat-contract-sizer` | Community port | Remove the V2 package; if sizing is needed, install `@solidstate/hardhat-contract-sizer` (the original author's HH3 port, peer `hardhat ^3.8.0`) |
| `hardhat-deploy` (wighawag) | v2.x available | Remove — if used, reinstall `hardhat-deploy@latest` (v2.x, rocketh rewrite); see note below |
| `@openzeppelin/hardhat-upgrades` | V3 available (v4+) | Reinstall `@openzeppelin/hardhat-upgrades@^4` — see note below |
| `solidity-coverage` | Built-in | Remove — use `hardhat test --coverage` instead |
| `@typechain/hardhat`, `typechain` | Replaced | Remove — mocha-ethers toolbox bundles `@nomicfoundation/hardhat-typechain`; viem toolbox uses `@nomicfoundation/hardhat-viem` instead |
| `hardhat-docgen` / `solidity-docgen` | No V3 port | Remove — no replacement yet, note the gap for the user |
| `@nomiclabs/hardhat-etherscan` | Deprecated | Remove — replaced by `@nomicfoundation/hardhat-verify` (bundled by toolbox) |
| `@nomiclabs/hardhat-waffle` | Deprecated | Remove — migrate to `hardhat-toolbox-mocha-ethers` |
| `solhint` / `prettier-plugin-solidity` | Independent | **Keep** — not Hardhat plugins, no migration needed |

> **Note:** For packages marked "Built-in" (`solidity-coverage`, `hardhat-gas-reporter`), also update any npm scripts or CI commands that invoke them to use the corresponding flag instead (see step 6).

### Plugins not in the table

The table above covers the common cases — it is **not** exhaustive. For any other package that integrates with Hardhat (a name matching `hardhat-*` or `@<scope>/hardhat-*`, or anything importing `hardhat` in its API):

1. **Remove the V2 version** — a V2-only plugin breaks the V3 build.
2. **Check for a V3-compatible release** — npm `latest`, the plugin's repository, or issue #7207.
3. **If a V3 version exists**, reinstall it and note any config/API changes for the config migration step.
4. **If none exists**, leave it removed, note the lost functionality, and **flag it to the user**.

Never silently keep a V2-only plugin, and never silently drop functionality without telling the user.

### Plugin-specific notes

**`hardhat-deploy`**: Only relevant if the project uses it. The v2.x release is a full rewrite on the rocketh framework, not a drop-in upgrade. After removing the V2 version, reinstall it in this skill with the project's package manager (`hardhat-deploy@latest`, which is v2.x). Then **flag to the user** that deploy scripts need a manual rewrite (TypeScript functions, ESM, rocketh-extension plugin system) — this skill does not migrate them. See https://github.com/wighawag/hardhat-deploy for the full v1-to-v2 migration guide.

**`@openzeppelin/hardhat-upgrades`**: A Hardhat 3-compatible release exists — install `@openzeppelin/hardhat-upgrades@^4` (ESM-only; requires `hardhat ^3.6.0`, `@nomicfoundation/hardhat-ethers ^4`, `ethers ^6`) and register it in the config `plugins: [...]` array. The v3→v4 plugin moved off `extendEnvironment` to the V3 plugin system, so its usage in scripts/tests may need small adjustments. (If the project is pinned to an incompatible toolchain and can't move to v4 yet, OpenZeppelin Defender or manual proxy deployment remain fallbacks — flag this to the user.)

Then reinstall and verify no V2 dependencies remain.

```bash
# npm (has package-lock.json)
npm install && npm why hardhat

# pnpm (has pnpm-lock.yaml)
pnpm install && pnpm why hardhat

# yarn (has yarn.lock)
yarn install && yarn why hardhat
```

> **Stale lockfile:** A committed lockfile can still pin a removed V2 package (e.g. `@nomicfoundation/hardhat-toolbox`), so the install (or the `npm add`/`pnpm add`/`yarn add` in the next steps) can fail with a peer-dependency error (`ERESOLVE`) **even on a fresh clone with no `node_modules`**. If that happens, regenerate the lockfile (`rm package-lock.json` / `pnpm-lock.yaml` / `yarn.lock`) and re-run, or as a fallback use `--legacy-peer-deps` (npm).

If `why hardhat` still shows V2-related transitive dependencies (packages that depend on `hardhat@^2`), identify the parent package pulling them in and remove it. Repeat until no V2 transitive dependencies remain. After installing V3 in the next step, `why hardhat` will correctly show `hardhat@^3` — that's expected.

### Built-in replacements for removed packages

Several V2 plugins are now built into Hardhat V3. Coverage and gas reporting are global flags, so they work with any command that runs tests:

| Removed package          | V3 built-in replacement                                          |
| ------------------------ | ---------------------------------------------------------------- |
| `solidity-coverage`      | `hardhat test --coverage` (global flag)                          |
| `hardhat-gas-reporter`   | `hardhat test --gas-stats` (global flag)                         |
| `hardhat-contract-sizer` | No built-in replacement — remove; community `@solidstate/hardhat-contract-sizer` (peer `hardhat ^3.8.0`) is an option if sizing is needed                     |
| `hardhat-tracer`         | `-v` / `-vvv` verbosity flag (emits execution traces)            |

If the project uses `solidity-coverage` or `hardhat-gas-reporter`, remove them and update the relevant npm scripts or CI commands to use the corresponding flag instead (see step 6).

## 3. Make the project ESM

Add `"type": "module"` to `package.json`:

```bash
npm pkg set type=module
```

## 4. Install Hardhat V3

Install Hardhat 3 before adding the toolbox. Use the **same package manager** the project already uses. **Always pin `@latest`** — otherwise the existing `^2.x` range in `package.json` (and/or a stale lockfile) makes `add hardhat` resolve *within V2* and install the latest 2.x silently (exit 0, no error), so never install Hardhat without a version that forces the new major:

```bash
# npm
npm add --save-dev hardhat@latest
# pnpm
pnpm add -D hardhat@latest
# yarn
yarn add -D hardhat@latest
```

> **Version pinning:** After installing, verify that `package.json` contains actual version numbers (e.g. `"hardhat": "^3.0.2"`), never the string `"latest"`. This applies to all Hardhat packages and plugins installed during this migration. If any entry shows `"latest"`, replace it with the actual installed version by checking `node_modules/<package>/package.json`.

> **Note:** Do not run `npx hardhat --help` to verify yet — the old V2 config file is still in place with `require()`/`import` calls to packages that were just removed, so Hardhat will fail to load. Full verification happens after the config migration step.

## 5. Install the recommended toolbox

Add the toolbox:

```bash
# npm
npm add --save-dev @nomicfoundation/hardhat-toolbox-mocha-ethers
# pnpm
pnpm add -D @nomicfoundation/hardhat-toolbox-mocha-ethers
# yarn
yarn add -D @nomicfoundation/hardhat-toolbox-mocha-ethers
```

For Viem projects use `@nomicfoundation/hardhat-toolbox-viem` instead.

This step only installs the package. The toolbox import and plugin registration happen during the hardhat.config migration step — do not try to add imports to the config file here.

### Install the toolbox's plugin peers (REQUIRED on yarn/pnpm)

The toolbox bundles its plugins as **peerDependencies**. **npm 7+** auto-installs and hoists them so they resolve from the project root; **yarn (Berry) and pnpm do not make them resolvable to the project** — yarn Berry doesn't install undeclared peers at all, and pnpm auto-installs them but leaves them in `node_modules/.pnpm`, unreachable from the project root. So on yarn/pnpm the next phases can't import any plugin. Derive the list from the toolbox you just installed, then install with the project's package manager (point `node -p` at `@nomicfoundation/hardhat-toolbox-viem` for viem projects):

```bash
PEERS=$(node -p "Object.keys(require('@nomicfoundation/hardhat-toolbox-mocha-ethers/package.json').peerDependencies ?? {}).join(' ')")
yarn add -D $PEERS        # or: pnpm add -D $PEERS  ·  npm add --save-dev $PEERS
# verify each landed (catches the silent yarn/pnpm gap):
for p in $PEERS; do node -e "require.resolve('$p')" 2>/dev/null && echo "ok $p" || echo "MISSING $p"; done
```

> **yarn Berry (PnP) caveat:** with yarn's default Plug'n'Play linker there is **no `node_modules`**, so plain `node` can't resolve packages — both the `PEERS=…` derivation and the `require.resolve` verify fail with `MODULE_NOT_FOUND`. Run those `node` commands as **`yarn node …`** (e.g. `yarn node -p "…"`), or set `nodeLinker: node-modules` in `.yarnrc.yml` first. Either way, `yarn why <pkg>` / `pnpm why <pkg>` is a linker-agnostic way to confirm a plugin resolved.

Re-pinning `hardhat`/`ethers`/`mocha` to the toolbox-satisfying versions is safe. **`chai` is the exception:** the toolbox requires `chai >=5.1.2 <7` (latest is **v6**, which — like v5 — is **ESM-only** with behavioral changes vs v4), and a bare `add chai` does **not** bump an existing `chai@^4` pin on npm or pnpm — a retained `chai@4` then **hard-fails the install with an `ERESOLVE` peer conflict**. So upgrade chai explicitly — `npm add --save-dev chai@latest` (or `pnpm add -D chai@latest` / `yarn add -D chai@latest`) — and treat the chai v4→v5/6 jump as a Phase-5 test-migration risk. On npm the rest of this peer install is redundant (peers already resolve) but safe.

### Install the type packages the toolbox does not bundle (TypeScript + Mocha projects)

The mocha-ethers toolbox declares `mocha`/`chai`/`ethers` as **peer** dependencies and ships **no** `@types/mocha` (the V2 `@nomicfoundation/hardhat-toolbox` used to pull it in transitively). Without it, `npx tsc --noEmit` fails on every test file with `Cannot find name 'describe'` — which blocks the mandatory Phase 4/5 gates. Install the type packages the official HH3 mocha-ethers template ships as direct dev-dependencies, using the project's package manager:

```bash
# npm
npm add --save-dev @types/mocha @types/chai @types/chai-as-promised @types/node
# pnpm
pnpm add -D @types/mocha @types/chai @types/chai-as-promised @types/node
# yarn
yarn add -D @types/mocha @types/chai @types/chai-as-promised @types/node
```

`@types/mocha` is the one genuinely missing after the V3 install; `@types/chai`/`@types/node` often arrive transitively, but listing them matches the template and is safe across package managers. **Viem / node-test-runner projects do not need `@types/mocha`.** Installing `@types/mocha` is necessary but **not sufficient** on its own — you must also list `"mocha"` in the tsconfig `types` array (see [tsconfig-migration.md](tsconfig-migration.md) §1).

## 6. Update `scripts` entries

Review all entries in the `scripts` field for any `hardhat` commands and apply these mechanical replacements:

| V2 script                             | V3 replacement             |
| ------------------------------------- | -------------------------- |
| `hardhat compile`                     | `hardhat build`            |
| `hardhat test` with `REPORT_GAS=true` | `hardhat test --gas-stats` |
| `hardhat test` with coverage plugin   | `hardhat test --coverage`  |
| `hardhat test ... --parallel`         | Remove the `--parallel` flag — set `test.mocha.parallel: true` in the config to keep parallel runs |

Leave other task invocations (`hardhat run`, `hardhat verify`, `hardhat deploy`, custom tasks) alone for now — they may need updates during the config migration step.

## 7. Verify (HARD STOP)

This verification is **blocking** — do not return success or proceed until both checks pass.

### 7a. Install dependencies

Run `install` with the project's **existing** package manager (npm, yarn, or pnpm — whichever the project already uses). Never switch package managers during migration.

The install must exit cleanly with no errors. If it fails (missing packages, version conflicts, peer dependency issues), fix `package.json` and re-run until it succeeds.

### 7b. Confirm packages exist in node_modules

After a successful install, verify that the key V3 packages are actually present in `node_modules/`. Spot-check at minimum:

```bash
ls node_modules/hardhat/package.json
ls node_modules/@nomicfoundation/hardhat-toolbox-mocha-ethers/package.json 2>/dev/null || ls node_modules/@nomicfoundation/hardhat-toolbox-viem/package.json 2>/dev/null
```

If any expected V3 dependency is missing from `node_modules/`, the install did not work correctly — investigate and fix before continuing.
