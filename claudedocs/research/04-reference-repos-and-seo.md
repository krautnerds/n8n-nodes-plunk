# Research 04: Reference repos and README/SEO patterns

Ticket: `.scratch/plunk-n8n-node/issues/04-reference-repos-and-seo.md` (research, map: Plunk n8n node).
Researched: 2026-11-10. Every repo below was checked with `gh api repos/OWNER/NAME`, and every package with `npm view`. Verification status comes from n8n's own vetted-node API (see §2.1). Numbers are as of the research date.

## TL;DR

- **Main reference: `resend/n8n-nodes-resend`.** It is the closest analog (an email API with transactional mail, contacts, broadcasts and webhooks), and it is verified, vendor-maintained, publishes with provenance, has 15 vitest suites, and has a webhook trigger that registers its own webhook and checks signatures. Copy its layout, CI split, trigger design and README skeleton.
- **Closest analog for our situation: `fredfrom/n8n-nodes-loops`.** A third party (not the vendor) built a verified node for an email-marketing SaaS. That is exactly our position with Plunk.
- **Testing/CI reference: `umanamente/n8n-nodes-imap`.** It runs integration tests against a real server in Docker (Greenmail), a smoke matrix that installs the packed tarball into real n8n versions, coverage comments on PRs, and `npm audit signatures`. This maps directly onto "local self-hosted Plunk via Docker". It is **not** verified, because it has runtime dependencies.
- **Discovery, from primary sources:**
  1. **n8n nodes panel.** Verified nodes appear under "More from the community" when someone searches. The panel search matches `displayName` (weight 1.3) and `codex.alias` (weight 1).
  2. **npm search.** It is plain text relevance over name, description, README and keywords. The old quality/popularity/maintenance scores are fixed at 1.
  3. **n8n.io/integrations pages.** In practice only vendor-maintained ("official") verified nodes get one. A krautnerds-authored node most likely won't, unless Plunk adopts it.
  4. **Vendor docs.** Resend's own docs and changelog link its node. A guide in Plunk's wiki would be our best equivalent.
- No `n8n-nodes-plunk` package exists on npm (404), and there is no Plunk node on GitHub, npm or the n8n forum. The name is free and there is no competition for "Plunk n8n".
- **Fabricated pre-research repos confirmed nonexistent (404):** `n8n-io/n8n-nodes-telegram`, `-gotify`, `-aws`, `-openai`, `-http-request`, `-mcp`.

---

## 1. Reference repos

### 1.1 Table (verified facts)

