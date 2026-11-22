# Repo baseline scaffold

Type: task
Status: open
Blocked by: 10, 11
Map: [../map.md](../map.md)

## Question

Build the engineering baseline decided in [Engineering standards](11-engineering-standards.md) as a real repo on a feature branch, merged to `main` via PR, so its unverified parts are proven before the spec is final. Scope:

- `@n8n/node-cli` scaffold, then every config from Engineering standards: `tsconfig.json`, `test/tsconfig.json`, `eslint.strict.config.mjs`, `.fallowrc.json` (zones from [Node architecture](10-node-architecture.md)), `vitest.config.mts`, `lefthook.yml`, commitlint, `.npmrc`, `.nvmrc`, `.editorconfig`, Renovate, `ci.yml`, the three `scripts/check-*.mjs`, `AGENTS.md`/`CLAUDE.md`/`CONTRIBUTING.md`.
- `PlunkApi` credential plus a stub `Plunk` node with exactly one dummy resource: just enough code to see every gate fail once and pass once.
- **Not** in scope: real operations, the trigger node, `publish.yml` (Release and verification pipeline), README/SEO.

Record in the answer which of these held, and reopen Engineering standards if one breaks:
- lefthook hooks fire after `allowScripts` approval, and the approval key survives a Renovate bump of lefthook.
- `@typescript-eslint/dot-notation` × `noPropertyAccessFromIndexSignature` don't fight.
- `n8n-node dev` (`tsc --watch`) works on TS 6 + `node16`.
- No `coverage/*.js` lint noise with JSON-only reporters; no stray assets copied into `dist/`.
- fallow: an unzoned new resource folder fails `requireAllFiles`; forbidden-call rule catches `.call(this, …)`.
- `check-dist-imports.mjs` catches a planted `zod` import; `check-pack.mjs` catches a planted extra file; `check-toolchain-sync.mjs` catches a mismatched `typescript-eslint`.
- `lint` (both layers) gates `n8n-node release`'s local and CI paths.
