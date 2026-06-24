# ESM Source File Conversion (V2 to V3)

This reference covers converting all non-config source files from CJS to ESM. Phase 1 already set `"type": "module"` in package.json and updated tsconfig.json — this phase handles the downstream impact on every `.ts` source file.

The hardhat config is out of scope (the `config` domain handles it). See [troubleshooting.md](../troubleshooting.md) for common ESM error diagnosis.

## Table of Contents

1. [Scope boundaries](#1-scope-boundaries)
2. [`.js` extensions on relative imports](#2-js-extensions-on-relative-imports)
3. [`require()` to `import`](#3-require-to-import)
4. [`module.exports` to `export`](#4-moduleexports-to-export)
5. [Type import/export rule](#5-type-importexport-rule)
6. [`__dirname` / `__filename`](#6-__dirname--__filename)
7. [Dynamic `require()` to dynamic `import()`](#7-dynamic-require-to-dynamic-import)
8. [CJS config files (`.cjs` rename)](#8-cjs-config-files-cjs-rename)
9. [Verification (HARD STOP)](#9-verification-hard-stop)
10. [Escalation rules](#10-escalation-rules)

---

## 1. Scope boundaries

This phase owns `.ts` source files **and** any `.js` source files that still use CommonJS syntax (the `esm` domain in the [domain table](../migration-conventions.md)). Phase 1 set `"type": "module"`, so a `.js` file using `require()`/`module.exports` **no longer runs** — Node throws `require is not defined in ES module scope` the moment it executes.

**Handle each source file by what it already is:**

- **`.ts` files** — convert in place (the rest of this doc).
- **CommonJS `.js` source files** (using `require()`/`module.exports` — e.g. `scripts/deploy.js`, `helpers/*.js`) — **either** convert to ESM syntax (preferred; then they follow the same rules as a `.ts` file) **or** rename to `.cjs` to keep them running as CommonJS (the same escape §8 uses for tool configs). Do not leave them as broken `.js`.
- **Files already on pure ESM syntax** (or `.mjs`/`.cjs`) — leave untouched.

> **The Phase-4 gate will NOT catch a broken CJS `.js` source file.** `npx tsc --noEmit` skips `.js` (no `allowJs` in the HH3 tsconfig) and `npx hardhat build` only compiles Solidity, so a stranded `require()`-based `.js` script passes the gate and fails only when it is actually run (e.g. in Phase 6). Find these during Step 1 collection, not at runtime.

**In scope** — all `.ts` source files, plus any CommonJS `.js` source files, under: `test/`, `scripts/`, `lib/`, `deploy/`, `helpers/`, `tasks/`, `src/`, `utils/`.

**Out of scope — do not touch:** `hardhat.config.ts/.js` (config domain), `package.json` (package domain), `tsconfig.json` (tsconfig domain), `.sol` files (solidity-test domain), `node_modules/`, and source files already using pure ESM syntax.

---

## 2. `.js` extensions on relative imports

Under ESM with `"module": "node20"` (or `node16`/`nodenext`), TypeScript requires explicit `.js` extensions on all relative imports — Node's ESM loader resolves compiled `.js` output, not `.ts` source.

```ts
// Before
import { foo } from "./utils";
import { bar } from "../helpers/deploy";
export { baz } from "./baz";

// After — add .js to imports AND re-exports
import { foo } from "./utils.js";
import { bar } from "../helpers/deploy.js";
export { baz } from "./baz.js";
```

**Directory/barrel imports:** CJS auto-resolves `./dir` to `./dir/index.js` — ESM does not. If the target is a directory with an `index.ts`/`index.js`, make it explicit:

```ts
// Before
import { ether } from "./lib";
import { MyContract } from "../contracts";

// After
import { ether } from "./lib/index.js";
import { MyContract } from "../contracts/index.js";
```

Prefer granular imports (`"./lib/units.js"`) over barrel imports (`"./lib/index.js"`) when possible.

---

## 3. `require()` to `import`

Convert top-level `require()` calls to static `import` statements. Combine with the `.js` extension rule for relative paths.

```ts
// Before
const { expect } = require("chai");
const {
  time,
  loadFixture,
} = require("@nomicfoundation/hardhat-network-helpers");
const utils = require("./utils");

// After
import { expect } from "chai";
import { time, loadFixture } from "@nomicfoundation/hardhat-network-helpers";
import utils from "./utils.js";
```

**JSON imports** require an import attribute (Node.js v22+):

```ts
// Before
const config = require("./config.json");

// After
import config from "./config.json" with { type: "json" };
```

If a `require()` is inside a function or conditional, it cannot be a static import — see section 7 (dynamic import).

---

## 4. `module.exports` to `export`

Choose named or default exports based on how the module is consumed:

```ts
// Before — consumers destructure: const { foo, bar } = require(...)
module.exports = { foo, bar };
// After
export { foo, bar };

// Before — consumers use the whole object: const thing = require(...)
module.exports = MyClass;
// After
export default MyClass;

// Mixed: object with properties that consumers destructure
module.exports = { deploy: deployFunction, tags: ["all"] };
// After
export const deploy = deployFunction;
export const tags = ["all"];
```

---

## 5. Type import/export rule

With `verbatimModuleSyntax` enabled in tsconfig, TypeScript requires type-only syntax for symbols that have no runtime value (`type`, `interface`, type aliases). `tsc` will error if this is wrong — fix every occurrence.

```ts
import type { Foo } from "bar.js";           // all type imports
import { type Foo, realValue } from "bar.js"; // mixed type + value
export type { Foo };                          // type re-export
export type { Foo } from "bar.js";            // type re-export from module
```

---

## 6. `__dirname` / `__filename`

ESM does not provide these CJS globals. Since this migration targets Node.js v22.13+, use `import.meta.dirname` and `import.meta.filename` as direct inline replacements — no imports or helpers needed.

```ts
// Before
path.join(__dirname, "data.json");
console.log(__filename);

// After
path.join(import.meta.dirname, "data.json");
console.log(import.meta.filename);
```

---

## 7. Dynamic `require()` to dynamic `import()`

For `require()` calls inside functions, conditionals, or loops that cannot be hoisted to static imports:

```ts
// Before
function loadPlugin(name) {
  const p = require(`./plugins/${name}`);
  return p;
}

// After
async function loadPlugin(name) {
  const p = await import(`./plugins/${name}.js`);
  return p.default ?? p;
}
```

Key differences: `import()` is async (containing function must be async or handle the Promise), returns a module namespace (use `.default` for default exports), and still needs `.js` on relative paths.

**Escape hatch** — if a synchronous context truly cannot be made async:

```ts
import { createRequire } from "module";
const require = createRequire(import.meta.url);
const pkg = require("./legacy-sync-dep");
```

Flag every `createRequire` usage to the user — these are escape hatches, not the preferred pattern.

---

## 8. CJS config files (`.cjs` rename)

With `"type": "module"`, `.js` files are parsed as ESM. Tool config files using `module.exports` must be renamed to `.cjs`: `.eslintrc.js`, `.prettierrc.js`, `jest.config.js`, `commitlint.config.js`, `lint-staged.config.js`, etc.

**Do NOT rename if** the file already uses ESM syntax (`export default`) or the tool uses a non-JS config (`.json`, `.yaml`).

After renaming, update any references to the old filename in `package.json` or CI scripts.

---

## 9. Verification (HARD STOP)

**Blocking** — do not return success until all checks pass.

### 9a. No extensionless relative imports

Verify all relative `from` statements end in `.js`, `.json`, or `.cjs`.

---

## 10. Escalation rules

**Fix inline:** Adding `.js` extensions, converting straightforward `require()`→`import`, converting `module.exports`→`export`, renaming CJS configs to `.cjs`, replacing `__dirname`/`__filename`.

**Escalate to the user (or, in sub-agent mode, hand off to the owning domain):**

- `require()` in deeply nested conditional logic that can't trivially go async
- Files mixing CJS and ESM in complex ways
- Packages lacking ESM-compatible entry points
- More than 3 files needing `createRequire` (suggests systemic issue)
- Changes requiring business logic understanding to determine export shape
- `from "hardhat"` imports pulling plugin extensions where no shared connection module exists

**Never attempt:** Changes to `hardhat.config.ts`, `package.json`, `tsconfig.json`, `.sol` files, or installing/removing packages.
