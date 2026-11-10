# Research 05: Strict tooling vs n8n node tooling

Ticket: [`.scratch/plunk-n8n-node/issues/05-tooling-compatibility.md`](../../.scratch/plunk-n8n-node/issues/05-tooling-compatibility.md)
Date: 2026-11-10

**Method.** I unpacked the published tarballs of `@n8n/node-cli`, `@n8n/eslint-plugin-community-nodes`, `@n8n/scan-community-package`, `fallow` and `lefthook` and read their shipped source. Then I scaffolded a project from the CLI's own `declarative/github-issues` template and added a programmatic probe node, a Plunk-shaped `nodes/Plunk/{transport,resources,shared}` layout, a vitest test and a fallow config. I ran the real commands against it (`n8n-node build`, `n8n-node lint`, `tsc`, `eslint`, `vitest`, `fallow`). A finding marked **observed** comes from such a run. One marked **source** comes from reading code or docs. One marked **unverified** is an inference.

## TL;DR

1. **Leave `eslint.config.mjs` untouched.** With `"n8n": { "strict": true }`, `n8n-node lint` compares it (whitespace-normalised) to the template and exits 1 on any change. `n8n-node cloud-support` calls the package "eligible for n8n Cloud verification" only when strict mode is on and the default config is unchanged. Put the stricter rules in a **second config**, `eslint.strict.config.mjs`, and run it with `eslint -c`.
2. **Pin `typescript-eslint` to exactly the version node-cli pins (8.70.0 today).** With any other version, ESLint fails: `Cannot redefine plugin "@typescript-eslint"`.
3. **Use TypeScript 6.0.x with `module`/`moduleResolution: node16`.**
   - The template's `moduleResolution: node` is deprecated in TS 6 and removed in TS 7.
   - TS 7.0.2 builds fine, but `typescript-eslint@8.70.0` requires `typescript <6.1.0` (ERESOLVE), so type-aware linting rules TS 7 out.
4. **Almost every strict tsconfig flag works.** The exceptions:
   - `verbatimModuleSyntax` **breaks**, because n8n loads CommonJS.
   - `isolatedDeclarations` brings friction for no gain.
   - `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess` and `noPropertyAccessFromIndexSignature` bring small, local friction with `n8n-workflow` types.
5. **Tests go in a top-level `test/`, outside `nodes/` and `credentials/`.**
   - The verification scanner lints `{nodes,credentials}/**` with `allowInlineConfig: false`, and it lints every `.js` file in the published tarball.
   - `n8n-node lint` lints `test/` too, so tests must also avoid `process`, `setTimeout`, `globalThis` and Node built-ins other than `crypto`.
6. **Test stack: vitest 5 + `vitest-mock-extended` + nock 15** (it intercepts global `fetch`). This works (observed). n8n's own `NodeTestHarness` (workflow-level tests) is **not published**.
7. **Contracts: zod as a devDependency is fine, but it must be `zod@3.25.76`.**
   - `n8n-workflow@2.47.0` has the peer dependency `zod: 3.25.76`, so declaring `zod@4` next to an explicit `n8n-workflow` devDependency gives ERESOLVE.
   - `zod@3.25.76` ships the v4 API at `zod/v4` (observed).
   - `@plunk/types` is `private` and not on npm.
8. **fallow works, but needs `entry` configured.** It doesn't understand `package.json#n8n.nodes`. Boundaries and forbidden-call rules caught the seeded violations. Watch out for two things: the `autoDiscover` semantics, and the `.call(...)` callee pattern.
9. **Use lefthook, not husky.** Husky's standard `"prepare": "husky"` is an **error** under `@n8n/community-nodes/no-forbidden-lifecycle-scripts`.

## Versions checked (npm registry, 2026-11-10)

| Package | Version / dist-tags | Notes |
|---|---|---|
| `@n8n/node-cli` | latest 0.56.1 (stable 0.55.6, beta 0.57.0) | Deps (exact pins): `eslint 9.29.0`, `typescript-eslint 8.70.0`, `@n8n/eslint-plugin-community-nodes 0.39.0`; also `eslint-plugin-import-x ^4.15.2`, `eslint-plugin-n8n-nodes-base ^2.0.0`, `prettier 3.6.2`. Its own devDeps use `typescript: npm:@typescript/typescript6@6.0.2` |
| `n8n-workflow` | latest **2.16.0** (stale), stable 2.46.5, beta 2.48.0; node-cli resolves **2.47.0** | `peerDependencies: { zod: "3.25.76" }` |
| `n8n` | latest/stable 2.47.3 | |
| `@n8n/eslint-plugin-community-nodes` | latest 0.40.0, stable 0.37.1, rc 1.2.0 | node-cli pins 0.39.0 |
| `@n8n/scan-community-package` | 0.43.0 | |
| `typescript` | latest 7.0.2 (native), 6.0.3 | template pins 5.9.2; `n8n-nodes-starter` pins 5.9.3 |
| `typescript-eslint` | latest 8.72.2 | peer `typescript >=4.8.4 <6.1.0`, `eslint ^8.57 \|\| ^9 \|\| ^10` |
| `eslint` | latest 10.14.0 | template 9.32.0, starter 9.39.4 |
| `vitest` / `@vitest/coverage-v8` | 5.1.0 | n8n monorepo uses vitest |
| `jest` | 30.5.2 | used by e.g. `@apify/n8n-nodes-apify` |
| `nock` / `msw` | 15.0.6 / 3.0.4 | |
| `zod` | latest 4.6.7; 3.25.76 | |
| `fallow` | 3.42.0 | `engines.node >=22`; native binary via optionalDependencies |
| `lefthook` / `husky` / `lint-staged` | 2.2.2 / 9.1.7 / 17.6.0 | |
| `vitest-mock-extended` | 6.0.0 | |

