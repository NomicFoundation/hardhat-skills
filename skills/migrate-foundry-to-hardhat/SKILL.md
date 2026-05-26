---
name: dry-run-migrate-foundry-to-hh3
description: Dry-run migration of a Solidity project from Foundry to Hardhat 3.
disable-model-invocation: true
argument-hint: "[update | optional notes about the project]"
---

# Dry-run migrate Foundry project to Hardhat 3

**If the argument is `update`, skip to the [Update an existing migration](#update-an-existing-migration) section below.**

The goal is to get Hardhat compilation and Solidity tests passing, or identify any blockers and missing features. This involves config mapping, dependency conversion, import fixes, compilation, and test verification.

**Important:** Identifying blockers and missing Hardhat features relative to Forge is a primary goal of this dry run. Do not overlook or guess at feature mappings. If a Forge feature has no clear Hardhat equivalent, do not silently skip it — leave a TODO comment in the config or code, link to a tracking issue if one exists, and flag it in the final migration report.

**Dry-run preservation:** Do NOT delete `foundry.toml`, `foundry.lock`, or existing Forge-based scripts in `package.json`. This is a dry-run — both toolchains must coexist. Add Hardhat scripts alongside existing ones using a `-hardhat` suffix.

Follow these steps in order. Do not skip ahead — each step depends on the previous one succeeding.

## Step 1: Analyze the Foundry project

Before making changes, read and understand the existing project **thoroughly**:

1. Read `foundry.toml` — note **ALL** configured profiles and settings. Do not overlook any section.
2. Read `remappings.txt` if it exists
3. Check for git submodules in `lib/` — run `ls lib/` and read `.gitmodules` if present. **Ensure submodules are initialized** — run `git submodule update --init --recursive`. Compilation will fail if submodule directories are empty.
4. Identify the contract source directory (usually `src/` or `contracts/`)
5. Identify the test directory (usually `test/` or `tests/`)
6. Identify any scripts in `script/`
7. Scan `.t.sol` files for **inline test config** — `forge-config:` directives in both line comments (`/// forge-config:`, `// forge-config:`) and block comments (`/** forge-config: ... */`). Since Hardhat 3.3.0, inline config is supported at the **function level** and `forge-config:` is backwards-compatible ([docs](https://hardhat.org/docs/guides/testing/inline-configuration)). However, Hardhat only supports **function-level** inline config — **contract-level** inline config (directives placed on the contract definition rather than individual functions) is NOT supported. For contract-level configs, the workaround is to set the value globally in `hardhat.config.ts`. Additionally, some settings are **not yet supported inline** even at function level (notably `isolate`, `evm_version` — see [edr#1349](https://github.com/NomicFoundation/edr/issues/1349)). Flag any files using contract-level directives or unsupported inline settings.
8. **Scan `package.json` scripts** — read every script entry and identify all Forge-dependent commands (not just `forge test` / `forge build`). Custom scripts using `forge bind`, `forge script`, `forge snapshot`, `forge verify-contract`, etc. represent additional Forge features the project actively uses. List them in the analysis file — they must appear in the feature parity table.
9. **Scan for zkSync-specific profiles and contracts:** Check `foundry.toml` for zkSync profiles (e.g., `[profile.zksync]`) and scan for zkSync-specific contract directories (`contracts/zkSync/`, `contracts/zk*/`, `src/zkSync/`, etc.). Assess what percentage of the codebase these contracts represent (file count relative to total `.sol` files). zkSync compilation settings and contracts may not be testable under Hardhat — flag them for the migration report.
10. **Determine the package manager:** Check which lockfile exists (`yarn.lock`, `package-lock.json`, `pnpm-lock.yaml`) and note the corresponding tool (`yarn`, `npm`, `pnpm`). If **no lockfile exists**, use `pnpm` as the default. If **multiple lockfiles exist**, pick the one that is more recently modified OR has more dependencies defined — whichever gives the clearest signal. Check modification timestamps with `ls -lt yarn.lock package-lock.json pnpm-lock.yaml 2>/dev/null` and line counts with `wc -l`. Document the choice and the reason in the analysis file. Do **not** introduce a second lockfile. **Lock in this choice** — use it for every install command throughout the session (including `patch-package` steps). Never mix package managers.

**IMPORTANT:** Migrating all configurations is KEY to the migration succeeding. Every section of `foundry.toml` must be accounted for — either mapped to a Hardhat equivalent, or explicitly documented as unsupported with a comment and issue link.

After analyzing, write a structured summary to a file named `.claude/<project-name>-foundry-migration-analysis.md` (where `<project-name>` is the directory name of the project being migrated) in the repository root. Create the `.claude/` directory if it doesn't exist. The file includes:

- Every `foundry.toml` section and setting found
- All profiles (`[profile.default]`, `[profile.ci]`, etc.) and their settings
- Remappings, submodules, source/test/script directories
- Any notable patterns (absolute imports, unusual configs)
- Inline test config usage (`forge-config:` comments) — list affected files and which inline settings are used, noting whether each directive is at the **function level** (supported in Hardhat) or **contract level** (NOT supported in Hardhat — must be set globally). Also flag any that use settings not yet supported inline even at function level (notably `isolate`, `evm_version` — see [edr#1349](https://github.com/NomicFoundation/edr/issues/1349))
- All Forge-dependent `package.json` scripts identified in step 8 above
- zkSync-specific profiles and contracts identified in step 9 above (if any), with percentage of codebase affected
- Selected package manager (from step 10 above)

This file serves as a reference throughout the migration so you don't have to re-read `foundry.toml` repeatedly. Present the summary to the user before proceeding.

## Step 2: Add Hardhat to the project

**Package manager:** Use the package manager determined in Step 1.

**Fetch the latest Hardhat version:** Run `view hardhat version` using the package manager determined in Step 1 (e.g. `pnpm view hardhat version`), and note the version string returned. If the command fails, fall back to `3.3.0`.

Edit `package.json`:

- Add `"hardhat": "^<latest-version>"` to `devDependencies`, using the version fetched above.
- Add `"@nomicfoundation/hardhat-verify": "^3.0.11"` to `devDependencies` (if `[etherscan]` section exists)
- Ensure the top-level field `"type": "module"` is set. **Note:** This switches the project to ESM, which is required by Hardhat 3. If the project has existing CommonJS files (`.js` with `require()`), this may break them — flag it to the user if so.
- Keep existing `forge`-based scripts intact. Add Hardhat equivalents alongside them using a `-hardhat` suffix. For Forge scripts that have no Hardhat equivalent, do not add a `-hardhat` variant.
- If the project has **no Forge-related scripts** (no `scripts` section, or only non-Forge scripts like `lint`, `prettier`), do **not** add any Hardhat scripts to `package.json`.

Common script mappings:

| Existing Foundry script | Hardhat script to add alongside |
| --- | --- |
| `"test": "forge test"` | `"test-hardhat": "npx hardhat test solidity"` |
| `"build": "forge build"` | `"build-hardhat": "npx hardhat compile"` |
| `"coverage": "forge coverage ..."` | `"coverage-hardhat": "npx hardhat test solidity --coverage"` |
| `"snapshot": "forge snapshot ..."` | `"snapshot-hardhat": "npx hardhat test solidity --snapshot"` |
| `"snapshot:check": "forge snapshot --check ..."` | `"snapshot-check-hardhat": "npx hardhat test solidity --snapshot-check"` |

**Note on coverage:** Hardhat 3 has built-in coverage support (no plugin needed). The `--coverage` flag produces LCOV (`coverage/lcov.info`) and HTML (`coverage/html/index.html`) reports. Reference: https://hardhat.org/docs/tutorial/coverage

Then install dependencies using the package manager determined above. If the project has a `postinstall` script that runs Forge commands (e.g., `forge install`), use `--ignore-scripts` to skip it — it's unnecessary for the Hardhat migration and may be slow or fail. For yarn: `yarn install --ignore-scripts`. For npm: `npm install --ignore-scripts`. For pnpm: `pnpm install --ignore-scripts`.

Finally, add Hardhat's output directories to `.gitignore` if not already present:

```
# Hardhat
artifacts/
cache/
```

## Step 3: Create hardhat.config.ts

Create `hardhat.config.ts` by mapping settings from `foundry.toml`. Use the reference tables below for known mappings.

### Config structure: Always use `defineConfig()`

**Always use `defineConfig()`** — it is the recommended pattern across all Hardhat 3 documentation. Do NOT use the `HardhatUserConfig` type annotation approach (a Hardhat 2-era pattern that still compiles but is not the documented way to configure Hardhat 3).

When using plugins (e.g. `hardhat-verify`), pass them in the `plugins` array:

```typescript
import { configVariable, defineConfig } from "hardhat/config";
import hardhatVerify from "@nomicfoundation/hardhat-verify";

export default defineConfig({
  plugins: [hardhatVerify],
  solidity: { ... },
  networks: { ... },
  verify: { ... },  // typed correctly via plugin
});
```

When no plugins are needed:

```typescript
import { defineConfig } from "hardhat/config";

export default defineConfig({
  solidity: { ... },
});
```

### Solidity compiler settings (`solidity`)

Map these from `[profile.default]` in foundry.toml:

| foundry.toml | hardhat.config.ts |
| --- | --- |
| `solc = "0.8.X"` or `solc_version = "0.8.X"` | `solidity.compilers[0].version: "0.8.X"` |
| `optimizer = true` | `solidity.compilers[0].settings.optimizer.enabled: true` |
| `optimizer_runs = 200` | `solidity.compilers[0].settings.optimizer.runs: 200` |
| `evm_version = "cancun"` | `solidity.compilers[0].settings.evmVersion: "cancun"` |
| `via_ir = true` | `solidity.compilers[0].settings.viaIR: true` |
| `bytecode_hash = "none"` | `solidity.compilers[0].settings.metadata.bytecodeHash: "none"` |
| `src = "src"` | `paths.sources: "./src"` |
| `test = "tests"` | `paths.tests: "./tests"` |

**IMPORTANT — `solidity` structure rules:**

- When using per-file `overrides`, you MUST use the `compilers` array format. The top-level `version` field is incompatible with `overrides`.
- When using `profiles`, you CANNOT have top-level `compilers` or `version` — all compiler config must live inside named profiles. A `default` profile is **required**.

```typescript
// WRONG — overrides + version are incompatible
solidity: { version: "0.8.28", overrides: { ... } }

// WRONG — top-level compilers + profiles are incompatible
solidity: { compilers: [...], profiles: { ... } }

// CORRECT — without profiles, use compilers array with overrides
solidity: { compilers: [{ version: "0.8.28", settings: { ... } }], overrides: { ... } }

// CORRECT — with profiles, put everything inside named profiles (default is required)
solidity: { profiles: { default: { compilers: [...] }, production: { compilers: [...], overrides: { ... } } } }
```

### Build profiles (`solidity.profiles`)

Foundry's `[profile.*]` sections, `additional_compiler_profiles`, and `compilation_restrictions` map to Hardhat's build profiles system.

**When to use profiles vs plain compilers:**

- Use `solidity.profiles` when `foundry.toml` has multiple `[profile.*]` sections with **different compiler settings** (e.g., `[profile.debug]` changes `via_ir` or `optimizer_runs`). A `default` profile is required.
- Use plain `solidity.compilers` (without profiles) when there is only `[profile.default]` and no other profiles change compiler settings. Profiles that only override test settings (like `[profile.pr.fuzz]`) do not count — they have no Hardhat build profile equivalent.

**Create ALL Foundry profiles that have compiler settings** — do not leave any unmigrated with a TODO comment if they can be expressed in Hardhat. Every `[profile.*]` section with compiler settings (optimizer, viaIR, optimizer_details, etc.) must become a named Hardhat build profile.

Reference: https://hardhat.org/docs/guides/writing-contracts/build-profiles

| Foundry concept | Hardhat 3 equivalent |
| --- | --- |
| `[profile.default]` base settings | `solidity.profiles.default` (**required** when using profiles) |
| `additional_compiler_profiles` + `compilation_restrictions` | `solidity.profiles.<name>.overrides` with exact file paths |
| `[profile.coverage]` compiler settings | `solidity.profiles.coverage` |

**Key limitations:**

- Build profiles only cover **compiler settings**, NOT test settings (fuzz runs, etc.). Foundry profiles like `[profile.pr.fuzz]` or `[profile.ci.fuzz]` that only change test settings have no direct equivalent — use env vars or CLI args. For settings that can't be expressed in Hardhat profiles (fuzz runs, gas snapshots, isolate, etc.), use the values from `[profile.default]` in the top-level `test.solidity` config and leave a comment explaining that the other Foundry profiles (pr, ci, coverage, gas) override these values but Hardhat profiles don't support test settings.
- If a Foundry profile has **both compiler and test settings** (e.g., `[profile.debug]` sets `via_ir = false` AND `fuzz.runs = 100`), create the Hardhat build profile with only the compiler settings and add a `// TODO:` comment for the test settings that cannot be included.
- Hardhat does NOT yet support glob patterns (`**`) in `overrides`. Each file must be listed individually. See: https://github.com/NomicFoundation/hardhat/issues/4686
- Leave TODO comments for glob overrides that can't be expressed.

Usage: `npx hardhat compile --build-profile production`

**DRY principle for profiles:** When multiple profiles share most compiler settings, extract the common settings into a local variable to avoid duplication:

```typescript
const baseCompilerSettings = {
  optimizer: { enabled: true, runs: 1000000 },
  evmVersion: "shanghai" as const,
  viaIR: true,
};

export default defineConfig({
  solidity: {
    profiles: {
      default: {
        compilers: [{ version: "0.8.23", settings: baseCompilerSettings }],
      },
      lite: {
        compilers: [
          {
            version: "0.8.23",
            settings: {
              ...baseCompilerSettings,
              optimizer: {
                ...baseCompilerSettings.optimizer,
                details: { yulDetails: { optimizerSteps: "" } },
              },
            },
          },
        ],
      },
    },
  },
});
```

### Test settings (`test.solidity`)

Map these from `[profile.default]` or `[fuzz]`/`[invariant]` sections:

| foundry.toml | hardhat.config.ts |
| --- | --- |
| `fuzz.runs = 256` | `test.solidity.fuzz.runs: 256` |
| `fuzz.seed = "0x640"` | `test.solidity.fuzz.seed: "0x640"` |
| `fuzz.max_test_rejects = 65536` | `test.solidity.fuzz.maxTestRejects: 65536` |
| `fuzz.dictionary_weight = 40` | `test.solidity.fuzz.dictionaryWeight: 40` |
| `invariant.runs = 256` | `test.solidity.invariant.runs: 256` |
| `invariant.depth = 500` | `test.solidity.invariant.depth: 500` |
| `invariant.fail_on_revert = false` | `test.solidity.invariant.failOnRevert: false` |
| `ffi = true` | `test.solidity.ffi: true` |
| `block_gas_limit` | `test.solidity.blockGasLimit` |
| `gas_limit = N` | `test.solidity.gasLimit: Nn` (**must be bigint**, use `n` suffix) |
| `fs_permissions = [{ access = "read", path = "./file" }]` | `test.solidity.fsPermissions: { readFile: ["./file"] }` |
| `fs_permissions = [{ access = "write", path = "./file" }]` | `test.solidity.fsPermissions: { writeFile: ["./file"] }` |
| `fs_permissions = [{ access = "read-write", path = "./file" }]` | `test.solidity.fsPermissions: { readWriteFile: ["./file"] }` |
| `fs_permissions = [{ access = "read", path = "./dir" }]` | `test.solidity.fsPermissions: { readDirectory: ["./dir"] }` |
| `fs_permissions = [{ access = "write", path = "./dir" }]` | `test.solidity.fsPermissions: { dangerouslyWriteDirectory: ["./dir"] }` |
| `fs_permissions = [{ access = "read-write", path = "./dir" }]` | `test.solidity.fsPermissions: { dangerouslyReadWriteDirectory: ["./dir"] }` |
| `allow_internal_expect_revert = true` | `test.solidity.allowInternalExpectRevert: true` |
| `isolate = true` | `test.solidity.isolate: true` |

**Note on `isolate`:** The global `isolate = true` in `foundry.toml` maps to `test.solidity.isolate: true`. Per-test inline `/// forge-config: default.isolate = true` overrides are **not yet supported** — `isolate` is not one of the settings available for inline configuration in Hardhat 3.3.0. See [edr#1349](https://github.com/NomicFoundation/edr/issues/1349) for tracking. **Do NOT set `isolate` globally as a workaround** when only some tests use it — it dramatically slows down the entire test suite. Only set it globally if `foundry.toml` has `isolate = true` at the `[profile.default]` level (meaning it was already global in Forge). If only specific tests or contracts use inline `isolate`, document it as a gap.

**Note on inline config scope:** Hardhat only supports inline `forge-config:` directives at the **function level** (on individual test functions). Foundry also supports **contract-level** inline config (directives placed on the contract definition, which apply to all functions in that contract). Contract-level directives are silently ignored by Hardhat.

**When to apply global workarounds for unsupported inline config:** Not all settings are safe to set globally. Evaluate the side effects before applying a global workaround:

- **Safe to set globally:** `allowInternalExpectRevert` — enabling this globally is harmless; it only changes behavior when `vm.expectRevert` is used on internal/library calls, and enabling it on tests that don't use that pattern has no effect.
- **NOT safe to set globally:** `isolate` — this forces each test call to run in a separate EVM context, which dramatically slows down the entire test suite. If only a few test contracts/functions use `isolate`, setting it globally would impose a major performance penalty on all tests. **Do NOT set `isolate` globally as a workaround.** Instead, leave it as a documented gap in the migration report.
- **Use judgement for other settings:** `disableBlockGasLimit`, `evm_version`, etc. — consider whether enabling them globally changes behavior for tests that don't expect it. When in doubt, do NOT set globally; document as a gap instead.

The guiding principle: only apply a global workaround if it is **behaviorally neutral** for tests that don't use the setting. If a global setting would change behavior or performance for unrelated tests, leave it as a gap.

**`fsPermissions` key distinction:** `readFile`/`writeFile`/`readWriteFile` use **exact path matching** (single file). `readDirectory`/`dangerouslyWriteDirectory`/`dangerouslyReadWriteDirectory` use **prefix matching** (recursive directory access). Use the directory variants when the Foundry path points to a directory.

**Note:** This table is not exhaustive. Hardhat 3 may support additional test settings not listed here. When encountering a Foundry test setting without a mapping in this table, check the [Hardhat 3 documentation](https://hardhat.org/docs) and the TypeScript type definitions in `node_modules/hardhat/src/internal/builtin-plugins/solidity-test/type-extensions.ts` before concluding it has no equivalent.

### Network configuration (`networks`)

Map `[rpc_endpoints]` from foundry.toml. Use `configVariable()` for environment variables (lazy resolution — only resolved when needed).

**Note:** Foundry uses `"${VAR_NAME}"` syntax for env vars. Strip the `${}` wrapper when passing to `configVariable("VAR_NAME")`.

**Tip:** The `chainId` for each network can often be found in the `[etherscan]` section of the same foundry.toml.

| foundry.toml | hardhat.config.ts |
| --- | --- |
| `[rpc_endpoints]` `mainnet = "${RPC_MAINNET}"` | `networks.mainnet: { type: "http", chainId: 1, url: configVariable("RPC_MAINNET") }` |

`configVariable()` also supports a format parameter for URL templates:

```typescript
url: configVariable(
  "ALCHEMY_API_KEY",
  "https://eth-mainnet.g.alchemy.com/v2/{variable}",
);
```

Reference: https://hardhat.org/docs/reference/configuration#network-configuration

### Verification / Etherscan (`verify`)

Map `[etherscan]` from foundry.toml. In Foundry, this section configures API keys for contract verification via `forge verify-contract` and `forge create --verify` (ref: https://book.getfoundry.sh/reference/config/etherscan). Requires the `@nomicfoundation/hardhat-verify` plugin in Hardhat 3.

**IMPORTANT:** Hardhat 3 uses Etherscan API v2 which requires a **single API key** for all supported chains. Foundry often has per-chain keys (`ETHERSCAN_API_KEY_MAINNET`, etc.). Document this difference with a comment.

```typescript
verify: {
  etherscan: {
    apiKey: configVariable("ETHERSCAN_API_KEY"),
  },
},
```

### Foundry settings without direct Hardhat equivalents

These foundry.toml settings and commands have no direct Hardhat equivalent. Leave a comment in hardhat.config.ts for each one present in the project.

**Important:** Presence in this table does NOT mean the feature should be classified as 🚩 Gap in the migration report. Each row includes a Status that indicates the actual parity level — many have workarounds, partial support, or are tools that work standalone regardless of build system. Use the Status column to determine the correct parity classification.

| foundry.toml | Status |
| --- | --- |
| `dynamic_test_linking` | Foundry-only |
| `gas_snapshot_check` / `[profile.gas]` | Supported — use `npx hardhat test solidity --snapshot` to generate and `--snapshot-check` to verify. See: https://hardhat.org/docs/guides/testing/gas-snapshots |
| `[bind_json]` | Foundry-only |
| `[fmt]` | 🟡 Partial — `forge fmt` works standalone regardless of build tool; `prettier-plugin-solidity` is a mature alternative |
| `[lint]` | Foundry-only (projects typically use prettier or solhint) |
| `forge doc` | 🟡 Partial — community plugin [`@solarity/hardhat-markup`](https://www.npmjs.com/package/@solarity/hardhat-markup) supports HH3 for NatSpec documentation generation |
| `out = "out"` | Hardhat uses its own `artifacts/` + `cache/` dirs |
| `libs = ["lib"]` | Hardhat resolves `lib/` deps via `remappings.txt`; npm deps via Node.js resolution |
| `[profile.zksync]` / zkSync compilation (`--zksync`, `fallback_oz`, `is_system`, `mode`) | Foundry-only — check whether `@matterlabs/hardhat-zksync` supports HH3 (as of early 2026 it targets HH2 only). If not, zkSync contracts compile but can't be meaningfully tested |
| `/// forge-config:` inline test config | 🟡 Partially supported since Hardhat 3.3.0 — **function-level only**. `forge-config:` prefix is backwards-compatible. Most fuzz/invariant settings and `allowInternalExpectRevert` work inline at function level. **Contract-level** inline config (directives on contract definitions) is NOT supported — use global config as workaround. `isolate`, `evm_version`, and other settings are not yet supported inline even at function level — use global config. See: [docs](https://hardhat.org/docs/guides/testing/inline-configuration), [edr#1349](https://github.com/NomicFoundation/edr/issues/1349) |
| `forge coverage --ir-minimum` | Not needed — Hardhat 3's built-in `--coverage` handles via-IR projects natively without a separate flag. Note: coverage instrumentation may cause some tests to fail that pass without coverage — these are coverage-specific and should be investigated separately. |

**Note:** This table is not exhaustive. Before concluding a Foundry setting has no Hardhat equivalent, check `test.solidity` type definitions in `node_modules/hardhat/src/internal/builtin-plugins/solidity-test/type-extensions.ts` and the [Hardhat 3 documentation](https://hardhat.org/docs).

### IMPORTANT: Unknown settings

If you encounter foundry.toml settings that don't have a clear mapping in the tables above:

1. Check if it fits under `solidity.settings` (which accepts any solc input JSON option)
2. Check if it fits under `test.solidity`
3. **If neither — STOP and ask the user. Do not guess.**

**Do not assume mappings without documentation.** When uncertain about what a Foundry setting does, look up the Foundry docs (https://book.getfoundry.sh/reference/config/) before mapping it to a Hardhat equivalent.

### IMPORTANT: Leave comments for unsupported configs

For any foundry.toml setting that cannot be mapped due to Hardhat limitations:

1. Add a `// TODO:` comment in `hardhat.config.ts` describing the original Foundry setting
2. Include a link to the relevant Hardhat GitHub issue if one exists
3. Explain the workaround (if any) or that no equivalent exists

### IMPORTANT: Apply Forge defaults that differ from Hardhat

After mapping all **explicit** `foundry.toml` settings, consult the **"Reference: Forge vs Hardhat 3 default values"** table at the bottom of this file. For every setting where Forge's default differs from Hardhat's default, **explicitly set that value in `hardhat.config.ts`** — even if the setting is absent from `foundry.toml`.

For example: if `foundry.toml` does not set `optimizer` or `src`, Forge still defaults to `optimizer = true` and `src = "src"`. Hardhat's defaults are different (`optimizer` off, `paths.sources = "./contracts"`), so these must be explicitly configured to preserve the same behavior.

### Inline progress check

After finishing the config migration, briefly confirm with the user which settings were migrated, which are Foundry-only, and which have gaps — before moving on to Step 4. The full detailed report is presented in Step 7.

## Step 4: Handle absolute imports

Hardhat 3 does not support absolute imports in `.sol` files. Scan all `.sol` files for absolute imports (imports that don't start with `./`, `../`, or a package name like `@openzeppelin/`).

Common patterns to look for:

- `import "src/SomeContract.sol";`
- `import "contracts/SomeContract.sol";`
- `import "test/helpers/SomeHelper.sol";`
- `import "lib/some-dep/SomeFile.sol";`

**How to find absolute imports:** Absolute imports come in two Solidity syntaxes. Search for both:

- Direct imports: `import "src/...";`
- Named imports: `import { Foo } from "src/...";`

Use a regex like `import\s.*["'](src|test|lib|contracts)/` to catch both forms and both quote styles (Solidity allows single or double quotes for import paths). Make sure to search **all** directories that contain `.sol` files, including `examples/`, `script/`, and any other non-standard directories — not just `src/` and `test/`.

### Decision rule:

Count the number of **files** containing absolute imports (not the number of import statements):

- **~10 or fewer files** → convert them to relative imports (more compatible with all tools)
- **Significantly more than 10 files** → add remappings to the root `remappings.txt` file (e.g., `src/=./src/`, `tests/=./tests/`, `lib/=./lib/`). This is simpler than creating scoped remappings in subdirectories and works for all files.

**Note:** This is a heuristic, not a strict threshold. When the count is close to the limit (e.g., 8–12 files), use your judgement. Consider the overall complexity of the changes, how many imports per file, and whether the project already uses remappings extensively. When in doubt, prefer relative imports — they are more portable and don't add implicit resolution rules.

Reference: https://hardhat.org/docs/cookbook/absolute-imports

## Step 5: Compile

**IMPORTANT:** Only compile and test using the **default** build profile. Non-default profiles (production, coverage, etc.) are migrated for completeness but should NOT be compiled or tested during this migration session. They may require environment-specific tooling (e.g., native solc for viaIR) that isn't available.

Run and store the output:

```
npx hardhat compile --show-stack-traces 2>&1 | tee /tmp/hardhat-compile-output.txt
```

Do NOT run `npx hardhat compile --build-profile production` or any other profile.

**Output caching:** If `/tmp/hardhat-compile-output.txt` already exists, analyze whether any relevant changes have been made since it was written. If not, read the stored file instead of re-running the command. Only re-run when it is justified. When re-running, **state in your response what was fixed and why the re-run is necessary** before issuing the command — e.g., _"Added `lib/=./lib/` remapping to fix HHE902 error on absolute lib imports — re-compiling."_

If compilation fails, **group errors by type** before presenting them to the user. Then:

1. Fix import path issues, missing dependencies, or config problems
2. If you see `HHE902: ... is not exported by the package`, see the "npm packages with restrictive `exports` fields" reference section
3. If you see Solidity type mismatch errors involving types from shared dependencies, see the "Transitive dependency version conflicts" reference section
4. Re-run (overwriting the cached output file) until compilation succeeds

Do not proceed to Step 6 until compilation passes cleanly.

## Step 6: Run Solidity tests

Run and store the output:

```
npx hardhat test solidity --show-stack-traces 2>&1 | tee /tmp/hardhat-test-output.txt
```

Do NOT run tests with non-default build profiles.

**Output caching:** If `/tmp/hardhat-test-output.txt` already exists, analyze whether any relevant changes have been made since it was written. If not, read the stored file instead of re-running the command. Only re-run when it is justified. When re-running, **state in your response what was fixed and why the re-run is necessary** before issuing the command — e.g., _"Commented out UnsupportedCheatcode tests — re-running to confirm remaining tests pass."_

If tests fail:

1. Analyze the failures — distinguish between real test failures vs migration issues
2. **Group errors by type** before presenting them to the user
3. **Handle `UnsupportedCheatcode` errors:**
   - **ONLY** apply this procedure to tests whose error line reads exactly `Error: UnsupportedCheatcode: vm.xyz(...)`. Do **NOT** comment out tests that fail with any other error type — even if those errors appear migration-related. All other failures must be handled via items 4–6 below (fix or flag to the user).
   - For each test function that triggers an `UnsupportedCheatcode` error, comment out the **entire function** (signature + body) using `//` line comments, and **preserve the original code** as commented-out lines so the disabled test remains readable. Use `/* */` block comments only if the function body itself contains no block comments. Prefer `//` line comments in all cases to avoid nesting issues.
   - Add the HARDHAT-SKIP header immediately before the commented-out function, e.g.:
     ```solidity
     // HARDHAT-SKIP: This test uses vm.xyz() which is not supported by Hardhat 3 (UnsupportedCheatcode).
     // See: https://github.com/NomicFoundation/hardhat/issues/XXXX
     // function test_foo() public {
     //   vm.xyz("some", abi.encode(param));
     //   assertEq(result, expected);
     // }
     ```
     **Do NOT** replace the original function body with an empty stub. An empty `test_foo()` would pass trivially and give a false green — the test must not run at all.
   - Re-run the tests (overwriting the cached output file) to confirm remaining tests pass
4. **Handle suspected EDR / Hardhat bugs:**
   - Some test failures are neither `UnsupportedCheatcode` nor fixable by editing the config — they indicate a behavioral difference or outright bug in Hardhat / EDR. These tests should **not** be modified, commented out, or worked around — the test code is correct Foundry-idiomatic code.
   - When you suspect a group of failures has the same root cause, **investigate the call chain** to confirm. Trace from the failing test function through any helpers or modifiers to identify exactly which cheatcode call returns unexpected results or throws unexpectedly.
   - **Group tests by root cause** — tests failing for the same reason should be treated as a single bug, not as separate issues.
   - Once the root cause is confirmed, **write a structured bug report** to `.claude/bugs/<descriptive-slug>.md` (create the `.claude/bugs/` directory if needed). **Use a slug that describes the affected feature or cheatcode — do NOT prefix with `edr-` or `hardhat-`** (e.g. `vm-readCallers-prank-state.md`, not `edr-vm-readCallers-prank-state.md`). The responsible layer will be determined through the upstream filing process. The file should include:
     - Issue title suitable for filing as a GitHub issue on [NomicFoundation/hardhat](https://github.com/NomicFoundation/hardhat/issues) or [NomicFoundation/edr](https://github.com/NomicFoundation/edr/issues) depending on where the root cause lies
     - A self-contained minimal Solidity reproduction (small test contract + minimal `hardhat.config.ts`)
     - Steps to reproduce
     - Expected vs actual behaviour (table form)
     - The observed error message
     - The exact failing call chain with file paths and line numbers linking into the project
     - Which test(s) fail (file, contract, function name)
     - Suggested fix direction for the EDR team (reference Foundry's equivalent implementation if helpful)
     - The available workaround, if any — plus an explicit note stating the workaround is **not applied** and why (the test code is correct; applying it would mask the bug and add unnecessary boilerplate)
   - Leave the failing tests as-is in the codebase. Do not comment them out. They are genuine blockers.
5. Fix other migration-related issues (import paths, config differences)
6. For real test failures unrelated to migration, flag them to the user
7. If failures persist after applying fixes, ask the user whether they want to keep iterating or proceed directly to generating a **Failed** verdict report

**Common error patterns and fixes:**

- `"call didn't revert at a lower depth than cheatcode call depth"` or `"reverted with an unrecognized custom error"` on tests using `vm.expectRevert` on internal/library calls → enable `test.solidity.allowInternalExpectRevert: true` in config (maps Foundry's `allow_internal_expect_revert`)
- Gas-related failures, out-of-gas reverts, or unexpected reverts in large tests → check that `test.solidity.gasLimit` matches `foundry.toml`'s `gas_limit` (must be a bigint with `n` suffix)
- `vm.envString: environment variable "X" not found` — tests that read env vars via `vm.envString()` or `vm.envOr()` in `setUp()` will fail if those vars aren't set. Fix: set the required env var in the Hardhat test script in `package.json` (e.g., `"test-hardhat": "MY_VAR=value npx hardhat test solidity"`)

When tests pass, **always show the full test output to the user** so they can verify the results themselves.

**Coverage verification (optional):** If the project has a `forge coverage` script in `package.json`, run `npx hardhat test solidity --coverage` after tests pass to verify coverage works. Coverage instrumentation can reveal issues not seen in normal test runs. Compare the number of passing tests with and without `--coverage` — any discrepancy should be noted in the migration report. Note: Hardhat 3's coverage handles via-IR projects natively; Forge's `--ir-minimum` flag has no equivalent and is not needed.

## Step 7: Migration report

After compilation and tests complete (whether all pass or not), write the report to `.claude/<project-name>-hardhat-migration-report.md` in the repository root and present it to the user. The report must assess **feature parity** between Forge and Hardhat for this project and give a clear verdict.

**Report structure:** The report uses numbered `##` sections. Conditional sections are omitted when empty — remaining sections are numbered contiguously (no gaps). The possible sections in order are:

1. **Test Count Comparison** — always included
2. **Feature Parity** — always included
3. **Hardhat / EDR Bug Reports** — only if bugs were found in `.claude/bugs/`
4. **Workarounds Applied** — always included
   - Sub-section: **UnsupportedCheatcode errors** — only if test functions were commented out
5. **Next Steps** — always included

When conditional sections are omitted, renumber the remaining sections contiguously. For example, if there are no bug reports and no commented-out tests: 1. Test Count, 2. Feature Parity, 3. Workarounds, 4. Next Steps.

### Test count comparison

Count the number of `function test*` declarations in `.t.sol` files under the test directory (ignore `example/` and `examples/` directories). Compare against the number of tests Hardhat actually ran. Flag any discrepancy.

### Feature parity

The section is split into two parts to keep width manageable:

1. **Gaps, bugs & partial support** — a focused 4-column table for ❌ Bug, 🚩 Gap, and 🟡 Partial rows
2. **Full parity** — a compact bullet list for ✅ Full features, followed by a "Features not used by this project" bullet list

For each Forge feature **used by the project**, assess Hardhat 3 parity. Only include entries for features the project actually uses (check `foundry.toml`, `package.json` scripts, `script/` directory, test files).

The Parity column uses four values — use them precisely, and always include the corresponding emoji:

- ❌ **Bug** — the feature exists in Hardhat/EDR but behaves incorrectly relative to the Forge spec; a bug report has been filed in `.claude/bugs/`
- 🚩 **Gap** — the feature is absent or unsupported in Hardhat (missing cheatcode, no equivalent config, etc.)
- 🟡 **Partial** — the feature exists but with meaningful limitations (e.g. can only be set globally, not per-test)
- ✅ **Full** — Hardhat behavior is equivalent to Forge

**IMPORTANT — verify before classifying as 🚩 Gap:** Before marking any Forge feature as 🚩 Gap, check whether a Hardhat equivalent exists via:

1. Built-in Hardhat 3 features (docs, CLI flags like `--coverage`)
2. [Official plugins](https://hardhat.org/docs/plugins/official-plugins)
3. [Community plugins](https://hardhat.org/docs/plugins/community-plugins)

HH3 compatibility is a hard requirement. If a community plugin exists but only supports HH2 (`peerDependencies` does NOT include `hardhat ^3.0.0`), classify as 🚩 **Gap** (not 🟡 Partial). Mention the HH2 plugin in the Workaround / Notes column as context (e.g., "HH2 plugin exists: `<package-name>`"), but the parity rating must reflect what's usable with HH3 today. Check the [HH2 plugin directory](https://v2.hardhat.org/hardhat-runner/plugins) for plugins that may not yet have HH3 support. Only classify as 🟡 **Partial** if the plugin's `peerDependencies` actually includes `hardhat ^3.0.0`. Verify via the package manager (e.g., `npm view <pkg> peerDependencies`).

**Order rows from most to least severe:** ❌ Bug first, then 🚩 Gap, then 🟡 Partial.

**IMPORTANT — cross-reference checkpoint:** Before finalizing the gap table, re-read the analysis file from Step 1 and verify that **every** item is accounted for in the report — every Forge-dependent `package.json` script, every Foundry-only `foundry.toml` setting, every inline `forge-config:` usage, and every unsupported cheatcode must have a corresponding row in the gap table or an entry in the full parity list. Do not rely on memory — explicitly check each item off against the analysis file.

#### Canonical feature names

To ensure consistent gap reporting across different repos, **always use the exact feature names below** in the Feature column. Do not paraphrase, abbreviate, or add parenthetical clarifications — the dashboard aggregates gaps by feature name across repos, and inconsistent naming prevents deduplication.

| Canonical feature name | When to use |
| --- | --- |
| `Built-in formatter` | Forge's built-in formatter (`[fmt]` config section, `forge fmt` command) |
| `Documentation generator` | Forge's documentation generator (`forge doc`) |
| `Deployment scripting` | Deployment/scripting via `.s.sol` files (`forge script`, `vm.startBroadcast()`). Also use this name for `vm.startBroadcast()` / `vm.stopBroadcast()` gaps — these cheatcodes only function inside `forge script` and are not a separate feature |
| `ABI binding generation` | ABI binding generation (`forge bind --alloy`, etc.) |
| `Gas snapshots` | Gas snapshots (`forge snapshot`, `gas_snapshot_check`, `.forge-snapshots/`). Hardhat equivalent: `--snapshot` / `--snapshot-check` CLI flags |
| `Code coverage` | Code coverage (`forge coverage`, `--ir-minimum` flag; note: Hardhat 3 handles via-IR natively) |
| `Inline test config` | Per-test overrides via `/// forge-config:` or `/// hardhat-config:` comments |
| `Etherscan verification` | Contract verification via Etherscan (`[etherscan]` config) |
| `Gas reports` | Gas reporting (`forge test --gas-report`, `gas_reports` in foundry.toml) |
| `Fuzz/invariant profile overrides` | Per-profile fuzz/invariant settings (`[profile.ci.fuzz]`, `[profile.pr.fuzz]`, `[profile.debug]` fuzz/invariant overrides, etc.) |
| `Test path filtering` | `no_match_path`, `match_path`, test exclusion patterns in config |
| `Glob overrides` | Glob patterns (`**`) in compiler overrides |
| `Dynamic test linking` | Dynamic test linking config (`dynamic_test_linking` in foundry.toml) |
| `Compiler warning suppression` | Suppressing compiler warnings by source (`ignored_warnings_from` in foundry.toml) |
| `Memory limit` | Per-test or global memory limit (`memory_limit` in foundry.toml) |

If a feature is not listed above, use a short, descriptive name that starts with the Forge command or config section name (e.g., `forge verify-contract`, `[profile.solx]`). Avoid adding parenthetical synonyms — pick one name and use it.

#### Gaps, bugs & partial support table

Use this 4-column format (the Forge and Hardhat 3 columns are implicit — omit them):

| Feature | Parity | Impact | Workaround / Notes |
| --- | --- | --- | --- |
| `vm.someCheatcode()` | ❌ **Bug** | **High** — N tests failing; genuine blocker | [Local bug report](bugs/vm-some-cheatcode.md) (not yet filed upstream) — test code not modified; fix needed in Hardhat / EDR |
| `vm.unsupported()` | 🚩 **Gap** | **High** — M test instances disabled; layer X is untested | No tracking issue found — consider filing one; workaround: implement equivalent logic in a Solidity helper |
| Gas snapshots | 🚩 **Gap** | **Medium** — tests run but snapshots can't be generated | [#7769](https://github.com/NomicFoundation/hardhat/issues/7769) — no workaround currently |
| Etherscan verification | 🟡 **Partial** | **Low** — key model differs; minor CI update needed | Consolidate to a single `ETHERSCAN_API_KEY` — Etherscan API v2 accepts one key across all supported chains |

The Impact cell must start with a severity word (**High** / **Medium** / **Low**) followed by an em-dash and a short explanation of what breaks.

#### Full parity list

List ✅ Full features as a bullet list under the subheading `### Full parity`, preceded by the sentence "These features work equivalently in Hardhat 3:". Then add a **"Features not used by this project:"** bullet list for features the project doesn't use — do NOT add these as rows in the gap table.

Example:

```markdown
### Full parity

These features work equivalently in Hardhat 3:

- Solidity compilation (`forge build` → `npx hardhat compile`)
- forge-std cheatcodes (`vm.*`) — general

**Features not used by this project:**

- Deployment scripting (`forge script` / `.s.sol`) — project has no `script/` directory
```

If inline test config was detected in Step 1, check which inline settings the project uses and at what level (function vs contract). Inline config is 🟡 **Partial** if the project uses:

- **Contract-level** inline config (directives on contract definitions) — Hardhat only supports function-level; contract-level directives are silently ignored. The workaround is to set the value globally in `hardhat.config.ts`.
- **Unsupported inline settings** even at function level (notably `isolate`, `evm_version` — see [edr#1349](https://github.com/NomicFoundation/edr/issues/1349))

List the affected files and specify whether the issue is contract-level scope or unsupported setting. Only classify as ✅ Full parity if all inline directives are at function level AND use supported settings (fuzz/invariant settings, `allowInternalExpectRevert`).

If any `UnsupportedCheatcode` errors were encountered, add one row per distinct unsupported cheatcode to the gap table above.

For each 🚩 **Gap** and ❌ **Bug**, populate the Workaround / Notes column with a tracking issue link and/or a concrete suggested path forward. "No workaround currently" is a valid entry when none exists — don't leave the cell blank.

**Before including any GitHub issue link**, fetch the issue URL and confirm its title and description are actually related to the feature being documented. Do not guess issue numbers. If no relevant issue exists, write "No tracking issue found — consider filing one" instead of leaving the cell empty or inventing a link.

**Distinguish local bug reports from upstream issues.** When linking to a file in `.claude/bugs/`, use the link text **"Local bug report"** and add "(not yet filed upstream)" next to it — never "Bug report filed", which implies the issue has already been created in the upstream tracker. Only use "filed upstream" if you have actually opened a GitHub issue and have a real issue URL to link to.

**Environment variables read by tests** (e.g., via `vm.envString`) are a configuration concern, not a feature parity issue. Handle them by setting the required env vars in the Hardhat script definitions in `package.json`. Do NOT add them as rows in the gap table.

**zkSync-specific profiles with dedicated contract variants** (identified in Step 1, item 9) should be classified as 🚩 **Gap** with impact proportional to the percentage of codebase affected. When tests conditionally deploy different contracts depending on the active profile, the zkSync path is untested under Hardhat — flag this in the Impact column.

**Check for duplicate or overlapping rows** before finalizing the table. The same Forge feature can appear under multiple angles (e.g., per-profile vs per-file). Consolidate overlapping rows into one row in the gap table, with a Notes column that explains the combined limitation.

### Hardhat / EDR bug reports (conditional — tests left failing)

**Only include this section if behavioral bugs were identified in Step 6 (item 4).** If no bugs were found (no files written to `.claude/bugs/`), omit this section entirely from the report.

For each behavioral bug identified in Step 6 (item 4), reference the corresponding bug report file written to `.claude/bugs/`. Use the heading `## N. Hardhat / EDR Bug Reports` (where N is the next section number) — using both names since the culprit may be in Hardhat's test runner layer, EDR, or both.

In the report:

- In the **gap table**, set Parity to ❌ **Bug** and put the link in the **Workaround / Notes column** as `[Local bug report](bugs/vm-foo-bar.md) (not yet filed upstream) — <one-line summary>`
- In this section, list each failing test group with:
  - The root-cause summary (one sentence)
  - The exact failing tests (file, contract, function)
  - A link to the local bug report file
  - A note confirming the tests are **not** commented out — they are active blockers

The verdict for any session that produces a bug report is **Failed**.

### Workarounds applied

List every non-obvious change made to get compilation/tests working:

- Patches applied (`patch-package`) and why
- Package manager overrides (`pnpm.overrides`, npm `overrides`, yarn `resolutions`) and why
- Import path rewrites (absolute → relative)
- Disabled scripts and reason
- Any other config adjustments

**For each workaround, explain the root cause** — not just what was changed, but why it was necessary and how Forge avoids the same issue. When the workaround relates to a difference in how Forge and Hardhat 3 resolve dependencies, explain both resolution models (Forge's global remappings vs Hardhat 3's Node.js module resolution) so the reader understands why the workaround is needed and what it reproduces. Reference the relevant sections from this skill file (e.g., "Transitive dependency version conflicts", "npm packages with restrictive `exports` fields") for the underlying technical context.

### UnsupportedCheatcode errors (conditional — commented out)

**Only include this section if test functions were actually commented out due to `UnsupportedCheatcode` errors.** If no functions were commented out, omit this section entirely from the report. When included, number it as a sub-section of Workarounds Applied (e.g., `## 3b.` or `## 4b.` depending on the Workarounds section number).

List every test function that was fully commented out (signature + body) due to `UnsupportedCheatcode` errors. These functions do not appear in the Hardhat test run at all — they are not counted in the passing total. The discrepancy between Forge's test count and Hardhat's running count reflects the number of commented-out functions.

For each commented-out function, include:

- File path and function name
- The specific cheatcode that triggered the error
- A link to the Hardhat tracking issue if one exists — **verify the issue is relevant before linking** (fetch the URL and confirm the title matches; do not guess issue numbers)

Each distinct cheatcode that causes commented-out tests must have a corresponding row in the **gap table** (section 2) — the UnsupportedCheatcode section provides the per-function detail.

### Verdict header & next steps

The report header (immediately below the title) must contain:

- **Hardhat version installed** — read from `devDependencies.hardhat` in `package.json`
- **Migration date**
- **Foundry analysis** — link to the analysis file from Step 1 using a relative path from the report file (both are in `.claude/`, so use just the filename: `[Foundry analysis](<project-name>-foundry-migration-analysis.md)`)
- **Verdict** — one of the four values below, with the corresponding emoji
- **Blockers** bullet list — ❌ for EDR bugs, 🚩 for gaps that prevent tests from running (commented-out tests, missing cheatcodes). Only include issues that directly block test execution or coverage.
- **Notable gaps** bullet list — 🚩 / 🟡 for gaps and partial features that don't block tests but affect the project's workflow. **Only include Medium or High impact items** (as rated in the gap table) — Low impact gaps belong in the table only. Use the heading `### Notable gaps (non-blocking, medium+ impact)`. Omit the section entirely if no medium+ gaps exist.

Verdict values:

| Verdict | Emoji | Condition |
| --- | --- | --- |
| **Successful** | ✅ | Full feature parity, all tests pass, nothing commented out |
| **Successful with gaps** | 🟡 | All tests pass, but some used Forge features have no Hardhat equivalent |
| **Partial** | 🟡 | Tests pass but functions were commented out — coverage is reduced |
| **Failed** | ❌ | Compilation or tests fail |

Example header for a Failed result:

```markdown
**Hardhat version installed:** `^3.1.10` **Migration date:** 2026-03-03 **Foundry analysis:** [Foundry analysis](<project-name>-foundry-migration-analysis.md)

---

**Verdict:** ❌ **Failed**

### Blockers

- ❌ `vm.someCheatcode()` bug — N tests failing ([local bug report](bugs/vm-some-cheatcode.md), not yet filed upstream)
- 🚩 `vm.unsupported()` unsupported — M test instances commented out

### Notable gaps (non-blocking, medium+ impact)

- 🟡 Inline test config partially supported — `isolate` and `evm_version` not yet available inline
```

The final body section is **Next Steps** — it is always the last numbered section in the report. The verdict itself lives in the header, not here. This section contains a short numbered list of recommended actions, **ordered from most to least impact**. Each item must name the concrete action, state the impact justification (test count, severity, or workflow scope), and include a link where relevant. Three to five items is the target length — don't pad with low-impact items already covered by the gap table.

## Update an existing migration

This flow is triggered when the argument is `update`. It assumes a previous migration has already been completed — the project has `hardhat.config.ts`, the analysis file (`.claude/<project-name>-foundry-migration-analysis.md`), and the migration report (`.claude/<project-name>-hardhat-migration-report.md`).

**Goal:** Bring the migration up to date — upgrade Hardhat, re-evaluate every gap/partial/bug in the report against the current Hardhat version, remove workarounds that are no longer needed, and regenerate the report.

Follow these steps in order.

### Update Step 1: Read existing state

1. Read the existing migration report (`.claude/<project-name>-hardhat-migration-report.md`) and analysis file (`.claude/<project-name>-foundry-migration-analysis.md`).
2. Read `hardhat.config.ts` and `package.json` to understand the current configuration.
3. Read `foundry.toml` to cross-reference against the original Foundry settings.
4. Note the currently installed Hardhat version from `package.json`.

### Update Step 2: Upgrade Hardhat

1. Fetch the latest Hardhat version: run `<package-manager> view hardhat version` (use the same package manager as the original migration — check the lockfile).
2. Update `package.json` to use `"hardhat": "^<latest-version>"`.
3. Run `<package-manager> install` (with `--ignore-scripts` if needed, same as original migration).
4. Note the version change for the updated report header.

### Update Step 3: Remove obsolete global config workarounds

Since Hardhat 3.3.0, inline `forge-config:` directives are supported for most settings at the **function level**. Review `hardhat.config.ts` for global settings that were added **solely** as workarounds for missing inline config support:

1. **Check each global test setting** in `test.solidity` (e.g., `allowInternalExpectRevert`, `disableBlockGasLimit`) against the inline config that uses it.
2. **For each setting:** If the setting was set globally only because inline `forge-config:` was not supported (check the report's "Workarounds Applied" section and config comments), AND the setting is now supported inline by Hardhat (see [inline config docs](https://hardhat.org/docs/guides/testing/inline-configuration)):
   - Check whether the inline directive is at **function level** or **contract level**
   - If **function-level**: the global workaround can be removed — Hardhat supports it inline
   - If **contract-level**: the global workaround must stay — Hardhat does NOT support contract-level inline config. The global setting is still the only option.
   - Remove any related comments/TODOs that are no longer accurate
3. **Do NOT remove global settings** that are:
   - Required regardless of inline config (e.g., `gasLimit`, `fsPermissions`, `fuzz.*` — these are project-wide defaults from `foundry.toml`, not workarounds)
   - Workarounds for **contract-level** inline config (Hardhat only supports function-level)
   - For settings not yet supported inline even at function level (notably `isolate`, `evm_version` — check [edr#1349](https://github.com/NomicFoundation/edr/issues/1349) for current status)
4. Update comments in `hardhat.config.ts` about inline config support status (e.g., if the comment says "silently ignored", update it to reflect current support level — noting function-level vs contract-level distinction).

### Update Step 4: Re-evaluate every gap

Go through **every row** in the existing report's feature parity table (gaps, bugs, and partial support). For each one:

1. **Check if the gap has been resolved** in the current Hardhat version:
   - If the row references a GitHub issue, fetch it and check if it's been closed/resolved
   - Check the [Hardhat changelog](https://hardhat.org/docs/reference/changelog) and [EDR changelog](https://github.com/NomicFoundation/edr/releases) for relevant changes
   - For `UnsupportedCheatcode` entries, check if the cheatcode is now supported by running a quick test or checking EDR release notes
2. **If a gap is resolved:**
   - If tests were commented out with `// HARDHAT-SKIP` for this gap, **uncomment them**
   - Move the feature from the gap table to the full parity list
   - Remove any workaround code/config that was added for this gap
3. **If a gap is partially resolved** (e.g., new inline config settings added but not all), update the parity level and notes accordingly
4. **If a gap remains unchanged**, keep it as-is

### Update Step 5: Compile and test

1. Run compilation: `npx hardhat compile --show-stack-traces 2>&1 | tee /tmp/hardhat-compile-output.txt`
2. Fix any compilation errors introduced by the config changes (removed workarounds, uncommented tests, etc.)
3. Run tests: `npx hardhat test solidity --show-stack-traces 2>&1 | tee /tmp/hardhat-test-output.txt`
4. If previously commented-out tests now fail with a **different** error than `UnsupportedCheatcode`, investigate:
   - If it's a genuine test failure or new bug, handle per Step 6 rules (items 4–6)
   - If the cheatcode is still unsupported, re-comment with `// HARDHAT-SKIP`
5. Show the full test output to the user.

### Update Step 5b: Verify gas snapshots

If the project uses Forge gas snapshots (check for `forge snapshot` in `package.json` scripts, `gas_snapshot_check` in `foundry.toml`, `[profile.gas]` section, or `.forge-snapshots/` directory):

1. **Generate snapshots:** Run `npx hardhat test solidity --snapshot 2>&1 | tee /tmp/hardhat-snapshot-output.txt`. This writes gas snapshots to `.gas-snapshot` (or the configured location).
2. **Verify snapshots:** Run `npx hardhat test solidity --snapshot-check 2>&1 | tee /tmp/hardhat-snapshot-check-output.txt`. This compares current gas usage against the stored snapshots.
3. **If both commands succeed:**
   - Gas snapshots are working — remove the `Gas snapshots` row from the gap table (or move it to the full parity list)
   - Remove any comments in `hardhat.config.ts` about gas snapshots not being supported
   - Update the report's blockers/notable gaps sections accordingly
4. **If either command fails:**
   - Keep the `Gas snapshots` gap row as-is
   - Note the specific error in the gap table's Workaround / Notes column

### Update Step 6: Update the report

Regenerate the migration report following the same structure as Step 7 of the original migration:

1. **Update the header:**
   - Update **Hardhat version installed** to the new version
   - Update **Migration date** to today's date
   - Re-assess the **Verdict** based on current state
   - Update **Blockers** and **Notable gaps** lists
2. **Update test count comparison** — re-count `function test*` declarations and compare against the new test run
3. **Update the feature parity table** — reflect all changes from Update Step 4
4. **Update workarounds applied** — remove entries for workarounds that were removed, add any new ones
5. **Update UnsupportedCheatcode section** — remove entries for cheatcodes that are now supported, update counts
6. **Update Next Steps** — re-prioritize based on current state

### Update Step 7: Update the analysis file

Review the analysis file (`.claude/<project-name>-foundry-migration-analysis.md`) and update any sections that reference Hardhat support status (e.g., inline config notes). The analysis file primarily documents the Foundry project structure and should not change much, but notes about Hardhat compatibility should reflect the current state.

## Reference: Common dependency mappings

When converting git submodules (in `lib/`) to npm packages:

| Foundry (lib/)                       | npm package                           |
| ------------------------------------ | ------------------------------------- |
| `openzeppelin-contracts`             | `@openzeppelin/contracts`             |
| `openzeppelin-contracts-upgradeable` | `@openzeppelin/contracts-upgradeable` |
| `solmate`                            | `solmate`                             |
| `solady`                             | `solady`                              |
| `prb-math`                           | `@prb/math`                           |
| `prb-proxy`                          | `@prb/proxy`                          |

Note: Hardhat 3 **natively** resolves dependencies from both `node_modules/` and `lib/` via `remappings.txt` — no plugin required. The optional `@nomicfoundation/hardhat-foundry` plugin extends this (e.g., reading remappings from `foundry.toml`), but is not needed for standard `remappings.txt` support. In most cases, submodule dependencies in `lib/` can be kept as-is without converting to npm packages.

### Important: Hardhat 3 vs Forge dependency resolution

Hardhat 3 uses **Node.js module resolution** for Solidity imports, not Forge's remapping system ([Hardhat 3 dependencies docs](https://hardhat.org/docs/guides/writing-contracts/dependencies)). This has two key implications:

1. **Remappings targeting `node_modules/` are redundant.** Remappings like `@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/` duplicate what Node.js resolution already does. Hardhat 3 only needs remappings for non-npm paths (e.g., `src/=./src/` for absolute imports, or `lib/`-based submodule paths).

2. **Project-level remappings do not propagate into `node_modules/`.** Each `remappings.txt` [only affects files in its directory and subdirectories, excluding `node_modules/`](https://hardhat.org/docs/guides/writing-contracts/remappings). This is a key difference from Forge, where a remapping like `@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/` applies globally — forcing all packages to resolve to the same copy. In Hardhat 3, each npm package resolves its own dependencies via Node.js resolution, respecting the package manager's isolation rules. See the "Transitive dependency version conflicts" reference section for how this can cause Solidity type mismatches.

### Submodule transitive dependencies

When git submodules in `lib/` import npm-style packages (e.g., `@openzeppelin/contracts/`), those imports won't resolve because submodules don't have their own `node_modules/`. In Forge, `libs = ["lib"]` + global remappings handle this transparently.

In Hardhat 3, the root `remappings.txt` applies to all files in the project directory and subdirectories **except `node_modules/`**. Since `lib/` is not `node_modules/`, root remappings DO apply to files inside `lib/` submodules.

**Fix:** Add remappings to the root `remappings.txt` for any npm-style imports used by submodules:

```
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
```

**Discovery:** During compilation, `HHE902` errors referencing imports from files inside `lib/` (e.g., `"@openzeppelin/contracts/..." from "./lib/some-dep/contracts/Foo.sol"`) indicate this pattern. Check the submodule's own `package.json` to identify what npm packages it depends on, then add root remappings pointing to the corresponding `lib/` submodules.

## Reference: Hardhat Foundry compatibility

Hardhat 3 has native Foundry compatibility built into its core (no plugin required):

- Runs Solidity tests written for Foundry (forge-std cheatcodes supported via EDR)
- Loads `remappings.txt` automatically
- Resolves `lib/` dependencies through remappings
- Supports fuzz testing and invariant testing

**Note on forge-std resolution:** The Hardhat allowlist (`NPM_PACKAGES_WITH_SIMULATED_PACKAGE_EXPORTS`) only applies to forge-std installed via npm. When forge-std is a git submodule in `lib/forge-std/`, you must add an explicit remapping: `forge-std/=lib/forge-std/src/`. Similarly, any other submodule that isn't auto-discovered needs an explicit remapping (e.g., `murky/=lib/murky/`).

**Remapping target path pitfall:** When adding remappings for submodules, always verify the actual import paths used in the codebase — don't assume the standard `lib/<name>/src/` pattern. For example, if the codebase uses `import "murky/src/Merkle.sol"`, the correct remapping is `murky/=lib/murky/` (so it resolves to `lib/murky/src/Merkle.sol`). Using `murky/=lib/murky/src/` would incorrectly resolve to `lib/murky/src/src/Merkle.sol`. Check actual import statements before setting the remapping target.

Docs: https://hardhat.org/docs/reference/foundry-compatibility

## Reference: Transitive dependency version conflicts (Solidity type mismatch)

Package managers with strict dependency isolation can cause Solidity **type mismatches** when two packages depend on different versions of the same library. Each package gets its own copy of its declared dependency version, and Solidity treats identically-named types from different file paths as incompatible.

**Which package managers are affected:**

- **pnpm** — strict isolation by default; this is the primary package manager where this issue occurs
- **npm** — hoists dependencies by default (flat `node_modules`), which naturally deduplicates to one version; unlikely to hit this issue unless using npm's experimental isolated mode
- **yarn classic** — hoists by default, similar to npm; unlikely to hit this issue

**Why this doesn't happen in Forge:** Forge uses `remappings.txt` where a remapping like `@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/` applies globally across all compilation contexts ([Forge remappings docs](https://book.getfoundry.sh/reference/forge/forge-remappings/)). Every package sees the same types regardless of its declared dependency version.

**Fix:** Force all transitive copies to a single version using the package manager's override mechanism:

| Package manager | Field | Example |
| --- | --- | --- |
| pnpm | `pnpm.overrides` in `package.json` (or `overrides` in `pnpm-workspace.yaml`) | `"@openzeppelin/contracts": "$@openzeppelin/contracts"` |
| npm | `overrides` in `package.json` | `"@openzeppelin/contracts": "5.4.0"` (use explicit version — `$` prefix has a [known bug with scoped packages](https://github.com/npm/cli/issues/5078)) |
| yarn | `resolutions` in `package.json` | `"@openzeppelin/contracts": "5.4.0"` |

The `$` prefix (pnpm) means "use the version declared in this project's own `dependencies`/`devDependencies`". This reproduces Forge's flat resolution behavior at the package manager level.

**When to apply:** Check for this issue when compilation fails with type mismatch errors involving types from shared dependencies and the project has transitive dependencies that declare different versions of those shared libraries.

## Reference: npm packages with restrictive `exports` fields

Hardhat 3 respects Node.js `package.json` `exports` fields. Some Solidity packages ship with an `exports` field that only exposes JS/TS entry points, **not** their `.sol` contracts. This causes `HHE902` errors like:

```
The file "contracts/SomeFile.sol" is not exported by the package.
```

**Why standard remappings don't help:** When a remapping target starts with `node_modules/` and the prefix matches the target minus `node_modules/` (e.g., `@foo/bar/=node_modules/@foo/bar/`), Hardhat silently skips it as a no-op. The import then goes through npm resolution, which enforces the `exports` field.

**Why `./node_modules/` doesn't work either:** Using `./node_modules/` as the target prefix makes Hardhat treat it as a local remapping, which bypasses exports. However, this breaks **internal relative imports** within the package (e.g., `import "./Transient.sol"` inside the package) because Hardhat no longer treats those files as part of an npm package.

**Note:** `forge-std` is exempt from this problem — Hardhat has it hardcoded in a special allowlist (`NPM_PACKAGES_WITH_SIMULATED_PACKAGE_EXPORTS`).

### Fix: use `patch-package`

1. Add `patch-package` to `devDependencies` and add a `"postinstall": "patch-package"` script
2. Run an install using the package manager chosen in Step 1 (e.g., `pnpm install`, `npm install`, or `yarn`) so `patch-package` is available (this will also run `postinstall`, which is fine — it finds no patches yet)
3. Edit the package's `package.json` in `node_modules/` to add the missing exports (e.g., `"./contracts/*.sol": "./contracts/*.sol"`). **Be precise** — use `*.sol` suffix to export only Solidity files, not the entire directory
4. Run `npx patch-package <package-name> --exclude nothing` to generate a patch file in `patches/`

**IMPORTANT — ordering:** Steps 2 and 3 must happen in this order. If you edit `node_modules/` before running the install, the install will re-resolve dependencies and **revert your edits**. Always install first, then edit, then generate the patch.

The `--exclude nothing` flag is needed because `patch-package` ignores `package.json` changes by default.

## Reference: Forge vs Hardhat 3 default values

Forge and Hardhat have different defaults for several settings. When a Forge project does NOT explicitly set a value in `foundry.toml` (relying on Forge's defaults), the Hardhat config **must still explicitly set** that value if Hardhat's default differs. Silently inheriting a different default is a common source of behavioral divergence.

**Always cross-check against this table when migrating settings.** If a setting is absent from `foundry.toml`, check whether Forge's default matches Hardhat's. If not, set it explicitly in `hardhat.config.ts`.

| Setting | Forge default | Hardhat 3 default | Action needed |
| --- | --- | --- | --- |
| Source directory | `src` | `./contracts` | **Always set** `paths.sources: "./src"` if project uses `src/` |
| Test directory | `test` | `./test` | Same — no action unless non-standard |
| Optimizer | `true` | Not enabled | **Always set** `optimizer.enabled` explicitly |
| Optimizer runs | `200` | N/A (optimizer off) | **Always set** `optimizer.runs` explicitly |
| EVM version | Auto-detects (latest supported by solc) | Defers to solc default | Set explicitly if project depends on specific EVM features (e.g., transient storage needs `cancun`+) |
| `via_ir` | `false` | `false` | Same — no action unless overridden |
| Fuzz runs | `256` | `256` | Same — no action unless overridden |
| Fuzz max test rejects | `65536` | `65536` | Same |
| Fuzz dictionary weight | `40` | `40` | Same |
| Invariant runs | `256` | `256` | Same |
| Invariant depth | `500` | `500` | Same |
| Invariant fail on revert | `false` | `false` | Same |
| FFI | `false` | `false` | Same |
| Gas limit | ~1 billion (`1 << 30`) | `i64::MAX` | Different but both very large; rarely matters |
| Block gas limit | `30_000_000` | `30_000_000` | Same |

**Critical rows** where Forge and Hardhat defaults differ and require explicit config:

1. **Source directory** — Forge uses `src/`, Hardhat uses `./contracts`
2. **Optimizer** — Forge enables by default, Hardhat does not
3. **Optimizer runs** — Forge defaults to 200, Hardhat has no default (optimizer is off)

## Reference: Known Hardhat 3 limitations

Track these issues — they may be resolved in future versions:

- **Glob overrides:** https://github.com/NomicFoundation/hardhat/issues/4686
- ~~**Gas snapshot tests:**~~ Resolved — use `--snapshot` and `--snapshot-check` CLI flags. See: https://hardhat.org/docs/guides/testing/gas-snapshots
- **Inline test config — contract-level:** Hardhat only supports function-level inline config. Contract-level `forge-config:` directives (on contract definitions) are silently ignored — must use global config as workaround. Foundry added contract-level support in v0.3.0 ([#9430](https://github.com/foundry-rs/foundry/pull/9430)).
- **Inline test config — remaining settings:** Some settings (`isolate`, `evm_version`, etc.) are not yet supported inline even at function level: https://github.com/NomicFoundation/edr/issues/1349
