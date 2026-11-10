# 03 — n8n community node and verification rules (state on 2026-11-10)

Ticket: `.scratch/plunk-n8n-node/issues/03-n8n-verification-rules.md`
Researched: 2026-11-10. All sources were seen on that date.

## Source snapshot

Every claim below cites one of these sources. Citation keys are in brackets, for example [VG].

| Key | Source | Pinned at |
|---|---|---|
| VG | Verification guidelines: https://docs.n8n.io/connect/create-nodes/build-your-node/reference/verification-guidelines (`n8n-io/n8n-docs` `docs/connect/create-nodes/build-your-node/reference/verification-guidelines.md`) | docs `c3575d8` (main, 2026-11-10) |
| SUB | Submit community nodes: https://docs.n8n.io/connect/create-nodes/deploy-your-node/submit-community-nodes (raw: same URL + `.md`; source include `docs/reusable-content/.gitbook/includes/integrations/submit-community-node.md`, last changed 2026-10-29 "document review timeline", commit `89f22db`) | live page, 2026-11-10 |
| UX | UX guidelines: https://docs.n8n.io/connect/create-nodes/build-your-node/reference/ux-guidelines | docs `c3575d8` |
| CLI-DOC | Using the n8n-node tool: https://docs.n8n.io/connect/create-nodes/build-your-node/using-the-n8n-node-tool | docs `c3575d8` |
| LINT-DOC | Node linter: https://docs.n8n.io/connect/create-nodes/test-your-node/node-linter | docs `c3575d8` |
| STYLE | Choose a node building style: https://docs.n8n.io/connect/create-nodes/plan-your-node/choose-a-node-building-style | docs `c3575d8` |
| TYPES | Choose a node type: https://docs.n8n.io/connect/create-nodes/plan-your-node/choose-a-node-type | docs `c3575d8` |
| CRED-DOC | Credentials files: https://docs.n8n.io/connect/create-nodes/build-your-node/reference/credentials-files | docs `c3575d8` |
| CODE-STD | Code standards: https://docs.n8n.io/connect/create-nodes/build-your-node/reference/code-standards | docs `c3575d8` |
| CODEX | Codex files: https://docs.n8n.io/connect/create-nodes/build-your-node/reference/codex-files | docs `c3575d8` |
| STDPARAM | Standard parameters: `docs/connect/create-nodes/build-your-node/reference/base-files/standard-parameters.md` | docs `c3575d8` |
| ICONS | Node icons include: `docs/reusable-content/.gitbook/includes/integrations/creating-nodes/node-icons.md` | docs `c3575d8` |
| INST | Install and manage community nodes: `docs/integrations/community-nodes/installation-and-management/README.md`, `install-verified-community-nodes.md`, `gui-installation.md`, `environment-variable-installation.md` | docs `c3575d8` |
| RISKS | `docs/integrations/community-nodes/risks.md`, `blocklist.md` | docs `c3575d8` |
| ENV | Nodes env vars: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/nodes | live page, 2026-11-10 |
| V3 | n8n 3.0 breaking changes: `docs/changelog/v30-breaking-changes.md` ("Breaking changes coming in n8n 3.0") | docs `c3575d8` |
| CLI | `n8n-io/n8n` `packages/@n8n/node-cli/` (README, `src/commands/{lint,release,prerelease,build,cloud-support,dev/index}.ts`, `src/configs/eslint.ts`, `src/template/templates/**`) | n8n master `d5c034c` (2026-11-10) |
| PLUGIN | `n8n-io/n8n` `packages/@n8n/eslint-plugin-community-nodes/` (README, `src/plugin.ts`, `src/rules/*.ts`, `docs/rules/*.md`) | n8n master `d5c034c` |
| SCAN | `n8n-io/n8n` `packages/@n8n/scan-community-package/` (README, `src/scanner.ts`, `src/provenance.ts`) | n8n master `d5c034c` |
| CORE | `n8n-io/n8n` `packages/cli/src/modules/community-packages/{community-packages.config.ts,community-node-types-utils.ts,community-node-types.service.ts}`, `packages/@n8n/constants/src/community-nodes.ts`, `packages/workflow/src/nodes-api-version.ts` | master `d5c034c` and branch `3.x` `254dacf` (2026-11-10) |
| STARTER | `n8n-io/n8n-nodes-starter` (README, `package.json`, `nodes/Example/Example.node.json`) | master `6240cb4` (2026-08-26) |
| NPM | npm registry via `npm view` (versions/dist-tags), 2026-11-10 | — |
| NPM-TP | npm trusted publishers: https://docs.npmjs.com/trusted-publishers | live page, 2026-11-10 |
| CAT | Public verified-node catalog: `https://api.n8n.io/api/community-nodes` (the URL n8n itself calls, per CORE `community-node-types-utils.ts`) | live, 2026-11-10 |
| FORUM | community.n8n.io threads 290662, 302391, 325906, 275867, 233386 (user reports plus one n8n community moderator reply) | live, 2026-11-10 |

