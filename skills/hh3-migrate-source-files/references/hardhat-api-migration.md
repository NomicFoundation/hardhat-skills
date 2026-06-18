# Hardhat v2 → Hardhat v3 Migration: Replacement Rules

Concrete find-and-replace rules for API changes between Hardhat V2 and V3: chai assertion matchers, network helpers, import paths, library linking prefixes, and runtime behavioral differences. Each rule shows the V2 pattern and its exact V3 equivalent with code examples.

Hardhat 3 can have multiple connections at the same time, so when using methods that specifically interact with a connection, keep this in mind. As a rule of thumb, the method needs to get a specific connection to know where to perform the action. This is why, for example, `chai matchers` now need to receive a specific `ethers` instance, as the user could have multiple different parallel `ethers` instances. If in doubt, check the type definition in `Hardhat 3` for the method you are replacing — they are strongly typed so you can deduce what is needed or missing.

---

## 0. Replacing `import { ... } from "hardhat"` — Use a Network Connection

In Hardhat V3, you no longer import `ethers`, `network`, etc. directly from `"hardhat"`. Instead, you obtain them from a **network connection**, which you get from the HRE. Acquire it **at runtime, at the point of use** — inside an async function, a Mocha hook, or an `it(...)` — **never at module top level** (see the warning below):

```ts
import hre from "hardhat";

// inside a function / before(Each) hook / it(...) — NOT at module top level
const { ethers, networkHelpers, provider, networkName, networkConfig } =
  await hre.network.getOrCreate();
```

`artifacts` is **not** part of the connection — it stays on the HRE: `const artifacts = hre.artifacts;`.

> **Acquire the connection at the point of use, never at module top level.** Empirically, a module-top-level `const { ethers } = await hre.network.getOrCreate()` does **not** share one connection: **each module ends up with its own isolated connection and EVM state — even with `getOrCreate()`**. A contract deployed through one module's connection is then invisible to another (`getCode()` → `0x`, calls fail with ethers `BAD_DATA … value="0x"`). Calling `getOrCreate()` at **runtime** — inside functions / `before`/`beforeEach` hooks / `it` blocks — returns the *same* cached connection with shared state every time. **Rule: in every file, obtain the connection lazily at the point of use — not in a module-scope `const`.** *(Root cause appears to be HRE/module instancing under the test loader, not connection-setup timing.)*

### `create()` vs `getOrCreate()` — choose by connection lifecycle

Both return a `NetworkConnection`; the difference is whether a new connection is opened:

- **`hre.network.create()`** — opens a **new** connection on every call. Use when a file or test needs its own isolated network state.
- **`hre.network.getOrCreate()`** — at **runtime**, **reuses** the existing connection with the same network name + chain type, opening one only if none exists yet. This is the common case for production projects with many test/script files: each file calls `getOrCreate()` *at runtime* and they all share one connection and its EVM state. The sharing comes from runtime caching, **not** from where the call lives — a module-top-level call does **not** share (see the warning above).

> **Note:** `hre.network.connect()` is deprecated and slated for removal — use `hre.network.create()` / `hre.network.getOrCreate()`. Don't reintroduce `connect()`.
>
> **Do not** introduce a centralized helper module (e.g. `lib/hardhat.ts`) that calls `getOrCreate()` at module top-level and re-exports `ethers` / `networkHelpers` / `provider` / etc. Each file must obtain its connection directly via `create()` or `getOrCreate()`; sharing is achieved by `getOrCreate()`, not by a re-export module. If such a helper already exists in the project, remove it and convert its importers to call the connection directly.

### Two cases the connection does NOT cover

`await hre.network.create()`/`getOrCreate()` is async, so it can't be used at module top-level or inside a synchronous function. Two common patterns need a different fix:

- **Static `ethers` helpers** — `isAddress`, `parseEther`/`formatEther`, `keccak256`, `toUtf8Bytes`, `ZeroAddress`, `AbiCoder`, etc. are **network-independent**. Do **not** route them through a connection (it would force a needless `await` and break module-top-level `const`s). Import them directly from the `ethers` package: `import { isAddress, parseEther } from "ethers";`.
- **Sync utilities that branch on the network name** — e.g. a `getContracts()` that returns an address book keyed off `network.name`. There is **no synchronous** way to read the *in-process* network in V3: `hre.globalOptions.network` only reflects an explicit `--network <name>` flag, so it is `undefined` for the default in-process run these utilities target (and can't tell you the connection's actual network anyway). Either pass the name/connection in as a parameter, or make the utility `async` and read `const { networkName } = await hre.network.getOrCreate();` — module-top-level callers then use top-level `await` (`const contracts = await getContracts();`, valid in ESM). When you do this, also apply the §5.1 remap **inside** the utility: the in-process network is now `"default"`, so add it to the branch that previously matched `"hardhat"`/`"localhost"` or the utility will throw (`Unknown Network`).

---

## 1. Chai Assertion Changes

Several matchers changed to support multiple network connections:

| V2 Matcher                                       | V3 Matcher                                               |
| ------------------------------------------------ | -------------------------------------------------------- |
| `.changeEtherBalance(account, amount)`           | `.changeEtherBalance(ethers, account, amount)`           |
| `.changeEtherBalances(accounts, amounts)`        | `.changeEtherBalances(ethers, accounts, amounts)`        |
| `.changeTokenBalance(token, account, amount)`    | `.changeTokenBalance(ethers, token, account, amount)`    |
| `.changeTokenBalances(token, accounts, amounts)` | `.changeTokenBalances(ethers, token, accounts, amounts)` |
| `.revertedWithoutReason()`                       | `.revertedWithoutReason(ethers)`                         |
| `.reverted`                                      | `.revert(ethers)`                                        |

### Examples

All chai matchers that previously were property-based (`.reverted`) become function calls requiring the `ethers` instance as an argument.

### 1.1 `.to.be.reverted` → `.to.revert(ethers)`

```ts
// HH2
await expect(lido.initialize(ZeroAddress, eip712helperAddress)).to.be.reverted;

// HH3
await expect(lido.initialize(ZeroAddress, eip712helperAddress)).to.revert(
  ethers,
);
```

### 1.2 `.to.not.be.reverted` → `.to.not.be.revert(ethers)`

The only required change is `reverted` (property) → `revert(ethers)` (method call). `.be` is a Chai no-op language chain — keep it or drop it, it makes no difference (`.to.not.revert(ethers)` is equally valid).

```ts
// HH2
await expect(lido.setMaxExternalRatioBP(TOTAL_BASIS_POINTS)).to.not.be.reverted;

// HH3
await expect(lido.setMaxExternalRatioBP(TOTAL_BASIS_POINTS)).to.not.be.revert(
  ethers,
);
```

### 1.3 `.revertedWithoutReason()` → `.revertedWithoutReason(ethers)`

```ts
// HH2
await expect(
  stakeLimitUtils.updatePrevStakeLimit(2n ** 96n),
).revertedWithoutReason();
await expect(steth.eip712Domain()).to.be.revertedWithoutReason();

// HH3
await expect(
  stakeLimitUtils.updatePrevStakeLimit(2n ** 96n),
).revertedWithoutReason(ethers);
await expect(steth.eip712Domain()).to.be.revertedWithoutReason(ethers);
```

### 1.4 `.revertedWith(...)` is unchanged

`revertedWith` does **not** take an `ethers` argument and is unchanged in V3. `.be` is a Chai no-op language chain, so there is **no** requirement to drop it — `not.to.be.revertedWith(x)` remains valid as-is. (Only `.reverted` → `.revert(ethers)` and the matchers that gained a leading `ethers` argument — §1.1, §1.3, §1.5, §1.6 — are real changes.)

