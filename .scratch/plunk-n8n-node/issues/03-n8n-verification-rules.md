# n8n community node and verification rules 2026

Type: research
Status: resolved
Blocked by: —
Map: [../map.md](../map.md)

## Question

What are the current (Nov 2026) rules and tooling for n8n community nodes and for **verified** status? Cover: runtime dependency restrictions, required tooling (`@n8n/node-cli`, scaffold, lint), npm provenance / GitHub Actions publishing requirements, package.json `n8n` block, `usableAsTool`, declarative vs. programmatic style guidance, credential test requirements, icon/branding rules, the submission/review process (Creator Portal?), and common rejection reasons. Primary sources only: docs.n8n.io, n8n-io GitHub, official n8n forum announcements.

## Answer

Full findings: branch `research/n8n-verification-rules`, file `claudedocs/research/03-n8n-verification-rules.md` (n8n docs, pinned n8n source, npm, official forum; seen 2026-11-10).

**Hard requirements**
- Package: name `n8n-nodes-*`, keyword `n8n-community-node-package`, `n8n` block with `n8nNodesApiVersion: 1` (n8n 2.x supports only 1; 3.0 up to 3), node/credential paths under `dist/`.
- `dependencies` empty; `peerDependencies` exactly `{"n8n-workflow":"*"}`; no install scripts, no `overrides`.
- MIT license, public GitHub repo matching npm, README, English only.
- Since 2026-05-01: publish from GitHub Actions with npm provenance (scaffolded `publish.yml` + npm trusted publishing, `@n8n/node-cli` ≥ 0.23).
- `npx @n8n/scan-community-package` must pass: checks provenance, downloads the attested commit, lints source **and** published `.js` with its own config, **ignores `eslint-disable`**.
- Code: Cloud import allowlist (zod is on it); no `process`/timer globals (use `sleep` from n8n-workflow), no env/fs, no `console`; every credential has a test; action nodes declare `usableAsTool`, triggers must not; icons via `file:` (light/dark recommended); webhook triggers implement real `checkExists`/`create`/`delete` without swallowing errors; **triggers programmatic, action node declarative**.
- Policy: one service per package, no duplicate of an existing node (none for Plunk), no competing with n8n paid features, follow UX guidelines.

**Plan-shaping findings**
- In strict mode `eslint.config.mjs` must stay the 3-line default, so our strict rules need a second config/script.
- Tests must live in top-level `test/`; anything under `nodes/`/`credentials/` is scanned with Cloud rules and compiled into `dist/`.
- zod as devDependency is allowed, but the import rule exempts all devDependencies: a stray zod import in node code would pass lint and crash at runtime. We need our own boundary check (fallow/ESLint).
- No bundling: `n8n-node build` is plain `tsc`, and the scanner lints the tarball.
- **n8n 3.0** (unreleased) makes unverified packages opt-in (`N8N_UNVERIFIED_PACKAGES_ENABLED` defaults to false), so verification is close to mandatory.
- The scaffolded `release` command requires branch `main` and pushes release commit + tag there, which clashes with the "never commit on main" rule.
- Submission via Creator Portal (creators.n8n.io): automated then manual review, up to 4 weeks; every version re-vetted and pinned by checksum. Verified nodes install from the panel on Cloud and self-hosted.

Open questions handed on:
- Second ESLint config acceptable to reviewers? → Engineering standards.
- Release via version-bump PR + tag push instead of local `release`; trusted publishing before first publish; npm ≥ 11.5.1 on `lts/*`; Creator Portal form contents; review of updates → Release and verification pipeline.
- Do reviewers expect webhook signature verification (Plunk has none, only a shared-secret header)? → Trigger design.
- Vendor preference (`isOfficialNode`) → Contact Plunk maintainers.
- AI tool variant on unverified 2.x installs: low impact, not pursued.