Versions on npm [NPM]: `@n8n/node-cli` latest `0.56.1` (`beta` 0.57.0; `rc` 1.2.0 is the n8n 3.x line); `@n8n/scan-community-package` `0.43.0`; `@n8n/eslint-plugin-community-nodes` `0.40.0`; `n8n` latest/stable `2.47.3`, so n8n 3.0 is not released yet.

---

## 1. Hard requirements checklist (verification)

"Hard" means a doc states it as a must, or an automated check enforces it (the CLI strict lint or the scanner). Verification fails if any of these is missed.

### Package and repository

- [ ] **Package name** starts with `n8n-nodes-` or `@scope/n8n-nodes-` [SUB; PLUGIN `package-name-convention`]. `n8n-nodes-plunk` is free on npm [NPM].
- [ ] **Keyword** `n8n-community-node-package` is in `keywords` [SUB].
- [ ] **`n8n` object** in package.json [PLUGIN `n8n-object-validation`]:
  - `n8nNodesApiVersion` sits inside `n8n` and is a positive integer or a `"<major>"`/`"<major>.<minor>"` string.
  - `nodes` is a non-empty array of `dist/...` paths. `./dist` and `DIST` are rejected.
  - `credentials`, if present, holds `dist/...` paths.
- [ ] **`n8nNodesApiVersion: 1`.** n8n 2.x supports level `1.0` and the `3.x` branch supports `3.0`. A package is installable only if its declared level is ≤ the runtime's level [CORE `constants/src/community-nodes.ts` on master vs `3.x`; `workflow/src/nodes-api-version.ts`]. So `1` works on both, while `3` would be rejected by every 2.x instance. The 3.x templates still emit `1` [CORE, 3.x `node-cli` template package.json].
- [ ] **No runtime `dependencies`.** The field must be empty or absent [VG "No external dependencies"; SUB "verified community nodes aren't allowed to use any run-time dependencies"; PLUGIN `no-runtime-dependencies`, error].
- [ ] **`peerDependencies` is exactly `{"n8n-workflow": "*"}`.** `@n8n/ai-node-sdk` is also allowed, but only for AI nodes. No ranges and no `n8n-core` [PLUGIN `valid-peer-dependencies`].
- [ ] **No lifecycle scripts** (`prepare`, `preinstall`, `install`, `postinstall`, `prepublish`, `preprepare`, `postprepare`) [PLUGIN `no-forbidden-lifecycle-scripts`]. `prepublishOnly` is allowed: the template itself uses `"prepublishOnly": "n8n-node prerelease"` [CLI template package.json].
- [ ] **No `overrides` field** in package.json [PLUGIN `no-overrides-field`].
- [ ] **Valid `version`, non-empty `description`, and `author` with both name and email.** No `<...>` template placeholders [PLUGIN `require-version`, `valid-description`, `valid-author`, `no-template-placeholders`].
- [ ] **MIT license** [VG "Make sure your package license is MIT"]. The GitHub API currently detects no license on `krautnerds/n8n-nodes-plunk`, so a `LICENSE` file is needed.
- [ ] **Public GitHub repository**, with npm `repository.url` matching it. Author/maintainer must match between npm and the repo, and the git link on npm must work [VG "Package source verification"]. npm trusted publishing also requires `repository.url` to match the GitHub repo exactly [NPM-TP].
- [ ] **README documentation**: usage instructions, example workflows, and auth details [VG "Proper documentation"; SUB].
- [ ] **English only** in node UI, parameter names, descriptions, hints, error messages and README [VG].

### Publishing

- [ ] **Publish from GitHub Actions with an npm provenance statement.** This applies from 2026-05-01: "n8n won't accept verified nodes published directly from a local machine" [SUB; VG; CLI `release.ts`].
  - The scanner's first step checks for a SLSA v1 provenance attestation [SCAN `provenance.ts`, `NPM_PROVENANCE_PREDICATE_TYPE = 'https://slsa.dev/provenance/v1'`].
  - Next, the scanner downloads the **attested commit** from GitHub via codeload. That commit must be public and reachable, or the scan fails [SCAN README, `scanner.ts`].
- [ ] **`@n8n/node-cli` ≥ 0.23.0 as a devDependency.** Earlier versions lack the provenance flag [SUB; CLI `publish.yml` header].

### Code (enforced by lint and scanner)

- [ ] **`npx @n8n/scan-community-package n8n-nodes-plunk` passes** [VG "Follow n8n best practices"]. It lints two things with its own built-in config [SCAN `scanner.ts` `buildScanConfig`, `analyzePackageByName`]:
  1. the attested source: `package.json` plus `{nodes,credentials}/**/*.{js,ts,json}`;
  2. the published tarball: every `**/*.js` plus `package.json`.

  Only **errors** fail the scan (`errorCount > 0`). Warnings do not.