| # | Repo | Stars | Last push | License | npm package (version) | npm downloads (30 d) | n8n status | Why it's here |
|---|---|---|---|---|---|---|---|---|
| 1 | [resend/n8n-nodes-resend](https://github.com/resend/n8n-nodes-resend) | 33 | 2026-11-05 | MIT | `n8n-nodes-resend` (3.2.1) | 13,866 | **Verified + Official** (vendor: Resend) | Closest API analog; best-in-class structure, tests, trigger |
| 2 | [fredfrom/n8n-nodes-loops](https://github.com/fredfrom/n8n-nodes-loops) | 0 | 2026-03-28 | MIT | `n8n-nodes-loops` (0.1.7) | 78 | **Verified** (third-party) | Same situation as ours: third-party verified node for an email-marketing SaaS, with a trigger |
| 3 | [josephfeleke/n8n-nodes-agentmail](https://github.com/josephfeleke/n8n-nodes-agentmail) | 9 | 2026-09-08 | MIT | `n8n-nodes-agentmail` (1.0.3) | 260 | **Verified + Official** (AgentMail) | Clean tests split by concern, including trigger lifecycle and webhook tests |
| 4 | [Mailhooks/n8n-nodes-mailhooks](https://github.com/Mailhooks/n8n-nodes-mailhooks) | 0 | 2026-05-05 | MIT | `n8n-nodes-mailhooks` (1.0.3) | 204 | **Verified** (not flagged official) | Webhook trigger **and** polling trigger in one package; co-located tests |
| 5 | [umanamente/n8n-nodes-imap](https://github.com/umanamente/n8n-nodes-imap) | 166 | 2026-10-26 | MIT | `n8n-nodes-imap` (3.0.4) | 6,431 | **Not verified** (has runtime deps) | Strongest testing/CI of any email node found |
| 6 | [n8n-io/n8n-nodes-starter](https://github.com/n8n-io/n8n-nodes-starter) | 1,190 | 2026-09-13 | MIT | (template) | n/a | n/a (n8n's official starter) | Baseline package.json, CI, publish.yml and README_TEMPLATE |
| 7 | [mailtrap/mailtrap-n8n](https://github.com/mailtrap/mailtrap-n8n) (npm still points to the old `railsware/mailtrap-n8n` URL, which redirects here) | 4 | 2026-07-17 | MIT | `n8n-nodes-mailtrap` (1.1.5) | 446 | **Verified + Official** (Mailtrap) | **Counter-example:** legacy gulp/.eslintrc tooling, no tests, **no npm provenance** |

Also checked but not used as references:
- [nerding-io/n8n-nodes-mcp](https://github.com/nerding-io/n8n-nodes-mcp): 3,039 stars, the most-starred community node, 569k downloads/30 d. It is not verified (runtime deps such as `@modelcontextprotocol/sdk` and `zod`). It shows that popularity doesn't require verification, but it's not a structural model.
- `@elevenlabs/n8n-nodes-elevenlabs` ([elevenlabs/elevenlabs-n8n](https://github.com/elevenlabs/elevenlabs-n8n), 32 stars, Verified + Official): useful only as a keyword-stuffing example (18 keywords such as "elevenlabs n8n node", "voice agents n8n node").
- [LucasSovre/listmonk-n8n](https://github.com/LucasSovre/listmonk-n8n) (26 stars, `NOASSERTION` license, not verified): a self-hosted newsletter tool similar to Plunk. Its license isn't MIT, so it's not a verification model.
- **Brevo, Mailjet, Postmark, Lemlist, MailerLite, Mailgun and SendGrid** all have *built-in* n8n nodes (in `n8n-io/n8n` `packages/nodes-base`). None has a notable community node repo, and no Brevo/MailerLite community package appears in npm search or in the verified list. n8n's docs say to use its built-in nodes as code-pattern examples ([submit docs](https://docs.n8n.io/connect/create-nodes/deploy-your-node/submit-community-nodes)).

"Verified" and "Official" are defined in §2.1. Stars, push dates and licenses come from `gh api repos/...`. Downloads come from `api.npmjs.org/downloads/point/last-month/<pkg>` (window 2026-10-11 to 2026-11-09). Versions, keywords and `n8n` blocks come from `npm view`.

### 1.2 Per-repo takeaways

#### 1) resend/n8n-nodes-resend (primary model)

**Layout**
```
.github/{ISSUE_TEMPLATE/, workflows/{build,lint,publish}.yml, dependabot.yml, media/header.png}
credentials/{ResendApi, ResendOAuth2Api, ResendWebhookSigningSecretApi}.credentials.ts
nodes/Resend/{Resend.node.ts, Resend.node.json, ResendTrigger.node.ts,
              actions/<resource>/…, methods/, transport/, utils/, resend-icon-{black,white}.svg}
examples/0N-*.json + examples/README.md      # importable example workflows
tests/*.test.ts + tests/helpers/              # 15 vitest suites
biome.json, eslint.config.mjs, tsdown.config.ts, vitest.config.ts, CHANGELOG.md, CODE_OF_CONDUCT.md, LICENSE.md
```

**package.json**
- Name `n8n-nodes-resend`.
- Description: "Resend integration for n8n: Email, Broadcasts, Templates, Domains, Contacts, Segments, Topics, Webhooks, Automations, Events, and more". It names the service, then the platform, then the resources.
- Keywords (10): `n8n-community-node-package, resend, email, webhook, n8n, automation, integration, broadcast, template, contacts`.
- `n8n`: `{ n8nNodesApiVersion: 1, credentials: [3], nodes: [Resend, ResendTrigger] }`.
- `files: ["dist"]`, `peerDependencies: { "n8n-workflow": "*" }`, **no `dependencies`**.
- `engines.node >=22.13`, pnpm.
- devDeps: `@n8n/node-cli`, biome, tsdown, vitest, typescript, auto-changelog.
- Scripts: `lint` (biome), **`lint:n8n` (`n8n-node lint`)**, `typecheck`, `test`, and `prepublishOnly = build && lint && test`.

**Codex** (`Resend.node.json`): `categories: ["Communication"]`, `primaryDocumentation` set to resend.com/docs, and `alias: ["email","human","form","wait","hitl","approval","resend"]`.

**CI**
- `build.yml`: build.
- `lint.yml`: biome, `n8n-node lint`, `tsc --noEmit`, vitest. This runs on push to main and on PRs.
- `publish.yml`: runs on a GitHub Release. It uses `id-token: write`, a GitHub Environment (`npm-publish`), `pnpm publish --provenance`, and actions pinned by SHA.
- npm shows SLSA provenance (`dist.attestations` = slsa.dev/provenance/v1).

**Tests**: tests for each concern:
- `nodeStructure`, `credentials`, `router`, `transport` and `operationRequests`: a table-driven test of method, endpoint and body for each operation.
- `apiSync` and **`drift`**: these assert the exact request payloads, so API changes are caught.
- `trigger`, `webhook`, `methods`, `dynamicFields` and `sendAndWait`.

**Trigger** (`ResendTrigger.node.ts`):
- Full `webhookMethods.default.{checkExists, create, delete}`, so the trigger registers and removes the webhook through the API. The webhook id lives in `getWorkflowStaticData('node')`.
- Svix signature verification with `crypto.timingSafeEqual` and a timestamp-tolerance check. It needs no runtime dependency.
- There is a separate "Webhook Signing Secret" credential.

**README structure**
- Header image and badges: npm version, license, npm downloads/month, npm last-update, "n8n 2.7.0" (min version).
- Centered anchor nav.
- One-sentence pitch.
- API Coverage: a collapsible table of resource, endpoint, status and operations.
- Migration guide.
- Installation.
- Credentials: one subsection per credential.
- Human in the Loop (a feature section).
- Trigger Events.
- **Limitations**.
- Development.
- License.

**Copy**
- The `actions/<resource>` + `transport` + `methods` split.
- Table-driven request tests and "drift" tests.
- `lint:n8n` as a separate CI step.
- `prepublishOnly` gating.
- A GitHub Environment plus provenance.
- SHA-pinned actions and Dependabot.
- An `examples/` folder of importable workflows.
- The README's API Coverage table and Limitations section.
- Light and dark icons.

**Don't copy blindly**
- The `n8n` badge's version is hard-coded.
- The `author.email` is a personal address.

#### 2) fredfrom/n8n-nodes-loops (our situation)

**Layout**
- `credentials/{LoopsApi,LoopsWebhookApi}`.
- `nodes/Loops/{Loops.node.ts, Loops.node.json, LoopsTrigger.node.ts, GenericFunctions.ts, loops.svg}`.
- `test/unit/*.test.ts` (jest) plus `test/Loops_{Demo,Test}_Workflow.json`.
- A Dockerfile and `.nvmrc`.

**package.json**
- Description: "n8n community node for Loops (loops.so) — manage contacts, send events, and transactional emails".
- Keywords: `n8n-community-node-package, loops, email, marketing, transactional`.
- `n8n.strict: true`.
- Scripts are straight from the `n8n-node` CLI: `build`, `dev`, `lint`, `release`. Tests use jest.

**CI**: a single `publish.yml` that runs on `*.*.*` tags and does build, lint, test, then `npm publish --provenance --access public`. Provenance is present on npm.

**Trigger**: webhook with `checkExists`, Svix-style HMAC verification (`createHmac`), and a separate webhook-secret credential.

**README**
- Badges, including a custom "n8n reviewed & approved" badge.
- The template intro ("This is an n8n community node for [Loops]…").
- A screenshot.
- Anchor nav.
- Installation: GUI, Docker, and the npm name in bold.
- Operations: a table per node of resource, operation and description.
- Credentials.
- **Example Workflows**: named, concrete use cases.
- **Rate Limits**.
- Compatibility.
- Resources.
- License.

**Copy**
- The README section set. It is close to n8n's README_TEMPLATE and adds Example Workflows and Rate Limits.
- The operations table format.
- Verification "proof" via a badge. That badge is self-made shields.io, not an official n8n asset, so mark it as an opinion.

#### 3) josephfeleke/n8n-nodes-agentmail (test organisation)

**Tests**: `AgentMail.{description,execute,loadOptions,pagination}.test.ts`, `AgentMailApi.credentials.test.ts`, `AgentMailTrigger.{description,lifecycle,webhook}.test.ts` and `helpers.ts` (jest).

**CI**: `ci.yml` (lint, test, build) and `publish.yml` (`n8n-node release`).

**Docs**: `docs/getting-started.md`, `docs/examples/`, `examples/*.json`.

**README**: Features (Actions / Trigger) → Installation → Getting Started → Example Workflows → **Node Reference** (one subsection per resource) → Trigger Output → Resources.

**Copy**: the test-file naming scheme (description / execute / loadOptions / pagination / trigger lifecycle / trigger webhook), and the "Trigger Output" README section, which documents payload shapes.

#### 4) Mailhooks/n8n-nodes-mailhooks (two trigger styles)

- It has three nodes: `Mailhooks`, `MailhooksTrigger` (webhook) and `MailhooksPollingTrigger`. Each lives in its own folder with a co-located `__tests__/`.
- `__mocks__/n8n-workflow.ts`, plus tsup and jest.
- **Copy (if Plunk webhooks are limited)**: a polling-trigger fallback next to a webhook trigger, documented as separate README subsections.

#### 5) umanamente/n8n-nodes-imap (testing/CI gold standard; not verifiable as-is)

**Tests**
- `test/UnitTests/` (21 files).
- `test/WithImapflowMock/` (mocked client).
- **`test/WithGreenmail/`**: integration tests against a real IMAP server in Docker, with `globalSetup`/`globalTeardown`, run via `jest --selectProjects WithGreenmail` (DEVELOPER.md).

**CI** (`test.yml`, `release.yml`, `test-npm-package.yml`, `coverage-comment.yml`, plus beta-branch sync)
- `npm audit signatures` (it verifies the provenance and registry signatures of dependencies) and `npm audit`.
- Coverage uploaded as an artifact. A separate `workflow_run` job then posts a coverage table on the PR and updates a badge.
- An **`n8n-smoke` matrix**: it packs the tarball and installs it into real `n8n@1.98.0`, `2.16.0` and `2.38.7`, plus a minimum-peer target, across Node 20, 22 and 24. It then loads the node metadata with `scripts/verify-consumer-package.js`.
- semantic-release publishes with provenance.

**Repo hygiene**: `AGENTS.md`, `CONTRIBUTING.md`, `DEVELOPER.md`, five issue templates, `.releaserc.json`.

**Why it can't be verified**: `dependencies: imapflow, mailparser, nodemailer`. Verified nodes "cannot use any run-time dependencies" ([verification guidelines](https://docs.n8n.io/connect/create-nodes/build-your-node/reference/verification-guidelines)).

**Copy**
- A Docker-backed integration test project. For us that means self-hosted Plunk in Docker, kept in a separate jest/vitest project from unit tests.
- The tarball-install smoke matrix against pinned n8n versions.
- `npm audit signatures`.
- PR coverage comments.
- AGENTS.md / CONTRIBUTING.md / DEVELOPER.md. These fit the map's "strict lint rules that guide other contributors".

#### 6) n8n-io/n8n-nodes-starter (baseline)

**package.json template**
- `keywords: ["n8n-community-node-package"]`.
- `n8n: { n8nNodesApiVersion: 1, strict: true, credentials, nodes }`.
- `files: ["dist"]`, `peerDependencies: { n8n-workflow: "*" }`.
- Scripts delegate to `n8n-node build|dev|lint|release` and `prepublishOnly: n8n-node prerelease`.

**CI**: `ci.yml` (lint and build on PR and main) and `publish.yml`, which runs on a semver tag. It has extensive comments on Trusted Publishing (OIDC) versus an `NPM_TOKEN` fallback, and it requires `@n8n/node-cli >= 0.23.0` for `--provenance`.

**`README_TEMPLATE.md`**: n8n's canonical README skeleton.
- Title `n8n-nodes-<name>`.
- "This is an n8n community node. It lets you use _X_ in your n8n workflows."
- One or two sentences on the service.
- The n8n fair-code line.
- A TOC.
- Installation (link to n8n's install guide).
- Operations.
- Credentials (prerequisites, auth methods, setup).
- Compatibility (minimum n8n version and tested versions).
- Usage (optional).
- Resources.
- Version history (optional).

#### 7) mailtrap/mailtrap-n8n (what not to do)

- It is verified and official, and it has an n8n.io page. But it uses gulp icons, legacy `.eslintrc`, has **no tests**, and its npm release has **no provenance**. Its `package.json` repository URL still points to the old `railsware/` org.
- The lesson: the vendor's brand got it an integration page, not its engineering. Don't model the repo on it.

---

## 2. How community nodes get discovered

### 2.1 n8n itself (the highest-intent channel)

**Verified list = `https://api.n8n.io/api/community-nodes`.**
- n8n's backend fetches "vetted node types" from this URL (production; staging is `api-staging.n8n.io`). See `packages/cli/src/modules/community-packages/community-node-types-utils.ts` in [n8n-io/n8n](https://github.com/n8n-io/n8n).
- On 2026-11-10 it listed **2,278 node entries from 1,672 packages**, all `isPublished: true`.

**Badge semantics** (from `CommunityNodeInfo.vue` and `@n8n/i18n` `en.json`)
- Any package in that list gets a **"Verified"** badge. The tooltip reads "This community node has been reviewed and approved by n8n".
- Packages installed from npm but not on the list show **"Via npm"**, with the tooltip "This community node was added via npm and has not been verified by n8n".
- `isOfficialNode: true` changes the tooltip and adds an "official" icon: "This is an official node maintained by {author}". In practice this means maintained by the vendor (Resend, Mailtrap, ElevenLabs …). Loops and Mailhooks are verified but `isOfficialNode: false`.
- For us: krautnerds publishing a Plunk node would be **Verified, not Official**, unless Plunk's team is the maintainer of record.

**Nodes panel search**
- Docs: users "Search for the node … If there is a matching verified community node, you will see a **More from the community** section at the bottom of the nodes panel" ([install verified nodes](https://docs.n8n.io/integrations/community-nodes/installation-and-management/install-verified-community-nodes)). This works on n8n Cloud and self-hosted ([verification guidelines](https://docs.n8n.io/connect/create-nodes/build-your-node/reference/verification-guidelines)).
- Matching uses `sublimeSearch` with `DEFAULT_KEYS = [{ properties.displayName, 1.3 }, { properties.codex.alias, 1 }]` (`packages/@n8n/utils/src/search/sublime-search.ts`).
- **Finding:** in the vetted API payload, `codex` is `{}` for almost every entry: only 33 of 2,278 have an `alias`, and Resend's entry has none even though its `.node.json` defines one. So **before install, panel search effectively matches on `displayName`**.
- Implications:
  - The display name must be exactly **"Plunk"**, so it's found by the obvious query.
  - The codex aliases (e.g. `email`, `transactional`, `newsletter`, `smtp`) still help **after** install, and possibly in the future.
  - This is inferred from the API payload and n8n source, not documented.

**n8n.io/integrations/<slug> pages (Google-indexed)**
- These exist for vendor-built verified nodes. The Resend page says it is "built and maintained by our partners at Resend and verified by n8n", with a "Created by Resend" label and the list of operations.
- **Sample test** (15 random `isOfficialNode: true` and 15 random `false` entries, slug = display name in kebab-case): 14 of 15 official nodes had a page (HTTP 200), and **0 of 15** non-official nodes did. `n8n.io/integrations/loops/` is 404.
- **Inference:** an n8n.io page (a strong SEO asset) is effectively reserved for vendor/partner nodes. `n8n.io/integrations/plunk/` is currently 404.

**Verification requirements that shape packaging** ([verification guidelines](https://docs.n8n.io/connect/create-nodes/build-your-node/reference/verification-guidelines), [submit](https://docs.n8n.io/connect/create-nodes/deploy-your-node/submit-community-nodes))
- The name must start with `n8n-nodes-` or `@scope/n8n-nodes-`.
- `n8n-community-node-package` must be in `keywords`.
- Nodes and credentials must be registered in the `n8n` attribute.
- **MIT license.**
- The npm repository URL must match the public GitHub repo, and the author/maintainer must match between npm and GitHub.
- **No runtime dependencies.**
- No environment variable or filesystem access.
- English only, including the README.
- "Proper documentation (README, usage examples, etc.)", "Include usage instructions, example workflows, and any necessary authentication details".
- `npx @n8n/scan-community-package <pkg>` must pass.
- **From 2026-05-01: publish from GitHub Actions with provenance.**
- One service per package. A trigger may be in the same package, but must be programmatic style.
- Review takes up to 4 weeks via the [Creator Portal](https://creators.n8n.io/nodes).

**The lint enforces most of this.** `@n8n/eslint-plugin-community-nodes`, used by `n8n-node lint`, has rules including:
- `package-name-convention`, `n8n-object-validation`, `valid-description`, `require-homepage`, `valid-author`, `require-files-array`, `no-runtime-dependencies`, `valid-peer-dependencies`, `no-forbidden-lifecycle-scripts`
- `valid-node-categories`, `node-usable-as-tool`, `webhook-lifecycle-complete`, `trigger-node-conventions`, `credential-test-required`, `require-node-api-error`
- `operation-name-repeats-resource`, `icon-prefer-themed-variants`
- `no-restricted-globals`, `no-restricted-imports`
- Source: the README in `packages/@n8n/eslint-plugin-community-nodes`.

So the package.json "SEO fields" (description, homepage, author) are **lint-required**, not optional.

**Codex categories.** These control placement in the panel. The allowed set is Data & Storage, Finance & Accounting, Marketing & Content, Productivity, Miscellaneous, Sales, Development, Analytics, Communication, Utility ([codex docs](https://docs.n8n.io/connect/create-nodes/build-your-node/reference/codex-files)). Resend uses `Communication`; n8n.io shows it under "Marketing, Communication".

### 2.2 npm search

- **Primary source** ([npm docs: searching for packages](https://docs.npmjs.com/searching-for-and-choosing-packages-to-download)):
  - Results are "displayed based on keyword matching" against the package's "title, description, readme, and keywords".
  - "No subjective ranking criteria are applied", except "a minimal boost to deprioritize spammy or entirely new packages".
  - New packages "may take up to two weeks to appear".
  - Users can re-sort by downloads, dependents or last publish date.
- **Registry API confirmation**: `registry.npmjs.org/-/v1/search` returns `score.detail = {popularity: 1, quality: 1, maintenance: 1}` for every result. The legacy composite scores are neutralised and `searchScore` is text relevance.
- `keywords:n8n-community-node-package` matches **14,340** packages, so the keyword is necessary (it's a requirement) but gives no ranking advantage.
- Query test: "resend n8n" ranks `resend` (SDK) first and **`n8n-nodes-resend` second**. Having both the service name and "n8n" in the package name and description is what wins.
- **Implications (inference from the docs):**
  - The package name `n8n-nodes-plunk` already contains both terms.
  - Put "Plunk", "n8n", "email", "transactional", "newsletter/campaigns", "contacts" and "self-hosted" in the **description and README text**, since both are indexed.
  - Keywords add the same terms. Keep them relevant: there's no documented penalty for 18 keywords like ElevenLabs, but no documented benefit either, and the "spammy" deprioritisation is opaque. *Opinion:* 8–12 focused keywords, Resend-style.
- [package.json docs](https://docs.npmjs.com/cli/configuring-npm/package-json#keywords): keywords help "people discover your package as it's listed in `npm search`". `homepage`, `repository` and `bugs` give the links on the npm page.
- npm shows provenance on the package page ([npm docs](https://docs.npmjs.com/searching-for-and-choosing-packages-to-download)). That is a trust signal, and it's required for verification anyway.

### 2.3 GitHub

- Topics: GitHub says topics exist "to help other people find and contribute to your project", and the limit is 20 per repo ([GitHub docs: classifying with topics](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/classifying-your-repository-with-topics)).
- Counts on 2026-11-10: `n8n` 14,448, `n8n-community-node` 2,242, `n8n-node` 2,067, `n8n-nodes` 1,063, `n8n-community-node-package` 810, `plunk` 14.
- Of the references, only Resend sets topics (`n8n, n8n-node, n8n-nodes, n8n-webhook, resend, …`). The IMAP node uses `n8n-community-node-package, n8n-nodes`.
- **Recommendation**: set topics `n8n`, `n8n-community-node-package`, `n8n-nodes`, `n8n-node`, `plunk`, `email`, `transactional-email`, `self-hosted`, plus a description and the npm URL as the homepage.
- Curated list: [restyler/awesome-n8n](https://github.com/restyler/awesome-n8n) (3,004 stars) ranks the "Top 100 community nodes" **by monthly npm downloads**, indexed by a crawler. You can't get in by asking; downloads come first.

### 2.4 Google and other channels (mostly opinion, flagged)

- **Observed**: searching "resend n8n node" returns the GitHub repo, **Resend's own changelog and docs pages** (`resend.com/changelog/official-n8n-node-for-resend`, `resend.com/docs/knowledge-base/n8n-integration`), and package mirrors (pkgstats, socket.dev, jsDelivr). Searching "plunk n8n" returns nothing relevant today: no competition.
- **Inference**: the vendor's docs are the strongest off-n8n channel. Plunk's wiki has a `guides/` section in `useplunk/plunk` `apps/wiki/content/docs/guides/` that already includes `mcp-server.mdx` and `webhooks.mdx`. An upstream PR adding an "n8n" guide would be the closest equivalent to Resend's setup. It also feeds the npm description and Google.
- **Opinion**: post a "Show & tell" in the n8n forum's [Built with n8n](https://community.n8n.io/c/built-with-n8n) category (1,987 topics). Submit example workflows to the n8n template gallery (that's a "not yet specified" map item). Mirrors (pkgstats, socket.dev) index automatically.
- Self-hosting angle (inference): Plunk is self-hostable and so is n8n. "Self-hosted" plus "Plunk Cloud" in the README and description targets a niche that neither Resend nor Loops covers.

### 2.5 What demonstrably helps (summary)

| Lever | Evidence level | Action for n8n-nodes-plunk |
|---|---|---|
| Being **verified** (shows in the n8n panel on Cloud and self-hosted) | Primary (n8n docs and source) | Meet all verification rules from day one (already a map decision) |
| `displayName` = "Plunk" | Primary (n8n search keys) + observed (empty codex in API) | Node "Plunk", trigger "Plunk Trigger" |
| Codex `alias` + `categories` | Primary (search keys, codex docs) | `alias: ["email","transactional","newsletter","campaign","contacts","smtp","plunk"]` (draft); `categories: ["Communication","Marketing & Content"]` |
| npm name, description, README text and keywords | Primary (npm docs) | Name and description include "Plunk" and "n8n"; README body names the features in plain words |
| `n8n-community-node-package` keyword | Primary (required) | Required, but gives no ranking edge (14k packages) |
| Provenance | Primary (required; shown on npm) | GitHub Actions + Trusted Publishing |
| n8n.io integration page | Observed (official-only pattern) | Only possible if Plunk becomes the maintainer of record (open question) |
| Vendor docs link | Observed (Resend) | PR an n8n guide to Plunk's wiki |
| GitHub topics | Primary (GitHub docs), weak | Set the topics above |
| Badges, banner, example workflows | Opinion / convention (all good repos have them) | Include; example workflows are also a verification requirement |

---

## 3. Proposed README outline for n8n-nodes-plunk

This is derived from n8n's `README_TEMPLATE.md` (the canonical skeleton) plus the strongest additions from Resend (coverage table, trigger events, limitations), Loops (example workflows, rate limits) and AgentMail (trigger output). It is English only (a verification rule).

```markdown
<!-- banner image: .github/media/header.png (Plunk × n8n), alt text includes "Plunk n8n node" -->
# n8n-nodes-plunk

Badges: npm version · npm downloads/month · license (MIT) · provenance/CI status · "n8n community node"
(optionally "n8n verified" once verified; tested-with n8n version, generated not hard-coded)

This is an n8n community node. It lets you use [Plunk](https://www.useplunk.com), the open-source email
platform, in your n8n workflows: send transactional email, manage contacts, run campaigns, and trigger
workflows from Plunk events. Works with **Plunk Cloud** and **self-hosted Plunk**.

[n8n](https://n8n.io/) is a fair-code licensed workflow automation platform.

Nav: Installation · Credentials · Operations · Trigger · AI Agent tool · Examples · Compatibility · Limitations · Development · Resources

## Installation
- n8n panel (verified): search "Plunk" → "More from the community" → Install
- Settings → Community Nodes → `n8n-nodes-plunk`
- Self-hosted / Docker / queue mode: link to n8n's manual install docs
## Credentials
### Plunk API
- Where to find the API key in Plunk (secret vs public key, if applicable)
- **Base URL**: Plunk Cloud default vs self-hosted instance URL
- Credential test behaviour
### (Plunk webhook signing secret, if the trigger needs one)
## Operations
Table per node: Resource | Operation | Description   (collapsible "API coverage" table with endpoint + status, Resend-style)
## Plunk Trigger
- Events table (event → when it fires)
- Webhook lifecycle: auto-registration vs manual setup; signature verification
- Trigger output: example payload per event (AgentMail-style "Trigger Output")
## Use as an AI Agent tool
- Which operations work well as tools (e.g. Send Email), how to describe fields to the agent
## Example workflows
- 3–5 named use cases, each with an importable JSON in `examples/` (e.g. sign-up → contact + welcome mail;
  form → transactional mail; bounce/complaint → update CRM; scheduled campaign)
## Compatibility
- Minimum n8n version, tested n8n versions (from the CI smoke matrix), Plunk version(s)/API notes, Node.js
## Limitations / Rate limits
- Known Plunk API gaps, self-hosted differences, rate limits, pagination notes
## Development
- Prerequisites, `npm run dev`, tests (unit vs Docker-backed Plunk integration), lint (`n8n-node lint`, fallow), link to CONTRIBUTING.md
## Resources
- n8n community nodes docs · Plunk docs/wiki · Plunk API reference · Plunk GitHub
## Version history / Changelog
- Link to CHANGELOG.md / GitHub Releases
## License
MIT
```

Repo-level companions: `CONTRIBUTING.md`, `DEVELOPER.md` (or a section in CONTRIBUTING), `AGENTS.md`, `CHANGELOG.md`, `CODE_OF_CONDUCT.md`, `LICENSE.md` (MIT), `.github/ISSUE_TEMPLATE/`, `examples/` with its own README.

Draft package.json discovery fields (for the spec; to be confirmed in later tickets):
```jsonc
{
  "name": "n8n-nodes-plunk",
  "description": "Plunk integration for n8n: send transactional email, manage contacts, run campaigns, and trigger workflows from Plunk events. Works with Plunk Cloud and self-hosted Plunk.",
  "keywords": ["n8n-community-node-package", "n8n", "plunk", "useplunk", "email", "transactional-email",
               "newsletter", "email-marketing", "contacts", "self-hosted"],
  "homepage": "https://github.com/krautnerds/n8n-nodes-plunk#readme",
  "repository": { "type": "git", "url": "git+https://github.com/krautnerds/n8n-nodes-plunk.git" },
  "bugs": { "url": "https://github.com/krautnerds/n8n-nodes-plunk/issues" },
  "license": "MIT",
  "files": ["dist"],
  "n8n": { "n8nNodesApiVersion": 1, "strict": true, "credentials": ["dist/credentials/PlunkApi.credentials.js"],
           "nodes": ["dist/nodes/Plunk/Plunk.node.js", "dist/nodes/Plunk/PlunkTrigger.node.js"] },
  "peerDependencies": { "n8n-workflow": "*" }
}
```

---

## 4. Open questions

1. **Maintainer of record.** Should we approach the Plunk team so the node becomes "Official" (an n8n.io integration page and the official icon), or co-maintained? Otherwise it will be Verified only, with no n8n.io page (based on the observed pattern).
2. **Upstream wiki PR.** Do we want to contribute an `n8n.mdx` guide to `useplunk/plunk` `apps/wiki/content/docs/guides/`? That repo is AGPL-3.0; docs contributions probably fall under it too. Check their contribution terms.
3. **Codex aliases and categories.** Confirm the final alias list, and whether `Marketing & Content` should come alongside `Communication`.
4. **Banner and badges.** Do we want a header image (Resend-style)? Check whether Plunk's logo/brand guidelines allow its use. A self-made "verified" badge should only be added after verification.
5. **Tooling choice.** Resend uses pnpm + biome + tsdown + vitest; the starter uses `n8n-node` CLI + eslint + prettier; IMAP uses jest + semantic-release. This belongs to the engineering-standards ticket. This research only notes that verified repos use all of these.
6. **Unconfirmed behaviour.** Whether n8n will ever use codex aliases for *uninstalled* verified nodes. Today's API payload says mostly no; this is not documented either way.

---

## 5. Citations

**n8n docs (primary)**
- Verification guidelines: https://docs.n8n.io/connect/create-nodes/build-your-node/reference/verification-guidelines (source: `n8n-io/n8n-docs` `docs/connect/create-nodes/build-your-node/reference/verification-guidelines.md`)
- Submit community nodes: https://docs.n8n.io/connect/create-nodes/deploy-your-node/submit-community-nodes
- Building community nodes: https://docs.n8n.io/integrations/community-nodes/building-community-nodes
- Codex files: https://docs.n8n.io/connect/create-nodes/build-your-node/reference/codex-files
- Install verified community nodes: https://docs.n8n.io/integrations/community-nodes/installation-and-management/install-verified-community-nodes
- Creator Portal: https://creators.n8n.io/nodes

**n8n source (primary), in [n8n-io/n8n](https://github.com/n8n-io/n8n)**
- `packages/cli/src/modules/community-packages/community-node-types-utils.ts`: the vetted-node API URLs.
- `packages/frontend/editor-ui/src/features/settings/communityNodes/components/nodeCreator/CommunityNodeInfo.vue` and `packages/frontend/@n8n/i18n/src/locales/en.json`: the Verified, "Via npm" and Official semantics.
- `packages/frontend/editor-ui/src/features/shared/nodeCreator/components/ItemTypes/NodeItem.vue`: the official icon.
- `packages/@n8n/utils/src/search/sublime-search.ts`: the panel search keys.
- `packages/@n8n/eslint-plugin-community-nodes/README.md`: the lint rules.

**n8n data (primary)**
- Vetted nodes API: https://api.n8n.io/api/community-nodes (fetched 2026-11-10, 3 pages × 1,000).
- Integration pages: https://n8n.io/integrations/resend/, https://n8n.io/integrations/mailtrap/ (200), https://n8n.io/integrations/loops/ and /plunk/ (404).
- Forum categories: https://community.n8n.io/categories.json

**npm (primary)**
- Search behaviour: https://docs.npmjs.com/searching-for-and-choosing-packages-to-download
- package.json keywords: https://docs.npmjs.com/cli/configuring-npm/package-json#keywords
- Provenance: https://docs.npmjs.com/generating-provenance-statements
- Registry search API: `https://registry.npmjs.org/-/v1/search?text=…`; downloads: `https://api.npmjs.org/downloads/point/last-month/<pkg>`
- `npm view n8n-nodes-plunk`: E404 (name free).

**GitHub (primary)**
- Topics: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/classifying-your-repository-with-topics

**Reference repos**
- https://github.com/resend/n8n-nodes-resend
- https://github.com/fredfrom/n8n-nodes-loops
- https://github.com/josephfeleke/n8n-nodes-agentmail
- https://github.com/Mailhooks/n8n-nodes-mailhooks
- https://github.com/umanamente/n8n-nodes-imap
- https://github.com/n8n-io/n8n-nodes-starter (incl. `README_TEMPLATE.md`, `.github/workflows/publish.yml`)
- https://github.com/mailtrap/mailtrap-n8n
- https://github.com/nerding-io/n8n-nodes-mcp
- https://github.com/elevenlabs/elevenlabs-n8n
- https://github.com/restyler/awesome-n8n

**Vendor channel (observed)**
- https://resend.com/changelog/official-n8n-node-for-resend
- https://resend.com/docs/knowledge-base/n8n-integration
- Plunk wiki guides: https://github.com/useplunk/plunk/tree/next/apps/wiki/content/docs/guides
