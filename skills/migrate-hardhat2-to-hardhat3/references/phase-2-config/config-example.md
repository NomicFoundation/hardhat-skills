# Full V3 Config Example

This example shows a realistic V3 config with all the key patterns in one place — plugin imports, declarative plugin registration, typed networks with `configVariable`, solidity profiles, and test path configuration:

> **`configVariable` for secrets, keystore registered (empty), no `dotenv/config`.** This individual-plugin config lists `@nomicfoundation/hardhat-keystore` in `plugins[]` explicitly (the toolbox would auto-register it). For the rationale — env-var precedence, lazy resolution, `.env` loading, and the keystore workflow — see "Migrate network configuration" in [hardhat-config-migration.md](hardhat-config-migration.md).

```ts
import { configVariable, defineConfig } from "hardhat/config";
import HardhatNodeTestRunner from "@nomicfoundation/hardhat-node-test-runner";
import HardhatMochaTestRunner from "@nomicfoundation/hardhat-mocha";
import HardhatViem from "@nomicfoundation/hardhat-viem";
import HardhatViemAssertions from "@nomicfoundation/hardhat-viem-assertions";
import hardhatNetworkHelpersPlugin from "@nomicfoundation/hardhat-network-helpers";
import hardhatEthersPlugin from "@nomicfoundation/hardhat-ethers";
import hardhatChaiMatchersPlugin from "@nomicfoundation/hardhat-ethers-chai-matchers";
import hardhatTypechain from "@nomicfoundation/hardhat-typechain";
import hardhatIgnitionViem from "@nomicfoundation/hardhat-ignition-viem";
import hardhatVerify from "@nomicfoundation/hardhat-verify";
import hardhatKeystore from "@nomicfoundation/hardhat-keystore";
import hardhatLedger from "@nomicfoundation/hardhat-ledger";

export default defineConfig({
  networks: {
    op: {
      type: "http",
      chainType: "op",
      url: "https://mainnet.optimism.io/",
      accounts: [configVariable("OP_SENDER")],
    },
    edrOp: {
      type: "edr-simulated",
      chainType: "op",
      chainId: 10,
      forking: {
        url: "https://mainnet.optimism.io",
      },
      ledgerAccounts: [
        // Set your ledger address here
        // "0x070Da0697e6B82F0ab3f5D0FD9210EAdF2Ba1516",
      ],
    },
    opSepolia: {
      type: "http",
      chainType: "op",
      url: "https://sepolia.optimism.io",
      accounts: [configVariable("OP_SEPOLIA_SENDER")],
    },
    edrOpSepolia: {
      type: "edr-simulated",
      chainType: "op",
      forking: {
        url: "https://sepolia.optimism.io",
      },
    },
  },
  tasks: [],
  plugins: [
    hardhatEthersPlugin,
    HardhatMochaTestRunner,
    hardhatNetworkHelpersPlugin,
    HardhatNodeTestRunner,
    hardhatVerify,
    HardhatViem,
    HardhatViemAssertions,
    hardhatChaiMatchersPlugin,
    hardhatTypechain,
    hardhatIgnitionViem,
    hardhatKeystore,
    hardhatLedger,
  ],
  paths: {
    tests: {
      mocha: "test/mocha",
      nodejs: "test/node",
      solidity: "test/contracts",
    },
  },
  solidity: {
    profiles: {
      default: {
        compilers: [
          {
            version: "0.8.22",
          },
          {
            version: "0.7.1",
          },
          {
            // Required for @uniswap/core
            version: "0.8.26",
          },
          {
            version: "0.8.33",
          },
        ],
        overrides: {
          "foo/bar.sol": {
            version: "0.8.1",
          },
        },
      },
    },
    npmFilesToBuild: [
      "@openzeppelin/contracts/token/ERC20/ERC20.sol",
      "forge-std/Test.sol",
    ],
  },
  typechain: {
    tsNocheck: false,
  },
  test: {
    mocha: {
      color: true,
    },
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
});
```

## Key patterns

- **Plugin imports**: each plugin is imported as a default import and listed in `plugins[]`
- **Tasks**: this template defines none (`tasks: []`) — see `task-migration.md` for the builder pattern and how to register tasks
- **Networks**: explicit `type` (`"http"` or `"edr-simulated"`), explicit `chainType`, and `configVariable()` for secrets
- **Solidity profiles**: replace the flat V2 compiler config. `default` and `production` are the only built-in profiles — most tasks use `default`, Hardhat Ignition uses `production`. Custom profiles can be added but apply only when selected with `--build-profile <name>`; there is no automatic `test` or `coverage` profile
- **`npmFilesToBuild`**: tells Hardhat to compile Solidity files from `node_modules`
- **TypeChain**: configured via the `typechain` field. Available options include `outDir` (output directory for generated types), `tsNocheck` (add `@ts-nocheck` to generated files), `alwaysGenerateOverloads` (generate overloads for all functions), and `dontOverrideCompile` (don't override the compile task). Because this config uses individual plugins rather than the toolbox, TypeChain is registered explicitly via `@nomicfoundation/hardhat-typechain` in `plugins[]`. (If you use `@nomicfoundation/hardhat-toolbox-mocha-ethers` instead, it's bundled — no separate registration needed.)