- [ ] **No `eslint-disable` escape hatch at verification.** The scanner runs ESLint with `allowInlineConfig: false` [SCAN `scanner.ts` `analyzePackage`], so inline disable comments are ignored. Every error-level rule must pass for real.
- [ ] **Cloud-only rules pass.**
  - `no-restricted-imports`: only `n8n-workflow`, `lodash`, `moment`, `p-limit`, `luxon`, `zod`, `crypto`/`node:crypto`, `@n8n/ai-node-sdk`, relative paths, or packages listed in `devDependencies`.
  - `no-restricted-globals`: no `process`, `global(This)`, `__dirname`, `__filename`, `setTimeout`/`setInterval`/`setImmediate` or their `clear*` counterparts.

  [PLUGIN `src/rules/no-restricted-imports.ts`, `docs/rules/no-restricted-globals.md`]. For delays, use `sleep`/`sleepWithAbort` from `n8n-workflow` [PLUGIN `no-restricted-globals.md`].
- [ ] **No environment-variable access and no file-system reads or writes.** Pass all data through node parameters [VG].
- [ ] **No `eval`, no `new Function`, no `child_process`** [PLUGIN `no-dangerous-functions`]. No hard-coded secrets [PLUGIN `no-hardcoded-secrets`]. No `console` (`no-console: error` in both the CLI config and the scanner config) [CLI `configs/eslint.ts`; SCAN `scanner.ts`].
- [ ] **Every credential has a `test`**, or a node covers it via `testedBy`. OAuth2 credentials that `extends: ['oAuth2Api']` are exempt [PLUGIN `src/rules/credential-test-required.ts`].
- [ ] **Credential settings:**
  - sensitive-named fields set `typeOptions.password: true` [PLUGIN `credential-password-field`; UX "API key ... should always be password fields"];
  - `documentationUrl` is a full URL, not a slug [PLUGIN `credential-documentation-url`];
  - the credential class has an `icon` [PLUGIN `cred-class-field-icon-missing`];
  - nodes reference only this package's credentials [PLUGIN `no-credential-reuse`, `valid-credential-references`].
- [ ] **`usableAsTool`** is declared on every action node. Trigger nodes (`group` includes `'trigger'`, or the class name ends in `Trigger`) must **not** set `usableAsTool: true` [PLUGIN `node-usable-as-tool`].
- [ ] **Node description includes `icon` and `subtitle`** [PLUGIN `require-node-description-fields`].
  - Icons use the `file:` protocol and the files must exist [PLUGIN `icon-validation`]. SVG is recommended; PNG is allowed at 60×60 [ICONS].
  - Inputs and outputs use the `NodeConnectionTypes` enum with no `as any`/`as never` casts [PLUGIN `node-connection-type-literal`, `no-unsafe-connection-type-cast`].
- [ ] **Resource/operation pattern** [PLUGIN `resource-operation-pattern`]. No emoji in option names [PLUGIN `no-emoji-in-options`]. Codex `categories` come only from: Data & Storage, Finance & Accounting, Marketing & Content, Productivity, Miscellaneous, Sales, Development, Analytics, Communication, Utility [PLUGIN `src/rules/valid-node-categories.ts`; CODEX].
  - The starter's own `Example.node.json` uses `"Developer Tools"`, which this rule rejects [STARTER]. Do not copy it.
- [ ] **Programmatic code:**
  - every returned item carries `pairedItem` [PLUGIN `missing-paired-item`];
  - caught errors are re-thrown as `NodeApiError`/`NodeOperationError` [PLUGIN `require-node-api-error`];
  - use `httpRequestWithAuthentication` instead of `httpRequest` plus manual `getCredentials` [PLUGIN `no-http-request-with-manual-auth`];
  - no deprecated `n8n-workflow` APIs [PLUGIN `no-deprecated-workflow-functions`].
- [ ] **Trigger naming:** a class ending in `Trigger` has a `name` ending in `Trigger`, a `displayName` containing "Trigger", and `inputs: []` [PLUGIN `trigger-node-conventions`].
- [ ] **Webhook triggers implement `webhookMethods.default.{checkExists, create, delete}`** [PLUGIN `webhook-lifecycle-complete`]. None of those methods may swallow errors: no empty catch, and no catch whose only statement is a `return` [PLUGIN `no-silent-error-swallowing`]. Polling triggers are out of scope for that rule.

### Scope and policy (reviewer-judged)

- [ ] **Not an existing node.** n8n core has no Plunk node; only Splunk exists [CORE tree search], and the verified catalog has no `plunk` package [CAT]. If a node iterates on an existing node, open a PR to n8n instead [VG].
- [ ] **One third-party service per package.** A trigger node for the same service may ship alongside the action node [VG].
- [ ] **No logic or flow-control nodes** [VG].
- [ ] **No competition with n8n paid features**: n8n "reserves the right to reject nodes that compete with any of n8n's paid features, especially enterprise functionality" [SUB].
- [ ] **UX guidelines followed**: "Your node's UI must conform to these guidelines to be a verified community node candidate" [UX]. See section 3.

