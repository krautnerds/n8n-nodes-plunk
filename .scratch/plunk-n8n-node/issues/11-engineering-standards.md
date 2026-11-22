# Engineering standards

Type: grilling
Status: resolved
Blocked by: 03, 05
Map: [../map.md](../map.md)

## Question

What exactly are the engineering standards: tsconfig flags, ESLint rule set, test layers and coverage gates, fallow rules, pre-commit hooks, CI gates, and the AGENTS.md/CLAUDE.md contributor rules?

## Answer

Grilled with the user on 2026-11-22, building on [Strict tooling compatibility with n8n node tooling](05-tooling-compatibility.md) (full doc: branch `research/tooling-compatibility`, `claudedocs/research/05-tooling-compatibility.md`) and [n8n community node and verification rules 2026](03-n8n-verification-rules.md). Versions re-checked on npm on 2026-11-19: `n8n@2.48.6` requires Node `>=24`; `@n8n/node-cli@0.57.4` still pins `eslint 9.29.0` + `typescript-eslint 8.70.0`; `n8n-workflow` stable 2.48.3 and rc 3.3.0 both peer `zod: 3.25.76`.

**Guiding rule**: the n8n-owned files (`eslint.config.mjs`, `.prettierrc.js`) stay byte-identical to the template. Everything stricter goes in our own layer next to them. Hooks are a convenience; CI is the hard gate.

### Toolchain
- npm (template default, trusted publishing), Node 24 (`.nvmrc`, `engines.node: ">=24"`), no matrix.
- Everything pinned exactly (`save-exact=true` in `.npmrc`): TypeScript 6.0.x (6.0.3), ESLint 9.39.x, `typescript-eslint` exactly node-cli's pin, `n8n-workflow` devDependency exactly n8n stable, `zod@3.25.76`. TS 7 / ESLint 10 only after node-cli and typescript-eslint move.

