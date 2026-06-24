# Environment File Setup

Before starting the migration, check if the project contains an example environment file (e.g., `.env.example`, `.env.sample`, `.env.template`).

## Steps

1. Look for an env-template file in the project root. These are the most common names, but the project may use a different one:
   - `.env.example`
   - `.env.sample`
   - `.env.template`
   - other variants such as `.env.dist`, `.env.local.example`, or `env.example` (no leading dot)

2. If one exists **and** there is no `.env` file already present, copy it to `.env`:
   ```bash
   cp <file-from-step-1> .env   # e.g. cp .env.example .env
   ```

3. If a `.env` file already exists, skip this step — do not overwrite it.

This seeds `.env` with the variable names the Hardhat config and deployment scripts expect, read via `configVariable("X")` (resolves from environment variables by default). Fill in any real values the validation steps need.

Hardhat does **not** auto-load `.env`, so do **not** add `import "dotenv/config"` to `hardhat.config.ts` — load `.env` at the shell when a command needs the values:

```bash
set -a; source .env 2>/dev/null; set +a
npx hardhat test
```

For the full secret-handling rules — `configVariable`, the keystore (incl. `npx hardhat keystore set --dev <KEY>` when a test needs a missing value), env-var precedence, and CI — see "Migrate network configuration" in [hardhat-config-migration.md](../phase-2-config/hardhat-config-migration.md).
