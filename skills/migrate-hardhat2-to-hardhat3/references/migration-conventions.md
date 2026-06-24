# Migration Conventions

Shared rules for every phase of the Hardhat V2→V3 migration. Each phase skill links here instead of repeating these rules. They apply whether the migration is run by a **single agent** (the default — read this file, follow each phase top-to-bottom) or, on a large project, by **sub-agents** (see the optional section at the end).

## Core rules

1. **No version-control commands.** Never run `git` (or any VCS) command — `commit`, `add`, `stash`, `checkout`, `reset`, `restore`, etc. VCS can discard the user's work; the user owns it entirely. Present suggested commit messages for the user to apply.
2. **Detect the package manager once.** Check for a lockfile and reuse the result for every install/add/remove command. Mixing package managers corrupts `node_modules`.
   - `pnpm-lock.yaml` → pnpm   ·   `yarn.lock` → yarn   ·   `package-lock.json` → npm
3. **Stay in scope.** Each phase owns a fixed set of files (see the domain table). Do not edit files outside the phase you are running. If a phase needs a change in another domain, make it when you reach that domain's phase, or — in sub-agent mode — hand it off (see below).
4. **Gates are ordered and atomic.** Every step ends with a numbered gate. Run the checks in order; each must pass before the next. If any check fails, fix it and **re-run the gate from the first check** (a fix can invalidate an earlier check). Never skip a check.
5. **Present changes honestly.** Before suggesting a commit, surface any unexpected file changes and their explanation verbatim — do not summarize them away. If every change was expected, just list the changed files.
6. **Keep command output out of context.** The Phase 4/5 fix loops re-run `npx tsc --noEmit`, `npx hardhat test`, and `npx hardhat build` repeatedly (and Phase 6 re-runs whatever `package.json` scripts the user picks); their raw output is large and repetitive, and on a long migration it — not the source files — is what fills the context window. During per-file iteration, filter to the file you're working on (`... 2>&1 | grep {filename}`); once a file passes, don't carry its earlier failing output forward. Gate runs still need a full clean pass, but a pass needs no detail retained. This matters even for a single agent.
7. **When uncertain, consult the official docs — don't guess.** If you're unsure about a Hardhat V3 API, config option, or migration step, consult the official docs index at `https://hardhat.org/llms.txt` (an LLM-readable map of the HH3 docs, including the "Migrate from Hardhat 2" guides) and fetch the relevant linked page before acting. Use it as a fallback for genuine uncertainty, not a routine lookup on every step.

## Domain table

Which files each phase/domain is responsible for. Use it to keep edits scoped and to route a needed change to the right phase.

| Phase | Domain          | Owns                                                                    |
| ----- | --------------- | ----------------------------------------------------------------------- |
| 1     | `env`           | `.env`                                                                  |
| 1     | `tsconfig`      | `tsconfig.json`, `tsconfig.*.json`                                      |
| 1     | `package`       | `package.json`, lockfiles, dependency installs                         |
| 2     | `config`        | `hardhat.config.ts`, shared config helpers, custom tasks               |
| 3     | `solidity-test` | `.sol` test files, test helpers                                         |
| 4     | `esm`           | `.ts`/`.js`/`.mjs`/`.cjs` source files (import/export syntax)           |
| 4     | `typechain`     | TypeChain import verification (reads all files, edits only its imports) |
| 5     | `test`          | `.ts` test files (test suite fix-up)                                    |

## Optional: sub-agent fan-out for large projects

**A single agent is the default** — the simplest, lowest-overhead way to run the migration. If that's you, skip this section and run the phases in order.

**When to fan out.** Reach for sub-agents when a single context would *overflow* before the migration finishes. The benefit is **context survival, not speed** — the strictly-sequential, checkpoint-gated phases cap any real wall-clock gain. Each sub-agent absorbs its own file-reads and debug-loop output (failing-test stack traces, `tsc` dumps) in its own window and returns only a compact summary, keeping the orchestrator lean enough to drive all six phases to completion. Use it when:

- you are on a **smaller-context harness** (~200k-token window) and the project is past trivial — roughly **>30–40 `.ts` test files** (a medium project already overflows ~200k mid-Phase-4); **or**
- you are on a **large-context harness** (~1M-token window) and the project is large — roughly **>150 `.ts` test files**, or when the relevant source + test bytes exceed about half your context window (e.g. ~320 TS files / ~3,500 tests).

Below those thresholds the coordination overhead below is net-negative — stay single-agent. If you can't tell which window you're on, assume the smaller one: overflowing mid-migration costs more than unnecessary fan-out.

**Where fan-out actually helps.** Only **Phase 4 (source files)** and **Phase 5 (tests)** own many independent units to split. Phases 1, 2, 3, and 6 are single-file or interactive (`package.json`/`tsconfig`/`.env`, one config file, one `.sol`-test domain, interactive script validation) and gain nothing from sub-agents — running them under an orchestrator is pure overhead. Cross-phase parallelism is impossible regardless: every phase depends on the previous one's output and ends at a user checkpoint, so the orchestrator still drives the phases strictly in order.

**Fan-out recipe (Phases 4 and 5).** Split the file list into batches of ~10, give one sub-agent per batch, then run a final regression pass that re-runs *every* file to catch regressions a later batch introduced — batch this pass too (each agent reporting only pass/fail) if the full file list would overflow one window.
- **Phase 5 (tests):** batches are independent — split freely.
- **Phase 4 (source):** independent leaf modules can split across agents, but **import-coupled files must stay on one agent** (the recursive dependency-first rule — migrate a dependency before its importer), and the shared `npx tsc --noEmit` / `npx hardhat build` gate is a global join point all batches must pass.

**Agent task framing.** Give each agent: its one-line task, the detected package manager, the reference docs for its domain, the domain table, and the rule that it must not edit files outside its domain. An agent that needs an install command but is not the `package` domain asks the orchestrator instead of running it.

**Cross-domain handoff (correctness, not speed).** This protocol keeps edits scoped and the orchestrator's window clean — it does not make anything faster. When an agent needs a change in a file owned by another domain — whether that domain already ran (e.g. a `package` install) or hasn't yet (a forward request):
1. The agent reports to the orchestrator: which domain, the exact change, and why.
2. The orchestrator spawns a short-lived **fix agent** scoped to that one change (name it `<domain>-fix-N` to avoid collisions), which makes the change, confirms back to the requester, and waits.
3. The requester verifies (runs the relevant build/check) and confirms or asks for a correction.
4. On confirmation the fix agent is terminated — it never runs the full phase for that domain, only the scoped change.

Common handoffs: `config` (add to `paths.sources.solidity`, `solidity.npmFilesToBuild`, `typechain.outDir`, mocha/test settings) and `package` (install a missing dependency).

**Lifecycle, escalation, recovery.** Terminate each agent as soon as its work is confirmed — idle agents are fragile; spawn a fresh one if a domain is needed again. If an agent is stuck, unsure, or needs user input (ambiguous config, missing credentials, conflicting deps), it reports to the orchestrator with what it tried, what's unclear, and the options it sees; the orchestrator relays to the user. Treat an agent as failed on a tool error, empty/truncated/off-topic output, or (for background agents) no completion when its batch is done. Never retry a dead instance — spawn a fresh replacement, passing any partial output and the specific remaining gaps, and **check for side effects (files already written) before replacing** so work isn't duplicated or lost. If the same task fails twice the same way, decompose it or escalate to the user — never silently drop work.
