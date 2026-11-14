# Map: n8n community node for Plunk

Label: wayfinder:map

## Destination

An implementation-ready **spec** for `n8n-nodes-plunk`: resources and operations, trigger design, credential/auth, node architecture, engineering standards, README/SEO plan, and the release + n8n verification pipeline. Building happens after this map, outside it (e.g. via `/to-issues`).

## Notes

- Domain: n8n community node (TypeScript, npm package) for the Plunk email platform (https://github.com/useplunk/plunk, default branch `next`, AGPL-3.0, self-hostable). Repo: https://github.com/krautnerds/n8n-nodes-plunk (public, MIT).
- Standing decisions from charting (2026-11-10):
  - Supports **Plunk Cloud and self-hosted** instances (configurable base URL).
  - Aim for **verified community node** status; respect its rules from day one.
  - Naming: npm/repo `n8n-nodes-plunk`; node "Plunk" (`plunk`); credential "Plunk API" (`plunkApi`); trigger node if warranted.
  - Triggers are **in scope** and should be comprehensive where Plunk allows.
  - Node is **usable as an AI Agent tool** (`usableAsTool`), e.g. sending transactional mail.
  - **Very strict engineering**: TypeScript strict, tests wherever sensible, strict lint rules that guide other contributors, **fallow** for architecture/dead-code hygiene. **zod only as devDependency** (contract tests), no runtime dependencies (verification).
  - All repo artifacts (issues, spec, README, code) in **English**.
  - Plunk API is understood by **reading Plunk's source** (apps/api, wiki, `@plunk/mcp`), not via `/reverse-engineer-anything` (REA is for shipped binaries, not source repos).
  - Test environment: **local self-hosted Plunk via Docker** (no cloud account for now).
- The originally pasted pre-research is unreliable: wrong service name ("UsePunk"), invented base URL, and 6 of 7 cited reference repos don't exist. Treat it as hypotheses only.
- Repo default branch is `main` (created 2026-11-14 from the first map commit, never committed to directly); work lands via PRs from feature branches. The scaffolded `n8n-node release` pushes to `main`, which clashes with that rule; see Release and verification pipeline.
- Skills for sessions: grilling + domain-modeling for grilling tickets; research for research tickets. Glossary: `/GLOSSARY.md`.
- Research findings live on throwaway `research/<slug>` branches under `claudedocs/research/`, never merged.

## Decisions so far

<!-- one line per closed ticket: [title](issues/NN-slug.md): gist -->

- [Plunk API surface](issues/01-plunk-api-surface.md): Bearer auth, `sk_` everywhere except `/v1/track` (`pk_` only); Cloud base `https://next-api.useplunk.com`, self-hosted at root; 3 pagination styles; two error shapes; unversioned resource routes; no webhook subscription API; Plunk schemas not reusable.
- [Plunk events for triggers](issues/02-plunk-events-for-triggers.md): only push path is a workflow WEBHOOK step (unsigned, no retry, SSRF-guarded), fully automatable via `sk_` `/workflows`; event set = email.*, contact.(un)subscribed, segment entry/exit, custom; polling fallback via `/events`, `/activity`, `/contacts`.
- [Reference repos and README/SEO patterns](issues/04-reference-repos-and-seo.md): model on `resend/n8n-nodes-resend` (+ Loops, AgentMail, IMAP for CI); display name must be exactly "Plunk"; npm search is plain text; Verified ≠ Official, and an n8n.io page realistically needs Plunk as maintainer; vendor wiki link is the best off-n8n lever.
- [n8n community node and verification rules 2026](issues/03-n8n-verification-rules.md): zero runtime deps, peer `n8n-workflow` only, provenance publishing from GitHub Actions, scanner lints source + tarball and ignores `eslint-disable`; action node declarative + `usableAsTool`, trigger programmatic; tests in top-level `test/`; strict lint needs a second config; n8n 3.0 makes verification near-mandatory.
- [Strict tooling compatibility with n8n node tooling](issues/05-tooling-compatibility.md): strict is feasible: n8n ESLint config untouched + second strict config (typescript-eslint pinned 8.70.0); TS 6 + node16, all strict flags except `verbatimModuleSyntax`/`isolatedDeclarations`; vitest + mock-extended + nock; own zod schemas (`zod@3.25.76`, `zod/v4`); fallow with explicit entries and transport boundary; lefthook.

## Not yet specified

- **AI Agent tool UX**: how operation/field descriptions should be worded so agents pick and fill them correctly; whether some operations should be hidden from tool use.
- **Example workflows / n8n templates**: which showcase workflows ship with the README or go to the n8n template gallery.
- **Error and pagination UX in detail**: how Plunk errors surface in n8n, "Return All"/limit behaviour, continue-on-fail semantics. Depends on the API research and the architecture decision.
- **API drift strategy**: resource routes are unversioned with docs/code drift; how the node pins and detects Plunk version changes (contract tests against a pinned Docker image, supported-version range).
- **Spec assembly**: the shape of the final spec document and who/what consumes it (likely `/to-issues`).

## Out of scope

- The implementation itself (this map ends at the spec).
- Plunk features that have no API surface.
- A separate MCP server: Plunk already ships `@plunk/mcp`.