---

## 2. Recommendations (not automatically enforced, but expected)

- **Scaffold with `npm create @n8n/node@latest`.** Docs say this is "strongly suggest[ed]" for verification but "isn't strictly required" [SUB; VG "All verified community node authors should use the n8n-node tool"].
- **Keep `"strict": true` in `package.json` `n8n` and leave `eslint.config.mjs` exactly as scaffolded:**

  ```js
  import { config } from '@n8n/node-cli/eslint';

  export default config;
  ```

  - In strict mode, `n8n-node lint` exits with an error if `eslint.config.mjs` differs from this default (whitespace-normalised comparison) [CLI `commands/lint.ts` `verifyEslintConfig`].
  - `n8n-node cloud-support` reports "eligible for n8n Cloud verification" only when strict is on **and** the default config is in place [CLI `commands/cloud-support.ts`].
  - `cloud-support disable` switches to `configWithoutCloudSupport`, which drops `no-restricted-imports` and `no-restricted-globals`, and warns that the node becomes ineligible [CLI `cloud-support.ts`; PLUGIN `plugin.ts`].
  - The scanner never reads `strict`; it applies its own config [SCAN]. So `strict` is a local guard, not a server-side gate. I could not verify whether reviewers check the flag by hand.
- **Fix warning-level rules too.** They don't fail the scan, but reviewers see them:
  - `icon-prefer-themed-variants`: provide a `{ light, dark }` icon object; both may point to the same file;
  - `require-files-array`, `require-homepage`, `node-registration-complete`, `no-dead-files`, `open-ended-node-fields`, `operation-name-repeats-resource`, `operation-standard-action` ("Get many", "Create or update") [PLUGIN `plugin.ts`].
- **Prefer the declarative style for the action node.**
  - The docs call it "the default for new nodes". The `HTTP API` choice in the scaffolder is "designed for faster approval for n8n Cloud" [STYLE; CLI-DOC].
  - Use the programmatic style only for triggers, non-REST APIs, nodes that transform items beyond `preSend`/`postReceive`, or feature-based versioning [STYLE; CODE-STD].
  - **Triggers must be programmatic.** Mixing styles in one package is explicitly fine for verification [STYLE; VG].
- **Use the built-in HTTP helpers** (`this.helpers.httpRequest(WithAuthentication)`), not service SDKs [CODE-STD].
- **Declare `usableAsTool: true`** unless the node is binary-heavy [CLI template `.agents/nodes.md`]. For AI tool UX, see the UX "Output" parameter in section 3.
- **Credential `test`** is "strongly recommended" in the scaffold notes and is an error in lint (see section 1) [CLI template `.agents/credentials.md`; PLUGIN].
  - The declarative `test.request.baseURL` can use an expression such as `'={{$credentials?.domain}}'` [CRED-DOC], so a configurable base URL (Plunk Cloud or self-hosted) works.
- **Codex file** `<Node>.node.json` carries `resources.primaryDocumentation[].url` and `resources.credentialDocumentation[].url`, which can point at README anchors on GitHub [STARTER `nodes/Example/Example.node.json`; CODEX].
- **Don't set `defaults.color`.** It is deprecated and removed in n8n 3.0 [STDPARAM; V3; ICONS].
- **Write a CHANGELOG** and keep GitHub `main`, the git tag and the npm version identical. A reviewer flagged "Version mismatch between GitHub and npm" as a manual-review finding [FORUM 325906, user report]. The scaffolded release flow maintains CHANGELOG via `auto-changelog` [CLI `release.ts`].

---

## 3. Node design guidance (UX guidelines, condensed) [UX]

**Credentials**
- Sensitive values are password fields.
- Include an OAuth credential whenever the service offers OAuth. Plunk uses API keys, so this likely doesn't apply.

**Operations**
- Aim for CRUD per resource: Create, Create or Update (upsert), Delete, Get, Get Many, Update.
- Use the standard vocabulary and description templates:
  - "Create a new <RESOURCE>"
  - "Retrieve a list of <RESOURCE>s"
  - "Delete a <RESOURCE> permanently"
  - …

**Resource Locator**
- Use it whenever a single item is selected, with `From list` as the default mode.

**Sorting**
- Put sort options in a dedicated collection below "Options".

**Delete output**
- Return `[{ "deleted": true }]`.

**Simplify**
- If a response has more than 10 fields, add a boolean `Simplify` returning at most 10 fields, flattened, most-used first. Use this description: "Whether to return a simplified version of the response instead of the raw data".

**AI tool nodes**
- If a response has more than 10 fields, add an `Output` option with three modes:
  - Simplified;
  - Raw;
  - Selected fields (multi-select, always including the ID).
- Reason: tools run out of context and get confused by large payloads.