### `tsconfig.json` (the build config)
- `module`/`moduleResolution: node16` (CJS output), `target`/`lib: ES2024`, `skipLibCheck: true`.
- Flags: `strict` (incl. `useUnknownInCatchVariables`; template's `false` removed), `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noPropertyAccessFromIndexSignature`, `noImplicitOverride`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, `noUnusedLocals`, `noUnusedParameters`, `allowUnreachableCode: false`, `allowUnusedLabels: false`, `isolatedModules`, `erasableSyntaxOnly` (no `enum`/`namespace`; use `as const` unions), `noUncheckedSideEffectImports`, `forceConsistentCasingInFileNames`.
- Not used: `verbatimModuleSyntax` (breaks CJS), `isolatedDeclarations` (friction, no consumer of our `.d.ts`).
- `test/tsconfig.json` extends root with `noEmit`, includes `test/`, `nodes/`, `credentials/`.

### ESLint
- `eslint.config.mjs` untouched (`n8n.strict: true`). Second config `eslint.strict.config.mjs` = n8n config + `strictTypeChecked` + `stylisticTypeChecked`, run with `--max-warnings 0` (n8n's warn-level rules become gates too).
- Extra rules: `consistent-type-imports`, `switch-exhaustiveness-check` (no `default` escape on unions), `explicit-module-boundary-types`, `import-x/no-cycle`, `import-x/no-default-export` (config files exempt), `eqeqeq`, `curly: all`, `no-param-reassign`, moderate `naming-convention` (types PascalCase, no `I` prefix on our own types). Complexity limits are left to fallow, not ESLint.
- Scripts: `lint` = `n8n-node lint && eslint -c eslint.strict.config.mjs --max-warnings 0 .`, so `n8n-node release` enforces both; `lint:n8n` and `lint:strict` stay callable separately.
- **`eslint-disable` policy**: forbidden in `nodes/**` and `credentials/**` (the scanner ignores it there, so a disable would only hide a verification failure); in `test/**` only with a description and never blanket (`@eslint-community/eslint-comments`: `no-use` on product paths, `require-description`, `no-unlimited-disable`); `reportUnusedDisableDirectives: "error"` everywhere.

### zod and the devDependency boundary
No zod at runtime. Three layers keep any devDependency out of shipped code:
1. Strict ESLint: `nodes/**` and `credentials/**` may import only `n8n-workflow` and relative paths (tighter than n8n's allowlist; additions such as `node:crypto` need an explicit decision).
2. fallow: the `tests` zone may import node code, never the reverse.
3. `scripts/check-dist-imports.mjs` after build: every `require()` in `dist/**/*.js` must be `n8n-workflow` or relative. A script, not a test, because `test/` may not use `fs` under `n8n-node lint`.

### Tests (vitest 5 + `vitest-mock-extended` + nock 15, all under top-level `test/`)
- `test/unit/`: pure functions (mappers, error translation, validators).
- `test/node/`: `execute`/trigger methods with `mockDeep<IExecuteFunctions>()`, nock with `disableNetConnect()`, assertions on `pairedItem`, `continueOnFail`, `NodeApiError`.
- `test/contract/`: own `zod/v4` schemas written from reading Plunk's source (never copy AGPL files); each validates the nock fixtures **and** the request bodies the node builds. The Plunk commit they were last checked against lives in `test/contract/plunk-commit.ts`.
- `test/e2e/`: smoke tests against the local Docker Plunk ([Local Plunk test environment](06-local-plunk-test-environment.md)); separate vitest project, not in the default run or PR CI.
- No workflow-level harness (`NodeTestHarness` is unpublished).
- Every operation needs at least one node test (happy path + error path) and a contract test.
- Coverage: v8 over `nodes/**` + `credentials/**`, reporters `text`, `json`, `json-summary` (no HTML). Thresholds: 100 % for `transport/` and `shared/`; global 95 % lines/statements/functions, 90 % branches; no `autoUpdate`. Mutation testing is not in the baseline (see map fog).

### fallow
- `entry`: `nodes/*/*.node.ts`, `credentials/*.credentials.ts`, `test/**/*.test.ts`; `ignoreDependencies: ["release-it"]`.
- All dead-code rules `error`: unused files/exports/types/dependencies/devDependencies, circular dependencies, boundary violations.
- Boundary principles (zone names come from [Node architecture](10-node-architecture.md)): only `transport/` performs HTTP (forbidden calls `*.httpRequest`, `*.httpRequestWithAuthentication`, `*.httpRequestWithAuthentication.call`, `fetch` everywhere else); resources never import each other; `shared/` depends on nothing; `credentials/` is isolated; node code never imports `test/`.
- **No `autoDiscover`**: one explicit zone per resource plus `coverage.requireAllFiles: true` (root config files via `allowUnmatched`), so a new resource folder without a zone fails fallow by itself.
- Health stricter than defaults: `maxCyclomatic 10`, `maxCognitive 12`, `maxCrap 15`, `maxUnitSize 60`. Duplication: default `mild`.
- Always a full run, `fallow --fail-on-issues` (no `audit` mode), after coverage so CRAP uses real data.

### Hooks (lefthook; never `prepare`/`postinstall` in our `package.json`)
- pre-commit (parallel, staged files): `prettier --write` with `stage_fixed: true`; strict ESLint `--max-warnings 0` on `*.ts`, no auto-fix.
- commit-msg: Conventional Commits via commitlint (`@commitlint/cli` + `@commitlint/config-conventional`).
- pre-push: `typecheck`, `test:coverage`, `fallow`.
- Install via `"allowScripts": { "lefthook@<version>": true }` in `package.json`; fallback `npx lefthook install` in CONTRIBUTING.

### Commits, PRs, merging
- Conventional Commits. Squash merge only, so the PR title becomes the commit on `main`; CI checks the PR title with the same commitlint config.
- `main` branch protection: CI required, no direct pushes, linear history. Changelog/release tooling belongs to [Release and verification pipeline](13-release-verification-pipeline.md).

### CI (`ci.yml`, on PRs and pushes to `main`; one job, one step per gate, fail-fast)
1. `npm ci`
2. `scripts/check-toolchain-sync.mjs`: `eslint`/`typescript-eslint` match node-cli's pins exactly, `zod` matches `n8n-workflow`'s peer.
3. `format:check` (incl. md/json/yml)
4. `typecheck` (build + test projects)
5. `lint:n8n`
6. `lint:strict`
7. PR title check (PRs only)
8. `test:coverage` with thresholds
9. `fallow --fail-on-issues`
10. `build`
11. `scripts/check-dist-imports.mjs`
12. `scripts/check-pack.mjs`: `npm pack --dry-run --json` against an allowlist (`dist/**`, `package.json`, `README.md`, `LICENSE`)

`npm run check` runs the same chain locally. Not in `ci.yml`: e2e (manual), the post-publish `scan-community-package` run (Release and verification pipeline), Plunk drift detection (map fog "API drift strategy").

### Dependency updates
Renovate: `rangeStrategy: pin`, weekly, `minimumReleaseAge: 3 days`, no automerge. Groups: "n8n toolchain" (`@n8n/node-cli`, `typescript-eslint`, `eslint`, `typescript`), "n8n runtime types" (`n8n-workflow`, `zod`), "test" (vitest, `@vitest/*`, nock, `vitest-mock-extended`). The toolchain sync check catches any grouping gap.

### Formatting
Template `.prettierrc.js` unchanged (n8n style); matching `.editorconfig`.

### Contributor docs
- `AGENTS.md` is the single source of rules (short, imperative: commands, where things live, hard rules). `CLAUDE.md` contains only `@AGENTS.md`. `CONTRIBUTING.md` is for humans (setup with nvm, `npm ci`, lefthook; branch → PR → squash) and points to AGENTS.md for rules. Write AGENTS.md with the `writing-for-agents` skill.
- Hard rules in AGENTS.md: never edit `eslint.config.mjs`/`.prettierrc.js`; no `dependencies`, no extra `peerDependencies`, no lifecycle scripts; no `eslint-disable` in `nodes/`/`credentials/`; node code imports only `n8n-workflow` + relative; HTTP only in `transport/`, resources never import each other, a new resource gets its own fallow zone; tests only in `test/`, no Node globals there except `crypto`; every operation has a node test (happy + error path) and a contract test; contract schemas are our own, never copied from Plunk (AGPL), update the Plunk commit when re-checking; run `npm run check` before a PR; feature branch, Conventional Commits, English.

### Execution exception
The map's Notes now allow one execution ticket before the spec is done: [Repo baseline scaffold](15-repo-baseline-scaffold.md) builds exactly this baseline and verifies the items ticket 05 left unverified.

No ADR: the choices are reversible config, and this ticket records the why.
