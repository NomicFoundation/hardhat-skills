# Troubleshooting: Hardhat V2 to V3 Migration

Common errors encountered during migration and how to fix them.

## "Cannot use require() in ES module"

The project is now ESM (`"type": "module"` in package.json), so every `.js` file is parsed as an ES module and `require()`/`module.exports` throw at runtime. Convert to ESM:

- **Do this:** Replace `require()` with `import` and `module.exports` with `export`. This is the supported path and the goal of the migration.
- **Fallback (scripts, tests, and tool configs *only*):** If converting a file is impractical, rename it from `.js` to `.cjs` to keep CommonJS syntax — Hardhat officially supports CommonJS in scripts and tests, and tool configs (`.eslintrc.js`, `.prettierrc.js`, etc.) often have to be `.cjs` because the tool has no ESM config. See [esm-migration.md](phase-4-source-files/esm-migration.md) §1 and §8.
- **Never rename the Hardhat config to `.cjs`.** `hardhat.config.ts`/`.js` must be a *true* ES module (`import` / `export default`) — Hardhat 3 dropped CommonJS config loading, and a `.cjs` config produces a more confusing failure. If the error points at the config file, convert it; do not rename it.

## "ERR_MODULE_NOT_FOUND" for relative imports

ESM requires file extensions in relative imports. Change:

```ts
// Before
import x from "./x";
// After
import x from "./x.js";
```

This applies to `.ts` files too — TypeScript with `"module": "node20"` (or `node16`/`nodenext`) resolves `.js` extensions to `.ts` files at compile time.

## Network URLs reject empty strings (HHE15)

**Symptom:** Config validation fails at load with error code `HHE15: Invalid config`, followed by a per-field line such as `* Config error in config.networks.<name>.url: Expected a URL or a Configuration Variable` (the message is multi-line, not a single em-dash string) on HTTP networks.

**Root cause:** Hardhat V2 accepted empty strings (`""`) for network URLs — the network just wouldn't work if you tried to connect. Hardhat 3 validates all network URLs at config load time, so `process.env.RPC_URL || ""` fails when the env var is unset.

**How to detect:** Look for these patterns in the config:

```ts
url: process.env.SOME_RPC_URL || "";
url: RPC_URL; // where RPC_URL defaults to ""
```

**Fix:** Replace `process.env.X || ""` with `configVariable("X")`, which defers resolution to connection time. See the full V3 config example using `configVariable()` in [config-example.md](phase-2-config/config-example.md).

## "hre.network.provider is not a function" / HRE API changes

V3 uses explicit network connections. In V2, plugins extended the top-level HRE (`hre.ethers`, `hre.network`). In V3, plugin functionality is attached to `NetworkConnection` objects. You get a connection by calling `hre.network.create()` (or `hre.network.getOrCreate()` to reuse a shared connection). Note: `hre.network.connect()` is deprecated and will be removed — don't use it.

**How to detect:**

```bash
grep -rn "hre\.ethers\|hre\.waffle\|hre\.network\." test/ lib/ scripts/ --include="*.ts" --include="*.js"
grep -rn 'from "@nomicfoundation/hardhat-network-helpers"' test/ --include="*.ts" --include="*.js"
```

**Fix:** Get a `NetworkConnection` and access everything through it:

```ts
const connection = await hre.network.create();
```

**Common HRE property migrations:** See the full migration table in [hardhat-config-migration.md](phase-2-config/hardhat-config-migration.md), section "Detailed HRE Property Migrations".

## "Plugin X is not compatible with Hardhat 3"

Check the Plugin Migration Map in [package-json-migration.md](phase-1-project-setup/package-json-migration.md) for the V3 equivalent. If no V3 version exists:

1. Check https://github.com/NomicFoundation/hardhat/issues/7207 for updates
2. Remove the plugin and note the gap for the user
3. Report missing plugins at the issue above

## TypeScript compilation errors after migration

Common causes:

- **Wrong tsconfig settings:** See [tsconfig-migration.md](phase-1-project-setup/tsconfig-migration.md), section "1. Update compilerOptions for ESM" for the required settings
- **Stale type packages / deps:** After dropping a V2 plugin, also remove any now-unused `@types/*` packages or leftover dev-deps it pulled in — stale entries can shadow or conflict with V3 types. (Most Hardhat plugins ship their own types via module augmentation rather than a separate `@types/` package, so this mainly applies to standalone typed deps.)
- **Missing `.js` extensions:** ESM + TypeScript requires `.js` extensions on relative imports (see ERR_MODULE_NOT_FOUND above)

## Tests timeout or hang

The `hre.network.create()` call is async. Ensure all `before` blocks properly `await` the connection.

## Still stuck?

If none of the above resolves the issue and you're unsure about a Hardhat V3 API, config option, or migration step, consult the official docs index at `https://hardhat.org/llms.txt` — an LLM-readable map of the HH3 documentation, including the "Migrate from Hardhat 2" guides — and fetch the relevant linked page rather than guessing.