**Copy**
- Title Case for node name, parameter labels and dropdown titles.
- Sentence case for actions, descriptions, tooltips, hints and dropdown descriptions.
- Operation `name`: Title Case, without repeating the resource ("Delete").
- `action`: sentence case, includes the resource, no articles ("Delete contact").
- `description`: sentence case, adds information.
- Use the service's UI terminology, not its API terminology, and avoid tech jargon ("field", not "key").
- Placeholders start with `e.g.` (for example `e.g. nathan@example.com`).
- Boolean descriptions start with "Whether…".
- Refer to parameters in single quotes.

**Errors**
- The message says what happened, includes the parameter `displayName` and `[item X]`, and avoids the words "error", "problem", "failure" and "mistake".
- The description says how to fix it.

**Scaffolded conventions** [CLI template `.agents/nodes.md`]
- Get Many has "Return All" plus "Limit".
- Return array items as separate n8n items rather than a wrapper object.
- Use camelCase property names.
- Mark required fields `required: true`.
- Reuse internal parameter names across operations so values survive a switch [CODE-STD].

---

## 4. Trigger nodes

- **Types**: webhook, polling, and "others" (queues or time-based) [TYPES]. Triggers must be programmatic [STYLE].
- **Webhook triggers (verification-relevant):**
  - full `checkExists`/`create`/`delete` lifecycle [PLUGIN `webhook-lifecycle-complete`];
  - no swallowed errors in those methods [PLUGIN `no-silent-error-swallowing`];
  - trigger naming conventions and `inputs: []` [PLUGIN `trigger-node-conventions`];
  - no `usableAsTool: true` [PLUGIN `node-usable-as-tool`].
- **Polling triggers**: no polling-specific verification rule found. They still fall under `no-restricted-globals` (no `setInterval`/`setTimeout`) and the other node rules [PLUGIN]. n8n drives the poll schedule, so the node never needs timers.
- **Webhook security** (for example signature verification): no verification rule found. Reviewers may still expect it. Not verified.

---

## 5. Runtime dependencies, bundling and zod

- **Rule:** `dependencies` must be empty [VG; SUB; PLUGIN `no-runtime-dependencies`].
  - The rule's own docs say shared libraries must come from `peerDependencies` "or bundled at build time into the published artifact" [PLUGIN `no-runtime-dependencies.md`, `no-restricted-imports.ts` comment].
  - But the verification text is stricter: "does **not** include any external dependencies" and "aren't allowed to use any run-time dependencies" [VG; SUB].
- **Bundling is not supported by the official toolchain, and the scanner would catch it.**
  - `n8n-node build` runs plain `tsc` and copies `**/*.{png,svg}` and `__schema__` JSON. It has no bundler [CLI `commands/build.ts`].
  - The scanner lints every `.js` file in the published tarball [SCAN], so a bundled library would be linted as our code. Most libraries touch `process`, `setTimeout`, `console` and similar, which would fail.
  - **Conclusion: no bundling; write the HTTP layer against `this.helpers`.**
- **zod:**
  - `zod` is on the Cloud runtime import allowlist [PLUGIN `no-restricted-imports.ts`], and n8n itself uses zod [CORE `community-packages` DTOs].
  - Importing zod from shipped code would bind us to whatever zod version the host ships, which is undeclared and unversioned. Avoid this.
  - The plan to keep **zod as a devDependency used only in tests** is compatible with every rule.
- **Footgun:** `no-restricted-imports` **exempts anything listed in `devDependencies`** [PLUGIN `no-restricted-imports.ts`].
  - So a stray `import { z } from 'zod'` (or any devDep) inside `nodes/` passes lint and scan, then fails at runtime on instances where it isn't installed.
  - We need our own guard, for example a fallow or ESLint boundary rule: "no devDependency imports from `nodes/**` or `credentials/**`".
- **Test file placement:**
  - The scanner lints `{nodes,credentials}/**/*.{js,ts,json}` in the attested source [SCAN `SOURCE_FILE_PATTERNS`]. The template `tsconfig` compiles `nodes/**` and `credentials/**` into `dist/`, which is then tarball-scanned [CLI template `tsconfig.json`; SCAN].
  - Tests under `nodes/` would therefore be linted with Cloud rules (no `setTimeout`, no `console`, and so on) and could ship in `dist/`.
  - **Put tests in a top-level `test/` folder**, outside `nodes/` and `credentials/`, with their own tsconfig. Also, `test.js` is a flagged dead-file name inside those folders [PLUGIN `no-dead-files`].

---

## 6. Tooling summary

### `@n8n/node-cli` (bin `n8n-node`) [CLI README; CLI-DOC; CLI `src/commands/*`]

