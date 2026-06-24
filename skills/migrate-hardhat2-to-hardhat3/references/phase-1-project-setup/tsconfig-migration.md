# tsconfig.json Migration (V2 to V3)

Skip this step entirely if the project has no `tsconfig.json`.

## 1. Update `compilerOptions` for ESM

Hardhat V3 requires ESM-compatible TypeScript settings. Update these fields:

| Field                  | Required value | Notes                                                                                    |
| ---------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| `module`               | `"nodenext"`   | Valid on the project's existing TypeScript (5.x or 6.x) with no version bump. Use `"node20"` only if TypeScript `>= 5.9` (matches the HH3 template's pinned `~6.0.3`); `node20` errors `TS6046` on older TS. |
| `target`               | `"es2023"`     | Matches the HH3 template (safe for Node 22)                                              |
| `lib`                  | `["es2023"]`   | Recommended for Node 22                                                                  |
| `verbatimModuleSyntax` | `true`         | Enforces type-only import/export syntax — prevents runtime import of types in ESM output |
| `types`                | includes `"node"` and `"mocha"` (Mocha projects) | **Required for Mocha + TS.** `describe`/`it` resolve only when `"mocha"` is listed here — they are not auto-included under the template's pinned TypeScript (`~6.0.3`). Also install `@types/mocha` (see [package-json-migration.md](package-json-migration.md) §5) — listing `"mocha"` without it errors `TS2688`. Node-test-runner projects use `["node"]`. Preserve any project-specific entries already present. |

`moduleResolution` no longer needs to be set explicitly — it is implied by the node-family `module` value (`nodenext`/`node20`/`node16`). (`node20` matches the current template but needs TypeScript `>= 5.9` — use `nodenext` on older TypeScript. Older HH3 docs used `node16`; it still works but no longer matches the official template.)

## 2. Remove unnecessary fields

- Keep `"resolveJsonModule": true` if the project imports any `.json` files (ABIs, artifacts, address books) — it is **not** implied by the `node`-family `module` values (`node20`/`node16`/`nodenext`); it defaults to `false`, so removing it breaks JSON imports. Leaving it in is always safe.
- Remove `"esModuleInterop": true` only if no CJS interop is needed; otherwise keep it (safe to leave in)
- Remove any `@types/` packages for deprecated V2 plugins (e.g. `@types/hardhat-gas-reporter`) — these will cause type conflicts with V3

## 3. Reference: V3 template tsconfig

This is the full tsconfig shipped by the official Hardhat V3 mocha-ethers template:

```json
{
  "compilerOptions": {
    "lib": ["es2023"],
    "module": "node20",
    "target": "es2023",
    "skipLibCheck": true,
    "verbatimModuleSyntax": true,
    "types": ["node", "mocha"],
    "outDir": "dist"
  }
}
```

Note the current template does **not** set `strict`, `esModuleInterop`, or `moduleResolution` (the last is implied by `module: "node20"`). The `types` array differs by toolbox — the node-test-runner template uses `["node"]`.

Do not blindly overwrite the project's tsconfig with this template — merge the required changes (`module`, `target`/`lib`, `verbatimModuleSyntax`, `types`) into the existing config, preserving project-specific settings like `paths`, `include`/`exclude`, `strict` preferences, `resolveJsonModule`, etc.

## 4. Note on verification

Do not run `tsc --noEmit` at this stage — with the node-family `"module"` just set (`nodenext`, or `node20` on TS `>= 5.9`), TypeScript will enforce `.js` extensions on relative imports, and the rest of the codebase hasn't been updated yet. Type checking will be meaningful only after all source files (tests, scripts, config) have been migrated to ESM imports.
