# Hardhat Skills

Agent skills for working on [Hardhat 3](https://hardhat.org) projects following the [Agent Skills](https://agentskills.io/) format.

## Installation

```bash
npx skills add nomicfoundation/hardhat-skills
```

This launches an interactive CLI that walks you through where to install. For instance, you can install into your current project (as opposed to globally) and specifically for Claude, which writes the skill files into `.claude/skills/` at the repository.

## Usage

Once installed, each skill is available to your agents, for Claude it can invoked by name:

```
/migrate-hardhat2-to-hardhat3
```

Claude loads the instructions and structure from the installed skill file and uses them to drive the work e.g. for the migration skills, that means kicking off a step-by-step campaign to migrate the repo.

## Available Skills

### hardhat

Core skill for any Hardhat 3 project. Covers testing (both Solidity & TS tests), cheatcodes, the `network.create()` connection API, `networkHelpers`, and more.

### hardhat-toolbox-viem

Companion to the `hardhat` skill for projects using `@nomicfoundation/hardhat-toolbox-viem`. Covers the viem clients exposed on `network.create()`, contract interaction, and `viem.assertions`.

### hardhat-toolbox-mocha-ethers

Companion to the `hardhat` skill for projects using `@nomicfoundation/hardhat-toolbox-mocha-ethers`. Covers ethers helpers on `network.create()`, contract interaction, TypeChain-typed contract instances, and chai matchers.

### migrate-hardhat2-to-hardhat3

Migrate a Hardhat 2 project to Hardhat 3, covering core changes (ESM, declarative config, hooks system, network connections), the plugin ecosystem (hardhat-verify, @openzeppelin/hardhat-upgrades, and others) and test migration.

### migrate-foundry-to-hardhat

Migrate a Foundry project to Hardhat 3, covering the build system and dependencies, Solidity test migration, coverage and gas reporting.
