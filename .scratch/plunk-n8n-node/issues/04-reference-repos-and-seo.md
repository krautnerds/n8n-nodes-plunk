# Reference repos and README/SEO patterns

Type: research
Status: resolved
Blocked by: —
Map: [../map.md](../map.md)

## Question

Which **real, existing** community node repos are good references (well maintained, verified, popular; ideally email/marketing APIs similar to Plunk, e.g. Resend, Brevo, Mailjet-style nodes)? What do their repo layout, README structure, npm keywords and package metadata look like? How do community nodes get discovered (npm search, n8n integrations pages, Google), and what README/package choices measurably help? Verify every repo exists.

## Answer

Full findings: branch `research/reference-repos-and-seo`, file `claudedocs/research/04-reference-repos-and-seo.md` (includes a proposed README outline, draft package.json discovery fields, codex aliases/categories, GitHub topics).

- **Reference repos** (all checked to exist; verified status from api.n8n.io/api/community-nodes):
  - `resend/n8n-nodes-resend`: **primary model**. Verified, vendor-maintained; 15 vitest suites incl. request "drift" tests; build/lint/publish workflows with provenance; self-registering, signature-verified webhook trigger with zero runtime deps.
  - `fredfrom/n8n-nodes-loops`: our exact situation (verified, third-party, email marketing, trigger).
  - `josephfeleke/n8n-nodes-agentmail`: clean per-concern test split incl. trigger tests.
  - `Mailhooks/n8n-nodes-mailhooks`: webhook + polling trigger in one package.
  - `umanamente/n8n-nodes-imap`: testing/CI model (Docker mail server integration tests, packed package installed into several n8n versions in CI); not verified (runtime deps).
  - `n8n-io/n8n-nodes-starter`: baseline package.json, publish workflow, README template.
  - `mailtrap/mailtrap-n8n`: anti-example (verified but no tests, no provenance).
- **Badges**: anything on n8n's list is "Verified"; "Official" means vendor-maintained. krautnerds would be Verified, not Official.
- **n8n.io integration pages**: 14 of 15 sampled vendor-maintained verified nodes have one, 0 of 15 community-maintained. Without Plunk as maintainer we very likely get no page.
- **n8n panel search** matches the display name (+ codex aliases, almost never present pre-install), so the display name must be exactly **"Plunk"**.
- **npm search** is plain text match on name/description/README/keywords; quality scores are fixed. `n8n-community-node-package` keyword is required (14,340 packages) but gives no ranking edge.
- **Lint/verification** already enforce description/homepage/author and no runtime deps; verification also needs MIT, English, GitHub Actions + provenance (since 2026-05-01).
- **Off-n8n discovery**: the best lever is the vendor linking us, e.g. a PR adding an n8n guide to Plunk's wiki (`apps/wiki/content/docs/guides/`).
- Name `n8n-nodes-plunk` is free; no Plunk node exists anywhere.

Open questions handed on:
- Approach the Plunk team to maintain/co-maintain (→ "Official" + n8n.io page)? Contribute a wiki guide? → folded into "Ask Plunk maintainers about API stability" (now the general Plunk-maintainers contact ticket).
- Final aliases/categories, Plunk logo/brand usage → README, SEO and listing.
- Whether n8n will ever use aliases for uninstalled nodes: low impact, not pursued.
