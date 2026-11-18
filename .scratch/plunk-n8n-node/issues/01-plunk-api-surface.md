# Plunk API surface

Type: research
Status: resolved
Blocked by: —
Map: [../map.md](../map.md)

## Question

What is Plunk's public HTTP API as of the current `next` branch (v0.18.x)? List every endpoint usable by an integration (method, path, purpose, request/response shape), authentication (public vs. secret key, header format), pagination, error format, rate limits, idempotency, and API versioning. How do base URLs differ between Plunk Cloud and self-hosted instances? Sources: `apps/api`, `apps/wiki`, `packages/types`, and `@plunk/mcp` (which endpoints it uses, as a signal of what matters).

## Answer

Full findings: branch `research/plunk-api-surface`, file `claudedocs/research/01-plunk-api-surface.md` (Plunk `next` @ `37e5f81`, 2026-11-09).

- **Base URL**: Cloud is `https://next-api.useplunk.com` (old `api.useplunk.com` returns 410 Gone). Self-hosted uses its own host, routes at root, no `/api` prefix. → One free-form base URL, Cloud default (same as `@plunk/mcp`).
- **Auth**: `Authorization: Bearer <key>`, one key = one project. `pk_` works **only** on `POST /v1/track`; `sk_` works everywhere else and is rejected on `/v1/track` (docs claim otherwise; code wins).
- **Resources with `sk_`**: actions `/v1/send`, `/v1/track`, `/v1/verify`; contacts, templates, campaigns, segments, emails (scheduled transactional), events, workflows (steps/transitions/executions), domains, activity, analytics.
- **Dashboard-only**: `/auth/*`, OAuth, `/users/@me/*` (key rotation, billing), `/projects/*`, unsubscribe pages, SNS/Stripe webhooks.
- **Pagination**: three styles: cursor (`limit`/`cursor` → `{data,cursor,hasMore,total}`: contacts, activity, campaign recipients), page (`page`/`pageSize` → `{data,total,page,pageSize,totalPages}`: templates, campaigns, workflows, executions, segment contacts), none (segments, domains, events).
- **Errors**: standard `{success:false,error:{code,message,statusCode,requestId,errors[],details,suggestion}}` (422 for validation), but many handlers return bare `{error:"string"}`. Must parse both. `X-Request-ID` always set.
- **Rate limits**: per-project buckets on track/send/contact writes with `RateLimit-*` + `Retry-After`; off by default on self-hosted. The documented "1000/min" isn't in code.
- **Idempotency**: `Idempotency-Key` on `/v1/send` and `/v1/track` only; reuse → `409 IDEMPOTENCY_KEY_REUSED` (no replay), 24 h TTL.
- **Versioning**: only the three action paths have `/v1`; resource routes are unversioned, with no compatibility promise.
- **Webhooks**: no subscription API; outbound webhooks are an unsigned, non-retried WEBHOOK step inside a workflow. Feeds the trigger design ticket.
- **Contract-test schemas**: `@plunk/types` / `@plunk/shared` (zod v3) are private, not on npm, and AGPL. → Hand-written schemas verified against a pinned Docker image.
- **Docs vs code drift**: campaign responses wrapped in `{success,data}`; three documented analytics paths don't exist; bulk contact ops need a `mode` field; IDs are UUIDs, not `cnt_`.

Open questions handed on:
- Credential holds `sk_` required + `pk_` optional, or Track Event via `sk_` (`POST /contacts` + `POST /events/track`, non-atomic, no idempotency)? → Credential and auth design.
- How an API-key caller gets its project id (`POST /domains` requires it, value ignored). → Resource and operation catalogue.
- Is an auto-created webhook workflow an acceptable trigger UX? → Trigger design.
- Cloud rate-limit values; client-side caps on campaign `pageSize` / events `limit`; reliance on unversioned routes. → Not yet specified (error/pagination UX, API drift strategy).
- What `/projects/*` returns for an API key (inferred, untested). → Local Plunk test environment can confirm.
