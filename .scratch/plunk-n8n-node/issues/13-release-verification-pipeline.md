# Release and verification pipeline

Type: grilling
Status: open
Blocked by: 03, 05
Map: [../map.md](../map.md)

## Question

How do we version, changelog, build, and publish (GitHub Actions + npm provenance), who owns the npm package, and what are the steps and timing for submitting to n8n verification?

## Notes from Engineering standards

- Conventional Commits are enforced (commit-msg hook + PR title check), squash merge only: pick the changelog/version tooling on top of that.
- `publish.yml` must run `npm run check` before `n8n-node release` (which only runs `lint` + `build` in CI). `lint` covers both ESLint layers.
- Post-publish `npx @n8n/scan-community-package` run belongs here, not in `ci.yml`.
- `release-it` is ignored in fallow (`ignoreDependencies`); Renovate groups `@n8n/node-cli` with `typescript-eslint`/`eslint`/`typescript`.
