# Strict tooling compatibility with n8n node tooling

Type: research
Status: open
Blocked by: —
Map: [../map.md](../map.md)

## Question

How far can we push strictness while staying compatible with `@n8n/node-cli` and verification? Cover: which TypeScript strict flags work with `n8n-workflow` types; layering strict ESLint (typescript-eslint strict-type-checked etc.) on top of the n8n lint config; how existing nodes are tested (Vitest/Jest, HTTP mocking via nock/msw, n8n's own test helpers, workflow-level tests); using zod as devDependency for contract tests, possibly against Plunk's `packages/types`; fallow configuration for this layout (architecture boundaries, dead code, duplication) and running it in CI; pre-commit hooks.
