# Hardhat Skills

Agent skills for working on [Hardhat 3](https://hardhat.org) projects following the [Agent Skills](https://agentskills.io/) format.

## Installation

```bash
npx skills add nomicfoundation/hardhat-skills
```

This launches an interactive CLI that walks you through installing the Hardhat skills for your agent. For instance, you can install into your current project (as opposed to globally) and specifically for Claude, which writes the skill files into `.claude/skills/` at the repository root.

## Usage

Once installed, each skill is available to your agents. For Claude the skill can be invoked by name:

```
/migrate-hardhat2-to-hardhat3
```

The agent loads the instructions and structure from the installed skill file and uses them to drive the work e.g. for the migration skills, that means kicking off a step-by-step campaign to migrate the repo.

## Available Skills

### hardhat

The core skill for Hardhat 3 project. Load it whenever your agent is writing or modifying contracts, tests, or config. It teaches the agent:

- When to reach for Solidity tests vs. TypeScript tests, and how to run each layer.
- `forge-std` cheatcodes and the Solidity test conventions Hardhat expects.
- The `network.create()` connection API and `networkHelpers` (fixtures, time/block manipulation).
- The compile-then-typecheck workflow that keeps contract changes and tests in sync.

### hardhat-toolbox-viem

A companion to the `hardhat` skill for projects using `@nomicfoundation/hardhat-toolbox-viem`. It adds information about:

- The viem clients exposed on `network.create()` (public, wallet, and test clients).
- Typed contract interaction — `viem.deployContract`, `read`, `write`, and `getContractAt`.
- `viem.assertions` for revert, event, and balance checks.

### hardhat-toolbox-mocha-ethers

A companion to the `hardhat` skill for projects using `@nomicfoundation/hardhat-toolbox-mocha-ethers`. It adds information about:

- The ethers helpers on `network.create()` (provider, signers, utilities).
- Contract interaction via `ethers.deployContract`, calling functions, and `connect`.
- TypeChain-typed contract instances and the `hardhat build` workflow that generates them.
- The chai matchers (`.to.emit`, `.to.be.revertedWith*`, `.to.changeEtherBalance(s)`).

### migrate-hardhat2-to-hardhat3

Migrate a Hardhat 2 project to Hardhat 3. The agent migrate the project by running through sequential phases, pausing at a user checkpoint after each:

1. **Project setup** — `tsconfig.json` for ESM and the `package.json` Hardhat 2 → Hardhat 3 dependency swap.
2. **Config** — rewrite `hardhat.config.ts` to the declarative `defineConfig` (plugins, networks, tasks, solidity, verify).
3. **Solidity tests** — migrate `.t.sol` files to Hardhat 3 conventions.
4. **Source files** — convert `.ts`/`.js` sources CJS→ESM and update Hardhat 2 APIs to Hardhat 3 (plus TypeChain import fixes).
5. **Test fix-up** — get the TypeScript test suite passing under Hardhat 3.
6. **Script validation** — run and fix the `package.json` scripts you select.

### migrate-foundry-to-hardhat

Migrate a Solidity project from Foundry to Hardhat 3. The agent will migrate the project by systematically working through the following steps in order:

1. **Analyze** the Foundry project — `foundry.toml`, remappings, submodules, inline config, and scripts.
2. **Add Hardhat** to the project with the detected package manager.
3. **Create `hardhat.config.ts`**, mapping every `foundry.toml` section across.
4. **Handle absolute imports** and remapping differences.
5. **Compile** the contracts.
6. **Run Solidity tests** and get them passing.
7. **Migration report** — migration gaps and a Foundry cleanup checklist (originals are preserved, not deleted).