The probe ran on Node 24.22.1 and npm 11.19.1.

## How the n8n toolchain actually behaves (source)

- **`n8n-node build`** (`dist/commands/build.js`) runs `rimraf dist`, then **bare `tsc`**, so it uses `tsconfig.json`. Then it copies `**/*.{png,svg}` and `**/__schema__/**/*.json` (ignoring only `dist` and `node_modules`) into `dist/`. So:
  - `tsconfig.json` *is* the build config.
  - Anything that compiles under its `include` ends up in `dist/`, and `files: ["dist"]` publishes it.
  - Stray PNG/SVG files anywhere, such as an HTML coverage report or test fixtures, get copied into `dist/` (source; whether the coverage PNGs get published is unverified).
- **`n8n-node lint`** (`dist/commands/lint.js`) runs `eslint .`, adding only `--fix`. In strict mode it first runs `verifyEslintConfig`: it reads `eslint.config.mjs`, normalises whitespace, compares the result with the template, and calls `process.exit(1)` on any difference. **Observed:** `Strict mode violation: eslint.config.mjs has been modified`.
- **Default flat config** (`@n8n/node-cli/eslint`, `dist/configs/eslint.js`):
  - Ignores: `globalIgnores(['dist'])`.
  - For `**/*.ts`: `@eslint/js` recommended, `typescript-eslint` **recommended (not type-checked)**, `@n8n/community-nodes` recommended, `import-x` flat/recommended (with the TypeScript resolver), plus `no-console: error` and `prefer-spread: off`.
  - Per-path rule sets from `eslint-plugin-n8n-nodes-base` for `package.json` and `*.node.json`, `credentials/**` and `nodes/**`.
  - Consequence (observed): `coverage/*.js` gets linted, because only `dist` is ignored.
- **`n8n-node release`** (`dist/commands/release.js`):
  - In CI (`GITHUB_ACTIONS`) it runs `<pm> run lint`, `<pm> run build`, then `npm publish` with `NPM_CONFIG_PROVENANCE=true`.
  - Locally it runs `release-it`, with `before:init` set to `<pm> run lint && <pm> run build`.
  - Either way, the **`lint` script** is what gates a release, so it can be widened (see Open questions).
- **Verification scanner** (`@n8n/scan-community-package/dist/scanner.js`):
  1. Requires npm provenance.
  2. Downloads the attested commit from GitHub.
  3. Lints `SOURCE_FILE_PATTERNS = ['package.json', '{nodes,credentials}/**/*.{js,ts,json}']` and the tarball's `**/*.js` and `package.json`. It uses its own config, which equals the community and n8n-nodes-base rule sets plus `no-console: error`, and it sets **`allowInlineConfig: false`**.
  - Consequences:
    - Our extra config files (`eslint.strict.config.mjs`, `.fallowrc.json`, `lefthook.yml`, `test/`) are invisible to the scanner.
    - `eslint-disable` comments under `nodes/` and `credentials/` don't count there.
    - Compiled tests in `dist/` would be scanned.
- **Community rules that matter for tooling** (`@n8n/eslint-plugin-community-nodes`, `docs/rules/*.md`, `src/rules/*.ts`):
  - `no-restricted-imports`:
    - Allowlist: `n8n-workflow`, `ai-node-sdk`, `@n8n/ai-node-sdk`, `lodash`, `moment`, `p-limit`, `luxon`, `zod`, `crypto`, `node:crypto`, plus relative imports.
    - **Modules listed in `devDependencies` are exempt**; 0.39.0 already has this (`readPackageJsonDevDependencies`).
    - Node built-ins such as `node:http` stay restricted, including in tests.
  - `no-restricted-globals`: `process`, `globalThis`, `global`, `setTimeout`/`setInterval`/`setImmediate` and their `clear*`, `__dirname`, `__filename`. `fetch` is **not** on the list.
  - `no-runtime-dependencies`: `dependencies` must be empty.
  - `valid-peer-dependencies`: only `n8n-workflow: "*"` (and `ai-node-sdk`).
  - `no-forbidden-lifecycle-scripts`: `prepare`, `preinstall`, `install`, `postinstall`, `prepublish`, `preprepare`, `postprepare`.
  - `no-dead-files`: warns on `test.ts`/`test.js`/`*.bak` under `nodes/` and `credentials/` (`*.test.ts` is fine).