```ts
// HH2 and HH3 — identical, no migration needed
await expect(
  impl.checkContractVersion(petrifiedVersion),
).not.to.be.revertedWith("UNEXPECTED_CONTRACT_VERSION");
```

### 1.5 `.changeEtherBalance(account, amount)` → `.changeEtherBalance(ethers, account, amount)`

`ethers` is inserted as the first argument.

```ts
// HH2
await expect(stakingVault.fund({ value: amount })).to.changeEtherBalance(
  stakingVault,
  amount,
);

// HH3
await expect(stakingVault.fund({ value: amount })).to.changeEtherBalance(
  ethers,
  stakingVault,
  amount,
);
```

### 1.6 `.changeEtherBalances([...], [...])` → `.changeEtherBalances(ethers, [...], [...])`

Same pattern — `ethers` inserted as first argument.

```ts
// HH2
await expect(tx).to.changeEtherBalances([sender, proxy], [-value, value]);

// HH3
await expect(tx).to.changeEtherBalances(
  ethers,
  [sender, proxy],
  [-value, value],
);
```

---

## 2. Network Helpers

> **Note:** In the V3 examples below, `networkHelpers` (and `ethers`) come from the connection (see Section 0) — they are no longer imported. The examples show only the call sites; the connection itself is obtained at runtime, at the point of use (inside functions/hooks/`it`, **not** at module top level), via `hre.network.create()` / `getOrCreate()`.

All individual imports from `@nomicfoundation/hardhat-network-helpers` are replaced by the single `networkHelpers` object from the network connection.

Network helpers are now a plugin and come from the network connection.

### Example

### 2.1 `time` → `networkHelpers.time`

```ts
// HH2
import { time } from "@nomicfoundation/hardhat-network-helpers";
await time.latest();
await time.latestBlock();
await time.increase(duration);

// HH3 — networkHelpers from the connection (Section 0)
await networkHelpers.time.latest();
await networkHelpers.time.latestBlock();
await networkHelpers.time.increase(duration);
```

### 2.2 `latestBlock` (direct import) → `networkHelpers.time.latestBlock()`

```ts
// HH2
import { latestBlock } from "@nomicfoundation/hardhat-network-helpers/dist/src/helpers/time";
const block = await latestBlock();

// HH3 — networkHelpers from the connection (Section 0)
const block = await networkHelpers.time.latestBlock();
```

### 2.3 `mine()` → `networkHelpers.mine()`

```ts
// HH2
import { mine } from "@nomicfoundation/hardhat-network-helpers";
await mine(10);

// HH3 — networkHelpers from the connection (Section 0)
await networkHelpers.mine(10);
```

### 2.4 `mineUpTo()` → `networkHelpers.mineUpTo()`

```ts
// HH2
import { mineUpTo } from "@nomicfoundation/hardhat-network-helpers";
await mineUpTo(blockNumber);

// HH3 — networkHelpers from the connection (Section 0)
await networkHelpers.mineUpTo(blockNumber);
```

### 2.5 `setBalance()` → `networkHelpers.setBalance()`

```ts
// HH2
import { setBalance } from "@nomicfoundation/hardhat-network-helpers";
await setBalance(address, amount);

// HH3 — networkHelpers from the connection (Section 0)
await networkHelpers.setBalance(address, amount);
```

### 2.6 `setStorageAt()` → `networkHelpers.setStorageAt()`

```ts
// HH2
import { setStorageAt } from "@nomicfoundation/hardhat-network-helpers";
await setStorageAt(address, slot, value);

// HH3 — networkHelpers from the connection (Section 0)
await networkHelpers.setStorageAt(address, slot, value);
```

### 2.7 `getStorageAt()` → `networkHelpers.getStorageAt()`

```ts
// HH2
import { getStorageAt } from "@nomicfoundation/hardhat-network-helpers";
await getStorageAt(address, slot);

// HH3 — networkHelpers from the connection (Section 0)
await networkHelpers.getStorageAt(address, slot);
```

