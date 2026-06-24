# Task Migration (Hardhat V2 → V3)

## Required Reading Checklist

Before migrating any tasks, you **MUST** read and understand the following reference documents. Do not skip this step.

- [ ] Read [Tasks (plugin development tutorial)](https://hardhat.org/docs/plugin-development/tutorial/task.md) — Explains how tasks are defined in HH3 plugins (`HardhatPlugin` object, `setAction` with lazy-loaded modules, task arguments interface, type-safe arguments)
- [ ] Read [Writing Hardhat tasks](https://hardhat.org/docs/guides/writing-tasks.md) — Covers writing user-defined tasks in HH3 config files (`defineConfig`, `setAction` vs `setInlineAction`, returning `Result`, choosing between lazy-loaded and inline actions)

## Key Differences from Hardhat V2

In Hardhat V2, tasks were defined imperatively using the global `task()` function with inline callbacks:

```ts
// HH2 style
task("accounts", "Prints accounts", async (taskArgs, hre) => {
  const accounts = await hre.ethers.getSigners();
  accounts.forEach((account) => console.log(account.address));
});
```

In Hardhat V3, tasks are defined declaratively using a builder pattern and must be registered in the config:

```ts
// HH3 style — in hardhat.config.ts
import { defineConfig, task } from "hardhat/config";

const printAccounts = task("accounts", "Prints accounts")
  .setInlineAction(async (args, hre) => {
    // `ethers` is NOT on the HRE in V3 — get it from a network connection
    const { ethers } = await hre.network.create(); // or hre.network.getOrCreate() to reuse
    const accounts = await ethers.getSigners();
    accounts.forEach((account) => console.log(account.address));
  })
  .build();

export default defineConfig({
  tasks: [printAccounts],
});
```

### Summary of changes

| Aspect | Hardhat V2 | Hardhat V3 |
|--------|-----------|-----------|
| Definition | `task("name", async (args, hre) => { ... })` | `task("name", "desc").setInlineAction(handler).build()` |
| Registration | Automatic (side effect of importing) | Explicit in `defineConfig({ tasks: [...] })` |
| Action handler | Inline callback | `setInlineAction(handler)` for simple tasks; `setAction(() => import("./file.js"))` for complex/lazy-loaded tasks |
| Arguments | `.addParam()`, `.addOptionalParam()` | `.addOption({...})` (named `--opt`), `.addFlag({...})` (boolean), `.addPositionalArgument({...})`, `.addVariadicArgument({...})` |
| Subtasks | `subtask("name", ...)` | Nested tasks: `task(["parent", "child"], ...)` or `emptyTask(["parent"], ...)` for a grouping parent |
| Override | `task("existing-name", ...)` overwrites | `overrideTask("existing-name", ...)` explicitly |
| Return values | Implicitly returned | Optional typed `Result<ValueT, ErrorT>` via `successfulResult()` / `errorResult()` |

## Not All Tasks Need Migration

Many Hardhat V2 custom tasks exist to provide functionality that is now **built into Hardhat V3 by default**. Before migrating a task, evaluate whether it is still needed:

- **`compile` wrappers/overrides** — HH3 has a new declarative compilation pipeline with hooks. Custom compile tasks that added pre/post steps should be replaced with build hooks instead.
- **`accounts` / `node-info` tasks** — Basic account listing and node information are available through HH3's built-in tooling.
- **`clean` tasks** — HH3 handles artifact cleanup natively.
- **`verify` tasks** — Use the official `@nomicfoundation/hardhat-verify` plugin for HH3 instead of a custom task.
- **`size-contracts` tasks** — There is no official Nomic sizer and no built-in HH3 replacement; if sizing is still needed, use the community `@solidstate/hardhat-contract-sizer` (peer `hardhat ^3.8.0`), otherwise drop the task.
- **Tasks that wrap `hre.run("compile")`** — In HH3, compilation is triggered through hooks, not by running tasks programmatically.
- **Gas reporting tasks** — Gas reporting is built into HH3; run `hardhat test --gas-stats` instead of a custom task or a gas-reporter plugin.

**Rule of thumb:** If a V2 task exists solely to work around a V2 limitation or to provide functionality that a V3 plugin now handles, **delete it** rather than migrate it. Only migrate tasks that implement genuinely custom project-specific logic.