- **Verification guidelines** (https://docs.n8n.io/integrations/creating-nodes/build/reference/verification-guidelines/):
  - No external dependencies.
  - The linter must pass (`npx @n8n/scan-community-package`).
  - MIT licence, public GitHub repository.
  - Published from GitHub Actions with provenance since 2026-05-01.
  - No environment-variable or filesystem access.
  - **Tests are not mentioned.**

## Compatibility matrix

Legend: ✅ works · ⚠️ friction · ❌ breaks.

### TypeScript (template `tsconfig.json` + one flag each; TS 5.9.2 unless noted; skipLibCheck true; observed)

| Flag / setting | Result | Evidence |
|---|---|---|
| `strict` (template default) | ✅ | 0 errors |
| `useUnknownInCatchVariables: true` (template sets **false**) | ✅ | 0 errors in probe; forces `error as Error`/narrowing in catch blocks |
| `noUncheckedIndexedAccess` | ⚠️ | 3 errors: `items[0].json`, `res.emails[0]`, template `utils.ts` regex group as index. Real but local fixes |
| `exactOptionalPropertyTypes` | ⚠️ | 4 errors. `INodeListSearchResult.paginationToken` (template listSearch code ×3) and `IHttpRequestOptions.qs` reject explicit `undefined`, so optional fields need conditional spreads (`...(qs && { qs })`). Template `body: IDataObject \| undefined` passed because `body`'s type includes `undefined` |
| `noImplicitOverride` | ✅ | 0 errors; nodes `implements INodeType`, no subclassing |
| `noPropertyAccessFromIndexSignature` | ⚠️ | 3 errors: `credentials.baseUrl`, `additionalFields.subject`, `item.json.previous`. `IDataObject`/`ICredentialDataDecryptedObject` are index signatures, so bracket access (`creds['baseUrl']`) is required. Arguably good: it marks untyped data |
| `noFallthroughCasesInSwitch`, `noUnusedParameters`, `noImplicitReturns`, `allowUnreachableCode:false`, `allowUnusedLabels:false`, `isolatedModules`, `erasableSyntaxOnly` | ✅ | 0 errors each |
| `verbatimModuleSyntax` | ❌ | 41 errors (TS1287/TS1295). With CJS output it forbids ESM `import`/`export` syntax. n8n `require()`s `dist/**/*.node.js`, so the output must stay CJS. Use `isolatedModules` + `@typescript-eslint/consistent-type-imports` instead |
| `isolatedDeclarations` | ⚠️ | 5 errors on idiomatic code (shorthand objects in `methods`, inferred arrays, missing return types). No consumer uses our `.d.ts`, so it isn't worth it |
| `skipLibCheck: false` | ❌ | 62 errors inside `node_modules` (`@codemirror/*` DOM types pulled in by `n8n-workflow`, `@n8n/errors`, `n8n-workflow/dist/esm/*.d.ts`). Keep `true` |
| Template `moduleResolution: "node"` on TS 6.0.3 | ⚠️ | TS5107 deprecated; needs `ignoreDeprecations: "6.0"`, which Apify's repo uses (`apify/n8n-nodes-apify/tsconfig.json`), and so does `@n8n/typescript-config` 1.12.0 |
| Template `moduleResolution: "node"` on TS 7.0.2 | ❌ | TS5108 "has been removed" |
| `module`/`moduleResolution: "node16"` on 5.9.2, 6.0.3, 7.0.2 | ✅ | `tsc` emits identical CJS (`require("n8n-workflow")`); `require()` of the built node works; `n8n-node build` succeeds |
| Full recommended flag set (below) on TS 6.0.3 + `n8n-node build` + `n8n-node lint` | ✅ | Probe node: build OK, 0 errors |
| TS 7.0.2 with `typescript-eslint@8.70.0` | ❌ | npm ERESOLVE: peer `typescript@">=4.8.4 <6.1.0"` |

For context, n8n's own base config (`@n8n/typescript-config` 1.12.0, `tsconfig.common.json`) is `strict` + `useUnknownInCatchVariables` + `noUnusedParameters` + `ignoreDeprecations: "6.0"`. It has no `noUncheckedIndexedAccess` and no `exactOptionalPropertyTypes`, so n8n core itself doesn't go further than that. Don't extend it: it targets the monorepo, includes the `dom` lib, and is under n8n's licence (`SEE LICENSE IN LICENSE.md`).

### ESLint (observed unless noted)

| Item | Result | Evidence |
|---|---|---|
| Editing `eslint.config.mjs` while `n8n.strict: true` | ❌ | `n8n-node lint` exits 1 with "Strict mode violation" |
| Setting `n8n.strict: false` to allow edits | ❌ (for us) | `cloud-support` then reports "NOT eligible for n8n Cloud verification" (source) |
| Separate `eslint.strict.config.mjs` importing `config` from `@n8n/node-cli/eslint` + `tseslint.configs.strictTypeChecked` + `stylisticTypeChecked`, run with `eslint -c` | ✅ | Runs in about 2 s on the probe. `n8n-node lint` stays green because it only reads `eslint.config.mjs` |
| …with `typescript-eslint@8.72.2` (latest) | ❌ | `ConfigError: Key "plugins": Cannot redefine plugin "@typescript-eslint"`: two copies of the plugin |
| …with `typescript-eslint@8.70.0` exact (deduped with node-cli) | ✅ | `npm ls`: `typescript-eslint@8.70.0 deduped` |
| `import-x/*` rules (e.g. `no-cycle`) in the strict layer | ✅ | Plugin already registered by the n8n config; no extra dependency |
| Type-aware rules on `test/**` | ⚠️ | "was not found by the project service" until `test/tsconfig.json` exists (extending root, including `test`, `nodes`, `credentials`). Then ✅ |
| What strict-type-checked finds on idiomatic n8n code | ⚠️ | Fires on the CLI's own template: `no-unsafe-assignment`/`no-unsafe-return` (`httpRequestWithAuthentication` returns `Promise<any>`), `restrict-template-expressions`/`no-base-to-string` (`getNodeParameter` returns `NodeParameterValueType \| object`), `consistent-type-definitions`, `consistent-indexed-object-style`, `prefer-regexp-exec`, `array-type`, `no-unnecessary-type-assertion`. Every one is fixable. Design implication: one typed transport wrapper that returns `unknown`, plus typed parameter getters |
| ESLint 10.14.0 instead of 9.x | ✅ (probe) | `n8n-node lint` ran with the same results. node-cli internally depends on `eslint 9.29.0` (peer `>= 9`) and the starter uses 9.39.4. Recommend staying on 9.39.x until node-cli moves (unverified beyond the probe) |
| `coverage/` HTML report present | ⚠️ | `n8n-node lint` warns on `coverage/*.js` (only `dist` is ignored, and the config can't be changed) |

### Testing

| Item | Result | Evidence |
|---|---|---|
| vitest 5.1.0 + `@vitest/coverage-v8` + thresholds | ✅ | Probe test passes; thresholds fail the run as configured (observed) |
| `vitest-mock-extended` `mockDeep<IExecuteFunctions>()` | ✅ | `mock<>()` alone leaves `helpers.*` undefined; use `mockDeep` (observed) |
| nock 15 intercepting a fake `httpRequestWithAuthentication` built on global `fetch` | ✅ | Request matched, `scope.isDone()` true (observed) |
| `zod/v4` schemas in tests (zod 3.25.76) | ✅ | Observed |
| Tests in `test/` vs `n8n-node lint` | ⚠️ | `test/**/*.ts` is linted with the community rules. `process.env` → `no-restricted-globals` error, `setTimeout` → error (observed). devDependency imports are exempt. Inline disables work there because the scanner never reads `test/` |
| Tests in `nodes/**` | ⚠️ / ❌ | Compiled into `dist/` and published unless excluded from `tsconfig.json`; linted by the scanner with no inline disables. Apify keeps `nodes/Apify/__tests__` (excluded in tsconfig) and imports `http`/`https` there, which the current scanner rules would flag (unverified; not run against Apify) |
| n8n monorepo approach | source | `packages/nodes-base/package.json`: `vitest run`, `@vitest/coverage-v8`, `nock`. Workflow-level tests use `NodeTestHarness` (`packages/core/nodes-testing/node-test-harness.ts`): it executes workflow JSON with `WorkflowExecute`, uses `pinData` as expected output and nock for HTTP |
| Reusing `NodeTestHarness` | ❌ | Not published. `n8n-core` `files: ["dist","bin"]`; the 2.47.0 tarball has no `nodes-testing`. Porting it means depending on `n8n-core` dist internals, under n8n's licence |
| Community precedent | source | `@apify/n8n-nodes-apify`: jest 30 + ts-jest + nock 14, hand-rolled `executeWorkflow` harness, workflow JSON fixtures |
| msw 3 | unverified | Not exercised; nock 15 covers the need |

### Contract tests

| Item | Result | Evidence |
|---|---|---|
| `@plunk/types` on npm | ❌ | npm 404. `packages/types/package.json` is `"private": true` (useplunk/plunk@`next`, commit `37e5f81`, 2026-11-09) |
| `@plunk/types` content | source | Plain TS interfaces, no zod (e.g. `packages/types/src/api/email.ts`: `SendEmailResult`, `EmailResource`) |
| Plunk's zod request schemas | source | `packages/shared/src/schemas/index.ts` (759 lines; `ContactSchemas`, `EmailSchemas`, `DeliverySchemas`, …). Written for zod `^3.23.8`; imports `@plunk/db` (Prisma enums) and `@plunk/types`, so it can't be copied verbatim. `@plunk/shared` is private too |
| Plunk OpenAPI | source | `apps/wiki/openapi.json`: OpenAPI 3.1, only **11 paths** and 11 component schemas (`/v1/send`, `/v1/track`, `/v1/verify`, `/contacts`, `/contacts/{id}`, `/emails/{id}`, `/emails/{id}/cancel`, `/campaigns`, `/campaigns/{id}/send`, `/segments`, `/templates`). Partial |
| zod@4 + explicit `n8n-workflow` devDependency | ❌ | npm ERESOLVE (`peer zod@"3.25.76" from n8n-workflow@2.47.0`) |
| zod@4 without declaring `n8n-workflow` | ⚠️ | Installs with "overriding peer dependency" warnings; fragile |
| zod@3.25.76 devDependency + `import { z } from 'zod/v4'` | ✅ | Satisfies the peer; v4 API available |
| zod at **runtime** | ⚠️ | Not our decision here (sibling ticket). `no-restricted-imports` allows `zod`, but `no-runtime-dependencies` and `valid-peer-dependencies` forbid declaring it. It would be an undeclared import resolved from the host, and its version is coupled to n8n's (3.25.x today) |

### fallow 3.42.0 (observed)

| Item | Result | Evidence |
|---|---|---|
| Zero-config run | ⚠️ | All `*.node.ts` and `*.credentials.ts` files reported as **unused files**. fallow doesn't read `package.json#n8n.nodes` (which points at `dist/*.js`). Fix: `entry` |
| `entry: ["nodes/*/*.node.ts", "credentials/*.credentials.ts", "test/**/*.test.ts"]` | ✅ | Remaining findings were real: a seeded unused export, an unused type, unused devDependencies |
| `dist/` output | ✅ | Not analysed; fallow noted that the `dist/**` ignore pattern "excluded no file", so it needs no ignore entry |
| `release-it` flagged as an unused devDependency | ⚠️ | False positive: node-cli spawns the `release-it` binary. Needs `ignoreDependencies: ["release-it"]` |
| Boundary zones + rules | ✅ | Caught `transport → resources/email` and `resources/email → resources/contact`. `fallow architecture` exits 1 |
| `autoDiscover: ["nodes/Plunk/resources"]` | ⚠️ | Creates zones `resources/<child>`. Docs: the **generated parent rule allows imports between the discovered children**, and "zones without a rule have no restrictions". Cross-resource isolation therefore needs an explicit rule per child, and a new resource folder starts out unrestricted until someone adds its rule |
| Glob semantics | ⚠️ | Since 3.34, `*` matches a single segment only (fallow emits a WARN when a pattern matched more before) |
| `boundaries.calls.forbidden` | ⚠️ | `*.httpRequestWithAuthentication` does **not** match `this.helpers.httpRequestWithAuthentication.call(this, …)`, the idiomatic n8n call. You need `*.httpRequestWithAuthentication.call` as well; `*.httpRequest` matched |
| `boundaries.coverage.requireAllFiles` | ✅ | Flags unzoned files (e.g. `.prettierrc.js`); use `allowUnmatched` for root config files |
| Health / CRAP | ✅ | Auto-reads `coverage/coverage-final.json` (Istanbul JSON) for CRAP scores, so run fallow **after** tests with coverage. Defaults: `maxCyclomatic 20`, `maxCognitive 15`, `maxCrap 30`, `maxUnitSize 60` |
| Duplication | ✅ | 0 clones on the template's look-alike declarative `getAll.ts` files (default `mild`, `minTokens 50`, `minLines 5`) |
| Exit codes | ✅ | `fallow --fail-on-issues` → 1. Bare `fallow --format json` → 0 even with findings (README: "Bare `fallow` in a machine format exits 0 on findings unless you pass `--fail-on-issues`") |
| CI | source | `fallow audit` (changed-code gate, needs `fetch-depth: 0`) or whole-project `fallow`. GitHub Action `fallow-rs/fallow@v3` (`skills/fallow-setup/references/ci-gate.md` in the npm package) |

### Hooks (observed unless noted)

| Item | Result | Evidence |
|---|---|---|
| husky `"prepare": "husky"` | ❌ | `@n8n/community-nodes/no-forbidden-lifecycle-scripts` error in `n8n-node lint` |
| lefthook auto-install | ⚠️ | Installs hooks from **its own** `postinstall`, not ours, but npm 11.19 skips dependency install scripts unless approved. `npm install-scripts approve lefthook` writes `"allowScripts": {"lefthook@2.2.2": true}` into `package.json`, and `n8n-node lint` accepts that key. Fallback: `npx lefthook install` in CONTRIBUTING |
| lint-staged | n/a | Not needed: lefthook has `{staged_files}` and `glob` built in (source: lefthook docs; not exercised end-to-end) |

## Recommended baseline (sketch)

### Layout

```
credentials/PlunkApi.credentials.ts
nodes/Plunk/Plunk.node.ts            # thin: description + dispatch
nodes/Plunk/{Plunk.node.json,*.svg}
nodes/Plunk/resources/<resource>/…   # descriptions + operation handlers
nodes/Plunk/transport/…              # the only place that performs HTTP
nodes/Plunk/shared/…                 # types, pure helpers
test/{unit,node,contract}/…          # never under nodes/ or credentials/
test/tsconfig.json
eslint.config.mjs                    # byte-identical to template — never edit
eslint.strict.config.mjs
.fallowrc.json
lefthook.yml
vitest.config.mts
```

### `tsconfig.json` (the build config; `n8n-node build` runs bare `tsc`)

Start from the template and change the following:

- `"module": "node16"`, `"moduleResolution": "node16"`, replacing `commonjs`/`node`. Keep CJS output (no `"type": "module"`).
- Remove `"useUnknownInCatchVariables": false`, so `strict` turns it on.
- Add `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `noFallthroughCasesInSwitch`, `noUnusedParameters`, `isolatedModules`, `allowUnreachableCode: false`, `allowUnusedLabels: false`. Keep the template's `noUnusedLocals` and `noImplicitReturns`.
- Optionally add `erasableSyntaxOnly` (0 errors; it bans enums and namespaces).
- Keep `skipLibCheck: true`, `target`/`lib` es2019 as in the template (see Open questions), and `include: ["credentials/**/*", "nodes/**/*", "nodes/**/*.json", "package.json"]`.
- **Do not add** `verbatimModuleSyntax` or `isolatedDeclarations`.
- Add `test/tsconfig.json`: `extends: "../tsconfig.json"`, `noEmit: true`, `incremental: false`, `include: ["./**/*.ts", "../nodes/**/*.ts", "../credentials/**/*.ts"]`. It serves type-checking and ESLint's project service.
- Toolchain: TypeScript **6.0.x**, exact pin. Bump in lockstep with typescript-eslint's peer range.

### ESLint layering

```js
// eslint.strict.config.mjs — run via `eslint -c eslint.strict.config.mjs --max-warnings 0 .`
import { config as n8nConfig } from '@n8n/node-cli/eslint';
import { defineConfig, globalIgnores } from 'eslint/config';
import tseslint from 'typescript-eslint'; // MUST be the exact version @n8n/node-cli pins

export default defineConfig(
  globalIgnores(['dist', 'coverage']),
  n8nConfig, // includes every n8n rule, so this run is a superset of `n8n-node lint`
  {
    files: ['**/*.ts'],
    extends: [tseslint.configs.strictTypeChecked, tseslint.configs.stylisticTypeChecked],
    languageOptions: { parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname } },
    rules: {
      '@typescript-eslint/consistent-type-imports': 'error',
      '@typescript-eslint/switch-exhaustiveness-check': 'error',
      'import-x/no-cycle': 'error',
      // project-specific rules (naming, no-restricted-syntax, …) go here
    },
  },
  { files: ['test/**/*.ts'], rules: { /* targeted relaxations only, e.g. non-null assertions on fixtures */ } },
);
```

- Keep `typescript-eslint` in sync with node-cli: for example, add a CI step that compares `npm ls typescript-eslint` output, or use a Renovate group for `@n8n/node-cli` + `typescript-eslint` (the check itself is unverified).
- `--max-warnings 0` turns n8n's warning-level rules (`operation-standard-action`, `open-ended-node-fields`, …) into gates as well.

### Test stack

- **Runner:** vitest 5 + `@vitest/coverage-v8`. Coverage `include: ['nodes/**/*.ts', 'credentials/**/*.ts']`, global thresholds (start at 80 % and ratchet up).
- **Reporters:** `['text', 'json', 'json-summary']`, no HTML. That avoids `coverage/*.js` lint noise and stray PNGs that `n8n-node build` would copy into `dist/`; the `json` reporter feeds fallow's CRAP scores.
- **Layers:**
  1. Pure unit tests: mappers, validators, error translation.
  2. Node-execute tests: `mockDeep<IExecuteFunctions>()`, with `helpers.httpRequestWithAuthentication` backed by a `fetch` shim, nock with `disableNetConnect()`, and assertions on `pairedItem`, `continueOnFail` and `NodeApiError`.
  3. Contract tests: zod (`zod@3.25.76`, `import { z } from 'zod/v4'`) schemas for every request and response we rely on. Each schema is checked against nock fixtures *and* against the request bodies our node builds. Record the Plunk commit SHA the schemas were last verified against.
  4. Optional e2e smoke against local Docker Plunk (map decision), tagged and excluded from default CI.
- Workflow-level tests à la `NodeTestHarness` aren't available off the shelf. See Open questions.

### fallow (`.fallowrc.json`, sketch)

```jsonc
{
  "$schema": "./node_modules/fallow/schema.json",
  "entry": ["nodes/*/*.node.ts", "credentials/*.credentials.ts", "test/**/*.test.ts"],
  "ignoreDependencies": ["release-it"],
  "rules": {
    "unused-files": "error", "unused-exports": "error", "unused-types": "error",
    "unused-dependencies": "error", "unused-dev-dependencies": "error",
    "circular-dependencies": "error", "boundary-violation": "error"
  },
  "boundaries": {
    "zones": [
      { "name": "node", "patterns": ["nodes/*/*.node.ts"] },
      { "name": "transport", "patterns": ["nodes/*/transport/**"] },
      { "name": "shared", "patterns": ["nodes/*/shared/**"] },
      { "name": "resources", "patterns": ["nodes/*/resources/*.ts"], "autoDiscover": ["nodes/Plunk/resources"] },
      { "name": "credentials", "patterns": ["credentials/**"] },
      { "name": "tests", "patterns": ["test/**"] }
    ],
    "rules": [
      { "from": "node", "allow": ["resources", "resources/contact", "resources/email", "transport", "shared"] },
      { "from": "resources/contact", "allow": ["transport", "shared"] },
      { "from": "resources/email", "allow": ["transport", "shared"] },
      { "from": "transport", "allow": ["shared"] },
      { "from": "shared", "allow": [] },
      { "from": "credentials", "allow": [] }
    ],
    "coverage": { "requireAllFiles": true, "allowUnmatched": ["*.js", "*.mjs", "*.mts"] },
    "calls": { "forbidden": [
      { "from": "resources/contact", "callee": ["*.httpRequest", "*.httpRequestWithAuthentication", "*.httpRequestWithAuthentication.call", "fetch"] }
      // repeat per resource zone; also for "node" if the node class must not do HTTP itself
    ] }
  }
}
```

The resource names (`contact`, `email`) are placeholders; the architecture ticket decides the real set. Because of the `autoDiscover` behaviour above, every new resource folder needs its own `rules` and `calls` entries. Enforce that with a small test that lists `nodes/Plunk/resources/*` and asserts each name appears in `.fallowrc.json` (unverified idea).

### Hooks (`lefthook.yml`, sketch)

- **pre-commit** (parallel, staged files only): `prettier --check {staged_files}` and `eslint -c eslint.strict.config.mjs --max-warnings 0 {staged_files}` on `*.ts`.
- **pre-push:** `npm run typecheck`, `npm test`, `npx fallow audit`.
- Install via `allowScripts` approval, or `npx lefthook install` documented in CONTRIBUTING.
- Never add `prepare` or `postinstall` to our `package.json`.

### Scripts and CI gate order

```jsonc
"scripts": {
  "build": "n8n-node build",
  "dev": "n8n-node dev",
  "lint": "n8n-node lint",                                     // the verification gate (unchanged config)
  "lint:strict": "eslint -c eslint.strict.config.mjs --max-warnings 0 .",
  "typecheck": "tsc --noEmit -p tsconfig.json && tsc -p test/tsconfig.json",
  "test": "vitest run",
  "test:coverage": "vitest run --coverage",
  "fallow": "fallow --fail-on-issues",
  "format:check": "prettier --check .",
  "check": "npm run format:check && npm run typecheck && npm run lint && npm run lint:strict && npm run test:coverage && npm run fallow && npm run build",
  "release": "n8n-node release",
  "prepublishOnly": "n8n-node prerelease"
}
```

CI order, from cheapest and most fundamental to most expensive:

1. `npm ci`
2. Format check
3. Typecheck (build and test projects)
4. `n8n-node lint`
5. Strict ESLint
6. Vitest with coverage thresholds
7. fallow (after coverage, so CRAP uses real data; `fallow audit` on PRs, full `fallow --fail-on-issues` on `main`)
8. `n8n-node build`, then `npm pack --dry-run` to assert the tarball contains only `dist/**`, `package.json`, README and LICENSE

For publishing: the template `publish.yml` runs `npm run release`, which in CI runs only `lint` + `build`. Add an `npm run check` step before it. Optionally add a post-publish job that runs `npx @n8n/scan-community-package n8n-nodes-plunk@<version>` (exit 2 means a check failed). It can't run before publishing, because it needs the npm provenance attestation.

## Open questions

1. **Should `lint` include `lint:strict`?** That would make `n8n-node release` enforce it locally and in CI, but `n8n-node lint`'s own output stays the verification signal. Either option is compatible.
2. **TS version policy.** 6.0.x now; TS 7 only once typescript-eslint supports it (no date found). Also check whether `n8n-node dev` (`tsc --watch`) behaves the same on TS 6 (not exercised).
3. **`target`/`lib`.** The template's es2019 is conservative. The right floor depends on n8n's supported Node versions (not researched here).
4. **fallow `autoDiscover` semantics.** Does a rule `{ from: "resources" }` apply to the generated children? The docs say an explicit child rule overrides the generated one, and they don't cover the parent case (unverified). Pick a guard against unruled new resource zones.
5. **Licensing.** Plunk is AGPL-3.0 and our repo is MIT; copying Plunk's schemas or `openapi.json` into the repo, even under `test/`, may raise licence questions. Porting n8n's `NodeTestHarness` (n8n licence) raises the same question. Recommendation: write our own zod schemas from reading the source, or fetch Plunk at a pinned SHA in CI without committing it. This is not legal advice.
6. **Workflow-level tests.** Either build a small harness on `n8n-core` as a devDependency (couples us to its dist internals), or rely on the Docker e2e smoke. Unit tests plus node-execute tests plus contract tests may be enough.
7. **Runtime validation** (sibling ticket). The rules technically allow importing host-provided `zod` without declaring it. Is that acceptable for verification, and wise given the version coupling?
8. **`n8n-workflow` devDependency pin.** Its npm `latest` tag (2.16.0) lags n8n `stable` (2.46.x/2.47.x), so a plain `npm i -D n8n-workflow` gets an old version. Pin it explicitly to the version node-cli resolves, or to n8n stable, and keep it together with `zod@3.25.76`.
9. **ESLint 9 vs 10.** 10.14.0 worked in the probe; the conservative choice is 9.39.x, matching `n8n-nodes-starter`.
10. **`noPropertyAccessFromIndexSignature` + `@typescript-eslint/dot-notation`.** typescript-eslint documents that `dot-notation` honours this flag. That wasn't exercised here (unverified).

## Unverified / not exercised

- The real verification scan of a published package (it needs provenance).
- msw 3.
- lefthook hooks actually firing.
- Whether Apify's package passes the current scanner.
- Whether HTML coverage PNGs end up in `dist/` (inferred from `build.js`'s copy glob).
- `@n8n/scan-community-package`'s `buildScanConfig` is exported from `dist/scanner.js`, but the package has no `exports`/`main` for programmatic reuse, so running the scanner's exact config pre-publish wasn't attempted.

## Citations

- `@n8n/node-cli@0.56.1` tarball: `package.json`; `dist/commands/{build,lint,release,prerelease,cloud-support}.js`; `dist/configs/eslint.js`; `dist/template/templates/declarative/*/template/{package.json,tsconfig.json}`; `dist/template/templates/shared/default/{eslint.config.mjs,.github/workflows/ci.yml,.github/workflows/publish.yml}`
- `@n8n/eslint-plugin-community-nodes@0.40.0` (and 0.39.0 for `no-restricted-imports`): `src/plugin.ts`; `src/rules/no-restricted-imports.ts`; `src/rules/no-forbidden-lifecycle-scripts.ts`; `src/utils/file-utils.ts`; `docs/rules/{no-restricted-imports,no-restricted-globals,no-runtime-dependencies,no-dead-files,require-files-array,no-forbidden-lifecycle-scripts,valid-peer-dependencies}.md`
- `@n8n/scan-community-package@0.43.0`: `README.md`; `dist/scanner.js` (`SOURCE_FILE_PATTERNS`, `buildScanConfig`, `analyzePackage` with `allowInlineConfig: false`)
- `@n8n/typescript-config@1.12.0`: `tsconfig.common.json`
- n8n monorepo (`n8n-io/n8n@master`): `packages/core/nodes-testing/node-test-harness.ts`; `packages/core/package.json` (`files`); `packages/nodes-base/package.json` (test scripts and devDeps)
- `n8n-io/n8n-nodes-starter` (commit `6240cb4`, 2026-08-26): `package.json`, `tsconfig.json`
- `apify/n8n-nodes-apify`: `tsconfig.json`; `nodes/Apify/__tests__/utils/executeWorkflow.ts`; npm `@apify/n8n-nodes-apify` devDependencies
- `useplunk/plunk@next` (commit `37e5f81`, 2026-11-09): `packages/types/package.json`; `packages/types/src/api/email.ts`; `packages/shared/package.json`; `packages/shared/src/schemas/index.ts`; `apps/wiki/openapi.json`; `apps/wiki/scripts/generate-openapi.js`
- `fallow@3.42.0` tarball: `README.md`; `schema.json` (`BoundaryConfig`, `BoundaryZone`, `BoundaryRule`, `ForbiddenCallRule`, `HealthConfig`, `DuplicatesConfig` defaults); `skills/fallow-setup/references/ci-gate.md`
- fallow docs: https://fallow.tools/docs/ ; https://fallow.tools/docs/configuration/boundaries/
- `lefthook@2.2.2` tarball: `package.json` (`postinstall`), `postinstall.js`
- n8n verification guidelines: https://docs.n8n.io/integrations/creating-nodes/build/reference/verification-guidelines/ ; publishing with provenance: https://docs.n8n.io/integrations/creating-nodes/deploy/submit-community-nodes/ (linked from `release.js`)
- npm registry metadata (`npm view <pkg> version dist-tags peerDependencies dependencies`), 2026-11-10