### 2.8 `setCode()` → `networkHelpers.setCode()`

```ts
// HH2
import { setCode } from "@nomicfoundation/hardhat-network-helpers";
await setCode(address, bytecode);

// HH3 — networkHelpers from the connection (Section 0)
await networkHelpers.setCode(address, bytecode);
```

### 2.9 `loadFixture()` → `networkHelpers.loadFixture()`

`loadFixture` is one of the most commonly used network helpers. In V2 it was imported from the helpers package; in V3 it comes from the network connection like all other helpers.

```ts
// HH2
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";

async function deployFixture() {
  const [owner] = await ethers.getSigners();
  const token = await ethers.deployContract("Token");
  return { owner, token };
}

it("should deploy", async function () {
  const { owner, token } = await loadFixture(deployFixture);
});

// HH3 — ethers + networkHelpers from the connection (Section 0)
async function deployFixture() {
  const [owner] = await ethers.getSigners();
  const token = await ethers.deployContract("Token");
  return { owner, token };
}

it("should deploy", async function () {
  const { owner, token } = await networkHelpers.loadFixture(deployFixture);
});
```

Also applies to `loadFixture` imported from deep paths:

```ts
// HH2
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers/dist/src/helpers/loadFixture";

// HH3 — networkHelpers from the connection (Section 0)
// use: networkHelpers.loadFixture(...)
```

---

## 3. Import Path Changes

### 3.1 `ethers` / `artifacts` from `"hardhat"` → network connection

```ts
// HH2
import { ethers } from "hardhat";
import { artifacts, ethers } from "hardhat";

// HH3 — ethers from the connection, acquired at point of use (Section 0); artifacts stays on the HRE
const { ethers } = await hre.network.getOrCreate(); // inside a function/hook/it, not module top level
const artifacts = hre.artifacts;
```

### 3.2 `network` object replaced with `networkConfig` / `networkName`

```ts
// HH2
import { network } from "hardhat";
network.config.chainId;
network.name;

// HH3 — from the connection, acquired at point of use (Section 0)
const { networkConfig, networkName } = await hre.network.getOrCreate(); // inside a function/hook/it
networkConfig.chainId;
networkName;
```

### 3.3 `HardhatEthersSigner`: `/signers` → `/types`

```ts
// HH2
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";

// HH3
import type { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/types";
```

### 3.4 `PANIC_CODES`: `hardhat-chai-matchers` → `hardhat-ethers-chai-matchers`

```ts
// HH2
import { PANIC_CODES } from "@nomicfoundation/hardhat-chai-matchers/panic";

// HH3
import { PANIC_CODES } from "@nomicfoundation/hardhat-ethers-chai-matchers/panic";
```

### 3.5 `anyValue`: `hardhat-chai-matchers` → `hardhat-ethers-chai-matchers`

```ts
// HH2
import { anyValue } from "@nomicfoundation/hardhat-chai-matchers/withArgs";

// HH3
import { anyValue } from "@nomicfoundation/hardhat-ethers-chai-matchers/withArgs";
```

### 3.6 Don't infer Hardhat types — import the exported named type

When you need to **type** a Hardhat object (a connection, `ethers`, `networkHelpers`, a signer, a fixture, etc.), do **not** reconstruct the type by inferring it from a variable or a function's return type. Hardhat's plugins export these as named types — find the one that matches and import it directly.

```ts
// ❌ Don't infer the type from the runtime value
type HardhatConnection = Awaited<ReturnType<typeof hre.network.getOrCreate>>;
let ethers: HardhatConnection["ethers"];

// ✅ Import the exported named type — the same module the file already pulls
//    `HardhatEthersSigner` from (see §3.3)
import { type HardhatEthers } from "@nomicfoundation/hardhat-ethers/types";
let ethers: HardhatEthers;
```

