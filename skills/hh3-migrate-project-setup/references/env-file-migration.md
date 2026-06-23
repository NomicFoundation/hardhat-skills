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

This seeds `.env` with the variable names the Hardhat config and deployment scripts expect. The V3 config reads these via `configVariable("X")`, which resolves from environment variables by default — so `.env` is all the config needs to keep working through the migration, and **the migration does not use the encrypted keystore** (an optional post-migration choice). Example files usually ship with empty placeholders, so fill in any real values the validation steps need. Note: Hardhat does not auto-load `.env` — the values are only read if the config imports a loader such as `dotenv/config`.