| Command | What it does | Notes |
|---|---|---|
| `npm create @n8n/node@latest [name] -- --template …` / `n8n-node new` | Scaffold | Templates: `declarative/github-issues`, `declarative/custom` (asks for base URL and auth type: API Key, Bearer, OAuth2, Basic, Custom, None), `programmatic/example`. The code also has AI templates (`programmatic/ai/*`). |
| `n8n-node dev` | Builds in watch mode and runs n8n with the node linked | **2.x CLI**: `npx n8n@latest` in a subprocess, data in `~/.n8n-node-cli`. **n8n 3.0 / CLI 1.x**: runs `docker.n8n.io/n8nio/n8n:latest` in Docker or Podman, with `--n8n-image` to pin a version and a per-image data volume. `--external-n8n` attaches to your own instance started with `N8N_DEV_RELOAD=true` [CLI `dev/index.ts` on master vs `3.x`; V3]. The docs already describe the 3.0 behaviour; the code on master still uses `npx`. |
| `n8n-node build` | `tsc` into `dist/` and copies icons and `__schema__` | No bundling. |
| `n8n-node lint [--fix]` | ESLint with `@n8n/node-cli/eslint` config | In strict mode it fails if `eslint.config.mjs` isn't the default. The config combines `@eslint/js` recommended, `typescript-eslint` recommended, `@n8n/eslint-plugin-community-nodes` recommended, `eslint-plugin-import-x`, the legacy `eslint-plugin-n8n-nodes-base` rulesets (community/credentials/nodes), and `no-console: error` [CLI `configs/eslint.ts`]. |
| `n8n-node cloud-support [enable\|disable]` | Toggles strict mode and the default lint config | See section 2. |
| `n8n-node release` | **Locally**: release-it lints, builds, bumps the version, writes the changelog (`auto-changelog`), commits, tags, pushes and creates a GitHub release, but does **not** publish to npm. **In CI** (`GITHUB_ACTIONS` set): lint, build, then `npm publish` with `NPM_CONFIG_PROVENANCE=true` [CLI `release.ts`]. | release-it is invoked with `--git.requireBranch=main --git.requireCleanWorkingDir --git.requireUpstream`. `--publish` (local publish) gives no provenance, so the package can't be verified. `--init-workflow` writes `.github/workflows/publish.yml`. |
| `n8n-node prerelease` (hidden) | `prepublishOnly` guard | Blocks a bare `npm publish` unless `RELEASE_MODE` is set [CLI `prerelease.ts`]. |

### Lint

- **`@n8n/eslint-plugin-community-nodes` (namespace `@n8n/community-nodes`)** is the current linter. It has two configs, `recommended` (Cloud) and `recommendedWithoutN8nCloudSupport`, and about 45 rules [PLUGIN `plugin.ts`, README; LINT-DOC].
- **`eslint-plugin-n8n-nodes-base`** (npm 2.0.0) is still wired in for its `community`, `credentials` and `nodes` rulesets, both by the CLI config and by the scanner [CLI `configs/eslint.ts`; SCAN `scanner.ts`]. It is no longer the primary documented linter [LINT-DOC].
- **The linter docs say "Don't edit" `eslint.config.mjs`** [LINT-DOC].

### Scanner: `npx @n8n/scan-community-package <pkg>[@version|@tag]` [SCAN]

Steps, in order:
1. npm provenance (SLSA v1).
2. Fetch the attested GitHub commit (the repo must be public).
3. ESLint on the source (`package.json`, `nodes/`, `credentials/`).
4. ESLint on the tarball (`.js` and `package.json`).

Exit codes: 0 = pass, 1 = could not complete, 2 = failed a check. The scanner fixes its own config and ignores inline disables. It can only run **after** an npm publish, so it isn't a pre-publish CI gate. Run `n8n-node lint` before publishing, and the scanner against a published prerelease or `next` tag.

### `n8n-nodes-starter` [STARTER]

- Still maintained (last commit 2026-08-26, CLA bot added). It now essentially mirrors the CLI's `declarative/github-issues` template plus a programmatic `Example`.
- The README points newcomers to `npm create @n8n/node`.
- It ships `publish.yml`, `ci.yml` (lint plus build on PR and `main` push) and `README_TEMPLATE.md`.

### Scaffolded GitHub workflows [CLI `templates/shared/default/.github/workflows/*`]

**`publish.yml`**
- Triggers on tags `'*.*.*'`. Change it to `'v*.*.*'` if you use a `v` prefix.
- `permissions: { id-token: write, contents: read }`.
- `actions/setup-node@v4` with `node-version: lts/*`, then install and `npm run release`.
- `NPM_TOKEN` is optional (fallback). With trusted publishing it stays unset.

**`ci.yml`**
- Lint and build.

---

## 7. npm publishing: provenance and trusted publishing

- **Deadline: 2026-05-01.** Since that date, every verification submission must be published from GitHub Actions with provenance [SUB; VG; CLI].
  - The VG banner says "you must publish **ALL** community nodes using a GitHub action and include a provenance statement". The SUB page scopes this to "nodes submitted for verification".
  - The n8n runtime does **not** check provenance when installing unverified packages. I found no provenance code in `packages/cli/src/modules/community-packages` [CORE]. In practice the requirement bites at verification and scanning.