These types are strongly typed and stable; the `Awaited<ReturnType<typeof ...>>["x"]` indirection is brittle, harder to read, and unnecessary. If you can't find the type by name, check the type definitions in the plugin you're using — they are exported from a `/types` subpath. Common ones:

| Need a type for…                                   | Import                                                              | From                                             |
| -------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------ |
| the `ethers` object (`typeof ethers` + helpers)    | `HardhatEthers`                                                     | `@nomicfoundation/hardhat-ethers/types`          |
| a signer (`.address`, etc.)                        | `HardhatEthersSigner`                                               | `@nomicfoundation/hardhat-ethers/types`          |
| a provider                                         | `HardhatEthersProvider`                                             | `@nomicfoundation/hardhat-ethers/types`          |
| `getContractFactory`/`getContractAt`/`getSigners`… | `HardhatEthersHelpers`                                              | `@nomicfoundation/hardhat-ethers/types`          |
| library linking / deploy options                   | `Libraries`, `FactoryOptions`, `DeployContractOptions`              | `@nomicfoundation/hardhat-ethers/types`          |
| the `networkHelpers` object                        | `NetworkHelpers`                                                    | `@nomicfoundation/hardhat-network-helpers/types` |
| time / fixtures / snapshots                         | `Time`, `Duration`, `Fixture`, `SnapshotRestorer`, `Snapshot`       | `@nomicfoundation/hardhat-network-helpers/types` |
| helper arg types                                   | `NumberLike`, `BlockTag`                                            | `@nomicfoundation/hardhat-network-helpers/types` |
| the whole connection (`network.create()` result)  | `NetworkConnection<ChainTypeT>`                                     | `hardhat/types/network`                          |
| `hre.network`                                      | `NetworkManager`                                                    | `hardhat/types/network`                          |
| chain-type generics / connection params            | `ChainType`, `DefaultChainType`, `GenericChainType`, `NetworkConnectionParams`, `CachedNetworkConnectionParams` | `hardhat/types/network` |

> **Rule of thumb:** never reconstruct a connection-member type via `Awaited<ReturnType<typeof hre.network.getOrCreate>>["x"]`. Import the named type directly — `HardhatEthers` for `.ethers`, `NetworkHelpers` for `.networkHelpers`, `NetworkConnection` for the whole connection.

---

## 4. Library Linking Path Prefix

In `getContractFactory` library linking, fully qualified contract paths gain a `project/` prefix.

### 4.1 Fully qualified path in `libraries` option

```ts
// HH2
await ethers.getContractFactory("NodeOperatorsRegistry__Harness", {
  libraries: {
    ["contracts/common/lib/MinFirstAllocationStrategy.sol:MinFirstAllocationStrategy"]:
      await allocLib.getAddress(),
  },
});

// HH3
await ethers.getContractFactory("NodeOperatorsRegistry__Harness", {
  libraries: {
    ["project/contracts/common/lib/MinFirstAllocationStrategy.sol:MinFirstAllocationStrategy"]:
      await allocLib.getAddress(),
  },
});
```

### 4.2 Hashed placeholder names in library address maps

The truncated placeholder also changes because the path changed.

```ts
// HH2
const allocLibAddr: NodeOperatorsRegistryLibraryAddresses = {
  ["__contracts/common/lib/MinFirstAllocat__"]: await allocLib.getAddress(),
};

// HH3
const allocLibAddr: NodeOperatorsRegistryLibraryAddresses = {
  ["__project/contracts/common/lib/MinFirs__"]: await allocLib.getAddress(),
};
```

---

## 5. Runtime Behavioral Changes

### 5.1 Default network name: `"hardhat"` → `"default"`

The in-process test network is now named `"default"` instead of `"hardhat"`.

```ts
// HH2
if (networkName === "hardhat") {
  networkName = "mainnet-fork";
}

// HH3
if (networkName === "default") {
  networkName = "mainnet-fork";
}
```
