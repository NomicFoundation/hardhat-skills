# hardhat.config Migration (V2 to V3)

This reference covers rewriting the Hardhat config file from V2's imperative style to V3's declarative ESM format. The official migration guide is the primary source of truth — read [Migrating from Hardhat 2](https://hardhat.org/docs/migrate-from-hardhat2.md) and the [configuration reference](https://hardhat.org/docs/reference/configuration.md) for full details.

`defineConfig` is strongly typed — when unsure where a V2 property moved or how it's structured in V3, inspect the type definition. It shows exactly which properties exist, where they're nested, and what values they accept.

## 1. Rename the old config

Keep the V2 config for reference:

```bash
mv hardhat.config.js hardhat.config.old.js
# or for TypeScript:
mv hardhat.config.ts hardhat.config.old.ts
```

## Quick Reference: V2 vs V3 Equivalents

Consult this table throughout the migration to understand how V2 concepts map to V3:

| Concept               | Hardhat V2                                            | Hardhat V3                                           |
| --------------------- | ----------------------------------------------------- | ---------------------------------------------------- |
| Config format         | `module.exports = {}`                                 | `export default defineConfig({})`                    |
| Module system         | CommonJS                                              | ESM (`.js` = ESM)                                    |
| Plugin registration   | `require("plugin")` side-effect                       | `plugins: [plugin]` array in config                  |
| Network access        | `hre.network.provider` global                         | `await hre.network.create()` explicit                |
| Ethers access         | `hre.ethers` global                                   | `const { ethers } = await hre.network.create()`      |
| Environment variables | `process.env.X`                                       | `configVariable("X")`                                |
| Task definition       | `task("name", async (args, hre) => {})`               | `task("name").setInlineAction(...).build()`          |
| Tasks in config       | Registered by side effect                             | `tasks: [task1, task2]` array                        |
| Subtasks              | `subtask("name", ...)`                                | Nested tasks: `task(["parent","child"], ...)` / `emptyTask(["parent"], ...)` |
| `extendConfig`        | `extendConfig(fn)`                                    | Hooks system                                         |
| `extendEnvironment`   | `extendEnvironment(fn)`                               | Hooks system                                         |
| Network helpers       | `require("@nomicfoundation/hardhat-network-helpers")` | `const { networkHelpers } = await network.create()`  |
| Compilation command   | `npx hardhat compile`                                 | `npx hardhat build`                                  |
| Clean command         | `npx hardhat clean`                                   | `npx hardhat clean`                                  |
| Default network type  | Implicit                                              | `"edr-simulated"` with `chainType`                   |

### Detailed HRE Property Migrations

When migrating code that accesses plugin functionality through the HRE, use `await hre.network.create()` (or `await hre.network.getOrCreate()` to reuse a shared connection) to get a `NetworkConnection` object, then access properties through it. (`hre.network.connect()` is deprecated and slated for removal.)

| V2 access                                               | V3 access                                         | Notes    |
| ------------------------------------------------------- | ------------------------------------------------- | -------- |
| `hre.ethers`                                            | `connection.ethers`                               |          |
| `hre.ethers.provider`                                   | `connection.ethers.provider`                      |          |
| `hre.ethers.getSigners()`                               | `connection.ethers.getSigners()`                  |          |
| `hre.network.provider.send(method, params)`             | `connection.provider.request({ method, params })` | EIP-1193 |
| `import { mine } from "hardhat-network-helpers"`        | `connection.networkHelpers.mine()`                |          |
| `import { time } from "hardhat-network-helpers"`        | `connection.networkHelpers.time`                  |          |
| `import { loadFixture } from "hardhat-network-helpers"` | `connection.networkHelpers.loadFixture()`         |          |

This table is the canonical find-and-replace reference for HRE property migrations. The Quick Reference table above covers conceptual changes; this table covers the specific API calls needed when rewriting code.

## 2. Create a minimal V3 config

Create a new `hardhat.config.ts` with just enough to verify Hardhat 3 loads:

```ts
import { defineConfig } from "hardhat/config";

export default defineConfig({});
```

Verify it works:

```bash
npx hardhat --help
```

## 3. Migrate solidity settings

Copy the `solidity` entry from the old config. The format is backwards-compatible.

### `paths.sources.solidity`

Controls which directories Hardhat scans for Solidity source files. By default only `contracts` is included. Add additional directories (e.g., `test`) when test contracts need to be compiled — this is required for TypeChain to generate types for test contracts:

```ts
paths: {
  sources: {
    solidity: ["contracts"],
  },
},
```

### `solidity.npmFilesToBuild`

Lists Solidity files from npm packages that Hardhat should compile. Required when the project imports contracts from dependencies (e.g., `@aragon`, `@openzeppelin`) that are not otherwise pulled in by the local source tree:

```ts
solidity: {
  npmFilesToBuild: [],
},
```

The full config at this point:

```ts
import { defineConfig } from "hardhat/config";

export default defineConfig({
  solidity: {
    // copy from old config as-is
    npmFilesToBuild: [],
  },
  paths: {
    sources: {
      solidity: ["contracts"],
    },
  },
});
```

Verify compilation:

```bash
npx hardhat build
```

## 4. Add plugins declaratively

In V2, plugins were registered via `require()` side effects. In V3, plugins are imported as values and listed in the `plugins` array:

**V2 (old):**

```ts
require("@nomicfoundation/hardhat-toolbox");
```

**V3 (new):**

```ts
import hardhatToolboxMochaEthers from "@nomicfoundation/hardhat-toolbox-mocha-ethers";
import { defineConfig } from "hardhat/config";

export default defineConfig({
  plugins: [hardhatToolboxMochaEthers],
  solidity: {
    /* ... */
  },
});
```

For Viem projects, use `@nomicfoundation/hardhat-toolbox-viem` instead.

## 5. Migrate network configuration

V3 networks use explicit types. Keep `process.env` for now — this lets you validate the network config works before switching to `configVariable()` in a separate step outside this reference:

**V2 (old):**

```ts
networks: {
  sepolia: {
    url: process.env.SEPOLIA_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
  },
}
```

**V3 (first pass — keep process.env for validation):**

```ts
import { defineConfig } from "hardhat/config";

export default defineConfig({
  networks: {
    hardhatMainnet: {
      type: "edr-simulated",
      chainType: "l1",
    },
    sepolia: {
      type: "http",
      chainType: "l1",
      url: process.env.SEPOLIA_RPC_URL!,
      accounts: [process.env.SEPOLIA_PRIVATE_KEY!],
    },
  },
});
```

> **Important:** If any network URL can be empty or unset at config load time, you **must** use `configVariable()` instead of `process.env` to avoid HHE15 validation errors. See the full V3 config example in `config-example.md` for the recommended pattern with `configVariable()`.

Key changes:

- Every network must have an explicit `type` (`"edr-simulated"` for local, `"http"` for remote)
- `chainType` is optional — defaults to `"generic"`. Set it explicitly when needed (e.g., `"l1"` for Ethereum mainnet, `"op"` for Optimism)
- `hre.network` no longer represents a single active network; you create connections explicitly via `hre.network.create()`

### Default network: `allowUnlimitedContractSize`

V3's default EDR network sets `allowUnlimitedContractSize: false`. If the project has test harness contracts that exceed the EIP-170 size limit (e.g., `*__Harness`, `*__Mock`), deployments will fail with `Transaction reverted: trying to deploy a contract whose code is too large`. Add this to the default network config:

```ts
networks: {
  default: {
    type: "edr-simulated",
    allowUnlimitedContractSize: true,
  },
}
```

## 6. Migrate custom tasks

**V2 (old):**

```ts
task("accounts", "Prints the accounts", async (taskArgs, hre) => {
  const accounts = await hre.ethers.getSigners();
  for (const account of accounts) {
    console.log(account.address);
  }
});
```

**V3 (new):**

```ts
import { defineConfig, task } from "hardhat/config";

const printAccounts = task("accounts", "Print the accounts")
  .setInlineAction(async (taskArguments, hre) => {
    const { provider } = await hre.network.create();
    console.log(await provider.request({ method: "eth_accounts" }));
  })
  .build();

export default defineConfig({
  tasks: [printAccounts],
});
```

Key changes:

- Tasks use builder pattern with `.build()` at the end
- Tasks are added to the `tasks` array in the config
- Use `setInlineAction` for simple tasks, `setAction` with a separate file for complex tasks (lazy loading)

## 7. Config section migration examples

Not every project has all of these — only migrate the sections that exist in the V2 config.

### Etherscan / verify config

V2's top-level `etherscan` key is split into `verify.etherscan` + `chainDescriptors` in V3:

**V2 (old):**

```ts
etherscan: {
  customChains: [
    { network: "hoodi", chainId: 560048, urls: { apiURL: "https://api-hoodi.etherscan.io/api", browserURL: "https://hoodi.etherscan.io/" } },
    { network: "sepolia", chainId: 11155111, urls: { apiURL: "https://api-sepolia.etherscan.io/api", browserURL: "https://sepolia.etherscan.io/" } },
  ],
  apiKey: process.env.ETHERSCAN_API_KEY || "",
},
```

**V3 (new):**

```ts
verify: {
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY || "",
  },
},
chainDescriptors: {
  560048: {
    name: "hoodi",
    blockExplorers: {
      etherscan: {
        name: "hoodi",
        apiUrl: "https://api-hoodi.etherscan.io/api",
        url: "https://hoodi.etherscan.io/",
      },
    },
  },
  11155111: {
    name: "sepolia",
    blockExplorers: {
      etherscan: {
        name: "sepolia",
        apiUrl: "https://api-sepolia.etherscan.io/api",
        url: "https://sepolia.etherscan.io/",
      },
    },
  },
},
```

Key changes:

- `etherscan.apiKey` → `verify.etherscan.apiKey`
- `etherscan.customChains` array → `chainDescriptors` object, keyed by `chainId` (numeric)
- `urls: { apiURL, browserURL }` → `blockExplorers: { etherscan: { name, apiUrl, url } }`
- Note casing: `apiURL` → `apiUrl`, `browserURL` → `url`

### Mocha / test config

V2's top-level `mocha` key moves under `test.mocha`:

**V2 (old):**

```ts
mocha: {
  fullTrace: true,
  rootHooks: mochaRootHooks,
  timeout: 20 * 60 * 1000,
},
```

**V3 (new):**

```ts
test: {
  mocha: {
    fullTrace: true,
    rootHooks: mochaRootHooks,
    timeout: 20 * 60 * 1000,
  },
},
```

### Warning: Mocha root hooks with HRE dependencies

When migrating `test.mocha.rootHooks` or `test.mocha.require` to V3:

1. Check if the hooks file imports anything from HRE (ethers, networkHelpers, hre, artifacts, network, etc.)
2. If it does, you **cannot** use a static import in `hardhat.config.ts` — it will hang because HRE isn't initialized yet at config load time
3. Instead, use dynamic `import()` inside a `rootHooks.beforeAll` wrapper to defer loading to test runtime
4. The dynamic `import()` also executes side-effect imports (e.g., custom Chai matcher registrations) in the hooks file

**Example — HRE-dependent hooks file:**

```ts
// hardhat.config.ts
export default defineConfig({
  test: {
    mocha: {
      rootHooks: {
        async beforeAll() {
          const hooks = await import("./test/hooks/index.js");
          const hook = hooks.mochaRootHooks?.beforeAll;
          if (typeof hook === "function") {
            await (hook as () => Promise<void>)();
          }
        },
        beforeEach(done: () => void) {
          done();
        },
      },
    },
  },
});
```

If the hooks file has **no** HRE dependencies, a static import is fine.

If the project has Solidity invariant/fuzz tests, they're configured under `test.solidity` (new in V3 — no V2 equivalent). Do **not** add a block here — use the single `test.solidity` block defined in **Reduced fuzz/invariant runs (MANDATORY)** below, which sets the run counts to use during migration.

### TypeChain

TypeChain is now integrated into the Ethers toolbox. Configuration is available via the `typechain` field in the config:

```typescript
export default defineConfig({
  typechain: {
    outDir: "./types",
    alwaysGenerateOverloads: false,
    dontOverrideCompile: false,
  },
});
```

### Reduced fuzz/invariant runs (MANDATORY)

**Always add this to the config**, regardless of whether the project currently has Solidity tests. It prevents slow fuzz/invariant runs from blocking migration validation. Production values should be restored after migration is stable.

```ts
test: {
  solidity: {
    fuzz: {
      runs: 4,
    },
    invariant: {
      runs: 4,
      depth: 4,
      failOnRevert: true, // only affects projects with invariant tests
    },
  },
},
```

If the config already has a `test` section (e.g., `test.mocha`), merge this into it — do not create a duplicate `test` key.

### Removed V2 config sections

These top-level sections should be deleted entirely if present:

| Section                     | Reason                                                                       |
| --------------------------- | ---------------------------------------------------------------------------- |
| `gasReporter: { ... }`      | Built-in — use `hardhat test --gas-stats` instead (no config section needed) |
| `tracer: { ... }`           | Plugin dropped (no V3 version)                                               |
| `watcher: { ... }`          | Plugin dropped (no V3 version)                                               |
| `defaultNetwork: "hardhat"` | Implicit — the `"default"` network replaces this                             |

> **`warnings: { ... }`:** Not removed. `hardhat-ignore-warnings` has a Hardhat 3 release (`0.3.0`, peer `hardhat ^3.1.0`). Keep the `warnings` section and register the plugin in `plugins: [...]` rather than deleting it.

## 8. Verify the full config (HARD STOP)

This verification is **blocking** — do not return success or delete the old config until all checks pass.

### Handling import failures during verification

When verification steps fail because of broken imports (in the config itself or in files it pulls in), assess the scope before fixing:

- **Small fix (<=5 files, mechanical changes):** Fix inline. Examples: updating `require()` → `import`, fixing a renamed export, adding `.js` extensions to relative imports, adjusting a path that moved. Include every touched file in your changed-file report.

- **Large fix (>5 files, or changes that require understanding business logic):** Do **not** attempt the fix. Instead, stop and report to the user: what broke (exact errors), how many files are affected, what the fix would involve, and whether it should be a follow-up step or a separate task. A focused migration commit is more reviewable than one that quietly refactors half the codebase.

The threshold is about complexity, not just file count — 3 files needing careful logic changes count as "large," while 6 files needing an identical one-line replacement count as "small." When in doubt, flag it rather than expanding scope.

### 8a. TypeScript compilation

Run `tsc` unfiltered so you can confirm it actually type-checked the config (a filtered command hides the "tsc never ran" case and would false-pass):

```bash
npx tsc --noEmit
```

Judge the output:

- If tsc prints a **usage/help banner** (e.g. `Common Commands`, `tsc --init`), it found no `tsconfig.json` and checked nothing — treat this as a **FAILURE**, not a pass. Ensure a `tsconfig.json` exists that includes `hardhat.config.ts`, then re-run.
- If any reported error line references `hardhat.config.ts`, the config has type errors — fix them and re-run.
- If any error references `tsconfig.json` itself or is a compiler-option / config-parse diagnostic (e.g. `TS6046` "Argument for '--module' option must be…", `TS5070`), the `tsconfig.json` is broken and tsc fell back to default module resolution — so import/module-resolution checks (notably the mandatory `.js`-extension enforcement, `TS2835`) silently do **not** run and the result can't be trusted — treat as a **FAILURE**, not "out of scope". (Basic in-file type errors may still be reported; that does not mean the check is sound.) The most common cause is `"module": "node20"` on TypeScript `< 5.9` (node20 needs TS `>= 5.9`); switch to `"module": "nodenext"` (or bump TypeScript), then re-run.
- Errors in *source* files (not the config or `tsconfig.json` itself) are out of scope for this step (handled in other migration phases). The config passes 8a only once tsc has actually run (no help banner, no `tsconfig.json`/compiler-option diagnostic) and reports **no** `hardhat.config.ts` errors.

### 8b. Hardhat loads

`hardhat --version` does **not** load `hardhat.config.ts`, so it cannot catch config errors. Use a command that actually loads the config (same check as step 2):

```bash
npx hardhat --help
```

Must exit without error and list the available tasks. If it fails (import resolution, config parse, plugin not found — e.g. `ERR_MODULE_NOT_FOUND` or "An unexpected error occurred"), the config is broken — fix and re-run.

### 8c. Clean up

Only after all three checks above pass, delete the old config file:

```bash
rm hardhat.config.old.js  # or .old.ts
```