- **Trusted publishing (recommended).** One-time setup on npmjs.com: package settings → Publish access → Trusted Publishers → GitHub Actions, with owner `krautnerds`, repo `n8n-nodes-plunk`, workflow `publish.yml` (the filename) [SUB; CLI `publish.yml`]. npm requirements [NPM-TP]:
  - npm CLI ≥ 11.5.1 and Node ≥ 22.14.0;
  - GitHub-hosted runners only (self-hosted not supported);
  - public repo and public package;
  - `repository.url` must exactly match the GitHub repo;
  - provenance is generated automatically;
  - a new trusted-publisher configuration "must complete its first successful publish within 2 days".
- **Fallback**: a Granular Access Token stored as the `NPM_TOKEN` Actions secret. Provenance still comes from GitHub OIDC (`id-token: write`) [SUB; CLI `publish.yml`].

---

## 8. Submission and review process

1. Meet sections 1 and 3, publish via Actions with provenance, and check with `npx @n8n/scan-community-package n8n-nodes-plunk`.
2. Sign up or log in at the **n8n Creator Portal**, https://creators.n8n.io/nodes, and submit [SUB]. The portal is login-gated. Its public page only says you can "get your community nodes verified".
3. **Stages** (from portal statuses in forum reports, not official docs):
   - "Automated Review", which has been known to get stuck "in progress" [FORUM 275867];
   - manual review with feedback such as "Update Changes Required", or "Version mismatch between GitHub and npm" [FORUM 302391, 325906];
   - "Resubmit package", which needs a confirmation checkbox [FORUM 325906].
   - A submitter offered to "record a new demo video", which suggests a demo video is part of the submission. Not verified.
4. **Timeline**: "Reviews can take up to four weeks. n8n emails you as your submission moves through review." This was added to the docs on 2026-10-29 [SUB; docs commit `89f22db`].
5. **Updates after verification:**
   - Updates go through review again: a portal status "Update Changes Required" exists for updates [FORUM 302391].
   - The portal won't let you swap the version while a submission is in review ("Package is already submitted for verification"). Users are told to email `nodes@n8n.io` or `help@n8n.io`, or use Discord `#community-nodes` [FORUM 290662, 275867; n8n moderator reply in FORUM 325906 points to `nodes@n8n.io`].
   - Technically, n8n installs verified packages at the **vetted** version with a pinned checksum from `api.n8n.io/api/community-nodes`, which keeps per-version `nodeVersions[{npmVersion, checksum}]` [CORE `community-node-types-utils.ts` `selectVettedVersion`; CAT]. Every new version therefore has to be vetted before verified users get it.
6. **Common rejection reasons** (compiled; no official list exists):
   - missing provenance or local publish [SUB; SCAN];
   - any scanner lint error [SCAN];
   - runtime dependencies [SUB];
   - non-MIT license or private/mismatched repo [VG];
   - GitHub/npm version mismatch [FORUM 325906];
   - package.json description or keyword content [FORUM 302391, user speculation];
   - multi-service or proxy packages, logic/flow nodes, or duplicates of existing nodes [VG];
   - competing with paid features [SUB];
   - UX-guideline violations [UX];
   - non-English content [VG].
7. **Scale**: the public catalog lists 2,278 verified packages, each with an `isOfficialNode` flag (vendor-built vs community) [CAT].

---

## 9. Self-hosted vs n8n Cloud

- **Verified nodes**: discoverable and installable from the nodes panel ("More from the community") "across all deployment types (self-hosted and n8n Cloud)". Only owners and admins install [VG; INST `install-verified-community-nodes.md`].
  - Cloud owners can turn this off in the Cloud console.
  - Self-hosted instances control it with `N8N_VERIFIED_PACKAGES_ENABLED` (default `true`) [INST; ENV; CORE config].
  - Submitting to the community node repository is "the only way to use custom nodes on Cloud" [docs `docs/connect/create-nodes/deploy-your-node/README.md`].
- **Unverified nodes**: self-hosted only.
  - Installed via Settings → Community Nodes (GUI, with a "risks" acknowledgement), manually via CLI, or via env vars (n8n ≥ 2.21.0: `N8N_COMMUNITY_PACKAGES_MANAGED_BY_ENV`, `N8N_COMMUNITY_PACKAGES`) [INST; ENV].
  - With env-var management, an unvetted package needs a pinned `checksum` or `N8N_UNVERIFIED_PACKAGES_ENABLED=true`, otherwise n8n fails to start [INST `environment-variable-installation.md`].
