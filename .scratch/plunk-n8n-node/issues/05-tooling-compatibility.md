# Strict tooling compatibility with n8n node tooling

Type: research
Status: resolved
Blocked by: —
Map: [../map.md](../map.md)

## Question

How far can we push strictness while staying compatible with `@n8n/node-cli` and verification? Cover: which TypeScript strict flags work with `n8n-workflow` types; layering strict ESLint (typescript-eslint strict-type-checked etc.) on top of the n8n lint config; how existing nodes are tested (Vitest/Jest, HTTP mocking via nock/msw, n8n's own test helpers, workflow-level tests); using zod as devDependency for contract tests, possibly against Plunk's `packages/types`; fallow configuration for this layout (architecture boundaries, dead code, duplication) and running it in CI; pre-commit hooks.

## Answer

Full findings: branch `research/tooling-compatibility`, file `claudedocs/research/05-tooling-compatibility.md` (compatibility matrix + baseline config sketch; parts verified by running them).

Very strict tooling is compatible, under four constraints:
- **ESLint**: with `n8n.strict: true`, `n8n-node lint` fails if `eslint.config.mjs` differs from the template, and turning strict off forfeits Cloud verification. Strict rules go in a second `eslint.strict.config.mjs` (`eslint -c`), reusing n8n's config + typescript-eslint `strictTypeChecked` + `stylisticTypeChecked`. `typescript-eslint` must match node-cli's pin (8.70.0); 8.72.2 fails with "Cannot redefine plugin".
- **TypeScript 6.0.x** with `module`/`moduleResolution: node16` (template's `node` is deprecated in TS 6, removed in TS 7; TS 7 conflicts with typescript-eslint 8.70's `typescript <6.1.0` peer). All strict flags work except `verbatimModuleSyntax` (n8n loads CJS) and `isolatedDeclarations` (friction, no gain). `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, `noPropertyAccessFromIndexSignature` need small local fixes; `skipLibCheck` stays true.
- **Tests in top-level `test/`** with their own `test/tsconfig.json`. `n8n-node lint` still lints `test/`, so no `process`/`setTimeout`/Node built-ins except `crypto` there (devDependency imports exempt). `n8n-node build` is bare `tsc` on `tsconfig.json`.
- **lefthook, not husky** (husky's `prepare` script violates the lifecycle-script ban).

Testing and contracts:
- vitest 5 + `vitest-mock-extended` (`mockDeep<IExecuteFunctions>()`) + nock 15 works (ran it). n8n's `NodeTestHarness` isn't published. Coverage output JSON/text only (n8n lints `coverage/*.js`; build would copy report assets into `dist/`).
- `n8n-workflow@2.47.0` peers `zod: 3.25.76`, so use `zod@3.25.76` with `import { z } from 'zod/v4'` (zod 4 package fails to install alongside).
- Plunk's schemas (`packages/shared`, zod 3) are private and depend on Plunk DB types; `openapi.json` covers 11 paths. → Write our own zod schemas from Plunk's source and record the Plunk commit checked against. Don't copy AGPL files.

fallow 3.42.0:
- Needs explicit `entry` (doesn't read n8n's `package.json` node list); false positive on `release-it` as unused dev dep.
- Boundary rules caught both planted violations; a forbidden-call rule can keep HTTP inside `transport/` (pattern must end in `.call` for `this.helpers.httpRequestWithAuthentication.call(...)`). Auto-discovered resource folders may import each other until each has an explicit rule.
- Runs after tests (reads coverage JSON).

**CI order**: format → typecheck → `n8n-node lint` → strict ESLint → vitest + coverage → fallow → build + `npm pack --dry-run`. `n8n-node release` in CI only lints + builds, so `publish.yml` needs a full check step first.

Open questions handed on → Engineering standards: does `lint` include the strict layer; TS 7 timeline; `target`; fallow zones for auto-discovered resources; workflow-level tests; ESLint 9 vs 10; stale `n8n-workflow` `latest` tag. Not verified: real scanner run, msw, lefthook firing, `dot-notation` × `noPropertyAccessFromIndexSignature`.
