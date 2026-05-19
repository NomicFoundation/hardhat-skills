# Hardhat Skills

Agent skills for working on [Hardhat 3](https://hardhat.org) projects following the [Agent Skills](https://agentskills.io/) format.

## Installation

```bash
npx skills add nomicfoundation/hardhat-skills
```

## Available Skills

### hardhat

Core skill for any Hardhat 3 project. Covers testing, cheatcodes, the `network.create()` connection API, `networkHelpers`, and more.

### hardhat-toolbox-viem

Companion to the `hardhat` skill for projects using `@nomicfoundation/hardhat-toolbox-viem`. Covers the viem clients exposed on `network.create()`, contract interaction, and `viem.assertions`.

### hardhat-toolbox-mocha-ethers

Companion to the `hardhat` skill for projects using `@nomicfoundation/hardhat-toolbox-mocha-ethers`. Covers ethers helpers on `network.create()`, contract interaction, TypeChain-typed contract instances, and chai matchers.