- **n8n 3.0 changes the picture**: `N8N_UNVERIFIED_PACKAGES_ENABLED` defaults to **`false`**. "After the upgrade, you can't install unverified community nodes from npm, and installed ones stop loading" [V3]. This is confirmed in source on the `3.x` branch (`unverifiedEnabled: boolean = false`) [CORE]. Without verification, self-hosted users on 3.0 must opt in explicitly. **This makes verification close to mandatory for adoption.**
- **Cloud runtime restrictions** are expressed as the Cloud-only lint rules (restricted imports and globals) [PLUGIN; CLI `lint.ts` `containsCloudOnlyErrors`].
- **Blocklist**: n8n can block malicious or harmful low-quality packages from installation [RISKS `blocklist.md`].
- **AI tool usage**: the previous self-hosted env flag for community-node tool usage no longer appears. It is absent from the nodes env-var docs [ENV] and from `CommunityPackagesConfig` [CORE]. n8n reads `usableAsTool` from the verified catalog's `nodeDescription` [CORE `community-node-types.service.ts`]. I did not test whether an unverified self-hosted install exposes the tool variant without any flag.

---

## 10. Implications for n8n-nodes-plunk (inputs for later tickets)

1. **Our own strict lint must live beside the n8n config, not in it.** `eslint.config.mjs` has to stay at the 3-line default (strict mode), and the scanner ignores our config and inline disables anyway.
   - Option: a second config (for example `eslint.strict.config.mjs`) run by a separate `lint:strict` script, plus fallow for architecture and dead code. See open question 1.
2. **Tests go in a top-level `test/` folder.**
   - zod and vitest stay devDependencies.
   - Add a boundary rule forbidding devDependency imports from `nodes/**` and `credentials/**` (section 5 footgun).
3. **No runtime deps and no bundling.** The HTTP layer goes through `this.helpers.httpRequestWithAuthentication`. Retry and backoff use `sleep` from `n8n-workflow`, not `setTimeout`.
4. **Action node: declarative, `usableAsTool: true`. Trigger: programmatic.** For webhooks, implement the full lifecycle with real error handling. For polling, use no timers.
5. **Credential `plunkApi`:**
   - secret key as a password field;
   - configurable base URL used in `test.request.baseURL` via expression;
   - full `documentationUrl` (README anchor);
   - credential `icon`.
6. **Icons:** SVG with `{ light, dark }` variants on both node and credential.
7. **Release flow conflicts with the "never commit on main" rule.** Local `n8n-node release` requires branch `main` and pushes a release commit and tag straight to it [CLI `release.ts`]. The CI half only needs a tag push.
   - Option: bump the version and changelog in a PR, merge, then push a tag, which triggers `publish.yml` → `npm run release` (CI mode). See open question 2.
   - The repo's default branch is currently `chore/wayfinder-map` (GitHub API) and needs to become `main`.
8. **Pin `n8nNodesApiVersion: 1`** and make the README ready for verified-only discovery on n8n 3.0.

---

## 11. Open questions / not verified

1. **Is a second, stricter ESLint config acceptable to reviewers?** Neither docs nor code forbid extra lint configs; only `eslint.config.mjs` is compared [CLI `lint.ts`]. Not verified with n8n.
2. **Can the release commit avoid `main`?** release-it's `--git.requireBranch=main` is hard-coded in `n8n-node release` (local mode). A custom tag-based flow (PR bump, then tag) is technically compatible with `publish.yml`, but this hasn't been tested. Also, does the GitHub-release step need a `GITHUB_TOKEN` locally?
3. **First publish with trusted publishing**: npm's docs don't say whether a trusted publisher can be configured before the package exists [NPM-TP]. The first publish may need a granular `NPM_TOKEN` (still via Actions, so provenance is kept). To check when setting up.
4. **Node/npm in `publish.yml`**: `lts/*` must provide npm ≥ 11.5.1 for trusted publishing [NPM-TP]. To verify on the runner, or pin `node-version: 24` plus `npm i -g npm@latest`.
5. **Creator Portal form fields** (demo video? test credentials? test account for reviewers?) are login-gated. A demo video is only hinted at [FORUM 325906]. Self-hosted-only testing could be an issue if reviewers need a reachable Plunk account. Since Plunk Cloud exists, reviewers can presumably use it.
6. **Do new versions of a verified node re-enter manual review, or get auto-vetted after scanning?** Re-vetting per version is technically required (checksum pinning), but the process and SLA aren't documented.
7. **`isOfficialNode`**: does n8n prefer, or require coordination with, the vendor (Plunk) for verification of a third-party-built node? Not documented.
8. **Is webhook signature verification expected by reviewers?** No rule or doc found.
9. **AI tool exposure for unverified installs** on current 2.x self-hosted (see section 9) was not tested.
10. **`n8n.strict`**: checked only by the CLI. Whether manual review requires it is unknown. Keeping it `true` is free.

## Third-party sources

None used. All claims come from n8n docs, n8n GitHub source, the npm registry and npm docs, n8n's public catalog API, and the official n8n community forum. Forum threads are user reports except where marked as a moderator reply, and are flagged inline.
