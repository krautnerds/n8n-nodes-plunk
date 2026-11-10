# Research: Plunk API surface

Ticket: `.scratch/plunk-n8n-node/issues/01-plunk-api-surface.md`
Date: 2026-11-10

**Source of truth:** `useplunk/plunk`, branch `next`, commit `37e5f818e8087fc5947ecdc399b7f59efed0b342` (2026-11-09, after v0.18.0). File paths below are relative to that repo. Permalink prefix: `https://github.com/useplunk/plunk/blob/37e5f81/`. Line numbers are for that commit.

I read the source directly (`apps/api`, `apps/wiki`, `apps/mcp`, `packages/shared`, `packages/types`, `packages/db`, `docker/`). The only live calls were two unauthenticated probes of the Cloud hosts (see Base URL). Where the docs and the code disagree, **the code wins** and the gap is flagged with ⚠️.

---

## TL;DR

- **One HTTP API, one host, mostly no path prefix.** Express app with routes at the root: `/v1/send`, `/v1/track`, `/v1/verify`, and unversioned resource routes `/contacts`, `/templates`, `/campaigns`, `/segments`, `/workflows`, `/events`, `/emails`, `/domains`, `/activity`, `/analytics`. Only the three "action" endpoints carry `/v1`.
- **Base URL:** Cloud is `https://next-api.useplunk.com`. `https://api.useplunk.com` (Plunk v1) is **dead**: it answers `410 Gone`. Self-hosted is whatever the operator sets as `API_URI`/`API_DOMAIN`, normally a dedicated subdomain such as `https://api.example.com`, with routes at its root. The node needs one configurable base URL, defaulting to the Cloud host.
- **Auth:** `Authorization: Bearer <key>`. Every project has two keys: a **secret key `sk_…`** and a **public key `pk_…`**. `POST /v1/track` takes **only** `pk_`. **Every other integration endpoint** takes **only** `sk_`. The key is scoped to one project; the project comes from the key.
- **The resource routes are not dashboard-only.** Their middleware, `requireAuth`, accepts a secret key *or* the dashboard cookie. The real dashboard-only areas are `/auth/*`, `/users/@me/*`, `/projects/*`, OAuth, uploads, and the public unsubscribe pages.
- **Pagination comes in three styles:** cursor (`limit` + `cursor` → `{data, cursor, hasMore, total}`) on contacts, activity and campaign recipients. Page (`page` + `pageSize` → `{data, total, page, pageSize, totalPages}`) on templates, campaigns, workflows, workflow executions and segment contacts. **No pagination** at all on segments, domains, events and a few others.
- **Errors:** the global handler returns a consistent envelope `{success:false, error:{code, message, statusCode, requestId, errors?, details?, suggestion?}, timestamp}`. ⚠️ Many controllers short-circuit with a bare `{error: "..."}` instead, so the node must parse both shapes.
- **Rate limits:** per-project token buckets on `/v1/track`, `/v1/send` and contact writes, with `RateLimit-*` and `Retry-After` headers. **Off by default on self-hosted** (`RATE_LIMIT_ENABLED` is opt-in). Cloud status is unknown.
- **Idempotency:** an optional `Idempotency-Key` header on `/v1/send` and `/v1/track` only. A reused key gets `409 IDEMPOTENCY_KEY_REUSED`. It is a refusal, not a replay of the original response.
- **No API versioning** beyond the literal `/v1` on three paths. No version header, no OpenAPI for most routes.
- **No reusable published schemas.** `@plunk/types` and `@plunk/shared` (where the zod schemas live) are `private` workspace packages that are not on npm, and they are AGPL-3.0.

---

## 1. Base URL: Cloud vs self-hosted

| Deployment | API base URL | Evidence |
|---|---|---|
| Plunk Cloud (current) | `https://next-api.useplunk.com` | `apps/mcp/src/config.ts:9` (`DEFAULT_API_URL`); `apps/wiki/openapi.json` `servers[0].url`; `apps/wiki/lib/remark-replace-env.mjs` default `API_URL`. Live probe: `GET /health` → 200; `POST /v1/send` without auth → `401 MISSING_AUTH` in the standard envelope. |
| Plunk Cloud (legacy v1) | `https://api.useplunk.com`: **discontinued** | Live probe: `410 Gone`, body says "Plunk v1 (api.useplunk.com) has been discontinued … the API is now available at https://next-api.useplunk.com". ⚠️ `apps/wiki/content/docs/guides/idempotency.mdx` still uses `https://api.useplunk.com` in its examples. That is a stale doc. |
| Self-hosted (official Docker image) | `${PROTOCOL}://${API_DOMAIN}`, e.g. `https://api.example.com`; defaults to `http://api.localhost` | `docker/nginx/setup-nginx.sh` (`API_URI="${API_URI:-${PROTOCOL}://${API_DOMAIN}}"`, `API_DOMAIN` default `api.localhost`, `USE_HTTPS` picks the scheme). `docker/nginx/nginx.conf.template:32-47`: the `server_name ${API_DOMAIN}` block proxies `location /` to the API, so there is **no path prefix**. |
| Self-hosted (raw / dev) | `http://<host>:8080` | API listens on `PORT` (default 8080) (`apps/api/src/app.ts:411`, `apps/wiki/content/docs/self-hosting/environment-variables.mdx:16`). `apps/wiki/.env.example` uses `http://localhost:8080`. |

Notes:
- Routing is **subdomain-based**: API, dashboard, landing and wiki each get their own host (`setup-nginx.sh`, "Using subdomain-based routing"). Express mounts every controller at the root (`app.ts:153-173`), so the API never lives under `/api`. An operator *could* put it behind a path-stripping proxy. The safe node design is therefore a free-form base URL that gets the path appended: strip the trailing slash, then concatenate. This is exactly what `@plunk/mcp` does (`apps/mcp/src/config.ts:63`, `client.ts:52`).
- `GET /health` is unauthenticated and returns `{status, time, environment, uptime, memoryUsage}` (`app.ts:175-183`). `GET /config` is unauthenticated and returns feature flags plus the instance's own `urls.api` (`apps/api/src/controllers/Config.ts:31-77`). Both are handy for diagnostics, but neither validates a key.
- Self-hosted instances can also use an SMTP relay (`apps/smtp`): username `plunk`, password = secret key (`apps/smtp/src/server.ts:279-286`). This is out of scope for an HTTP node; noted only for completeness.

## 2. Authentication

### Key types
- Each project has `public` and `secret` columns, generated as `pk_<64 hex>` / `sk_<64 hex>` (`apps/api/src/controllers/Users.ts:217-218`, and again on rotation at `:272-273`). Rotation (`POST /users/@me/projects/:id/regenerate-keys`) is dashboard-only (see §7).
- Keys are **project-scoped**: the middleware resolves the project from the key and puts it in `res.locals.auth.projectId` (`apps/api/src/middleware/auth.ts:139-154`, `:208-223`, `:275-290`). One credential is one project. `@plunk/mcp` supports several projects only through several keys (`apps/mcp/src/config.ts:110-182`).

### Header format
- `Authorization: Bearer <key>`. Exactly two space-separated parts, and the first must be literally `Bearer`, or the request gets `401 MISSING_AUTH` (`auth.ts:124-131`, `:193-200`, `:261-268`). No query-string or custom-header alternative exists.

### Middleware → key matrix

| Middleware | Accepts | Rejects | Used by |
|---|---|---|---|
| `requirePublicKey` (`auth.ts:114-174`) | `pk_*` only (`ProjectService.public` looks up the `public` column only, `apps/api/src/services/ProjectService.ts:22-30`) | `sk_*` → `401 INVALID_API_KEY` | `POST /v1/track` |
| `requireSecretKey` (`auth.ts:183-243`) | `sk_*` only | `pk_*` → `401 INVALID_API_KEY` | `POST /v1/send`, `POST /v1/verify` |
| `requireAuth` (`auth.ts:253-354`) | `sk_*` when an `Authorization` header is present; otherwise the dashboard JWT cookie `next_token` plus an `X-Project-Id` header | `pk_*` → 401 | every resource route (contacts, templates, campaigns, segments, workflows, events, emails, domains, activity, analytics, projects) |
| `isAuthenticated` (`auth.ts:20-24`) | JWT cookie only | any API key | `/users/@me/*` |
| `requireEmailVerified` (`auth.ts:363-404`) | no-op for API-key auth (`:367-369`) | – | stacked after `requireAuth` on nearly every resource route |

- ⚠️ **Docs disagree.** `apps/wiki/content/docs/api-reference/overview.mdx:208` lists `/v1/track` as "`pk_*` or `sk_*`". The code accepts only `pk_*`. The OpenAPI security description (`apps/wiki/openapi.json` `components.securitySchemes.ApiKeyAuth`) correctly says a secret key "is rejected there with `401`". `@plunk/mcp` works around this: without a public key it upserts the contact (`POST /contacts`) and then calls `POST /events/track` with the secret key (`apps/mcp/src/tools/events.ts:51-68`, `config.ts:24-28`).
- **Disabled projects:** with any auth type, `POST/PUT/PATCH/DELETE` on a disabled project gets `403 PROJECT_DISABLED`. Reads still work (`auth.ts:156-168`, `:225-237`, `:292-304`).
- **CSRF guard:** `assertTrustedOrigin` rejects non-GET requests with a foreign `Origin`, but only on the cookie path (`auth.ts:73-83`, called from `parseJwt`). Server-side callers send no `Origin`, so this does not affect the node.
- **Credential test candidates** (secret key, cheap, read-only): `GET /contacts?limit=1` or `GET /domains/project/current` (§3.9). For a public key, there is no read-only endpoint. The only `pk_` route is the `/v1/track` write.

## 3. Endpoints usable by an integration

Conventions: **Key** = which key works. **Shape** = success response. "Bare" means the resource is returned directly with no envelope. IDs are UUIDs (`packages/shared/src/schemas/index.ts:18`, Prisma `@default(uuid())`). ⚠️ The docs' example IDs `cnt_abc123` / `evt_xyz789` (`overview.mdx:78-79`) are illustrative only.

Several controllers read `req.body` without zod, so the "required" column below reflects the actual hand-written checks, not the shared schema. That matters because errors from those checks come back as bare `{error}` (§5).

### 3.1 Actions (`apps/api/src/controllers/Actions.ts`, schemas `packages/shared/src/schemas/index.ts:616-740`)

| Method | Path | Key | Purpose |
|---|---|---|---|
| POST | `/v1/send` | `sk_` | Send transactional email(s). Rate-limited (`send` bucket); supports `Idempotency-Key` (`Actions.ts:184-185`). |
| POST | `/v1/track` | `pk_` only | Upsert a contact and record an event, which can trigger workflows. Rate-limited (`track` bucket); supports `Idempotency-Key` (`Actions.ts:57-58`). |
| POST | `/v1/verify` | `sk_` | Check an email address for format, MX records, disposable domains and typos (`Actions.ts:470-484`). |

**`POST /v1/send` body** (`schemas/index.ts:623-736`):

| Field | Type | Req. | Notes |
|---|---|---|---|
| `to` | `email` \| `{email, name?}` \| array of either | ✔ | One email per recipient, processed sequentially (`Actions.ts:206-421`). |
| `subject` | string 1–998, no CR/LF | cond. | Required unless `template` is set (superRefine `:704-710`). |
| `body` | string ≥1 (HTML) | cond. | Same rule as `subject`. |
| `template` | UUID | cond. | A template of this project; `subject`/`body`/`from`/`replyTo` fall back to the template's values (`Actions.ts:237-263`). |
| `from` | `email` \| `{email, name?}` | cond. | Required unless the template has one (`Actions.ts:265-276`). The domain must be verified for the project (`Actions.ts:278`). |
| `name` | string | – | Sender name, used when `from` is a string or has no name. |
| `reply` | email | – | Reply-To. |
| `subscribed` | boolean | – | Only changes the contact when it is set explicitly. New contacts from `send` default to **unsubscribed** (`Actions.ts:319-323`). |
| `headers` | `Record<string,string>`, no CR/LF, value ≤998 | – | |
| `data` | JSON object | – | Plain values are persisted to the contact; `{value, persistent:false}` applies to this email only (`Actions.ts:132-134`). |
| `attachments[]` | `{filename (1–255), content (base64), contentType, contentId?, disposition: 'attachment'\|'inline' = 'attachment'}` | – | At most `MAX_ATTACHMENTS_COUNT` (default 10) attachments, `MAX_ATTACHMENT_SIZE_MB` (default 10) in total. `inline` requires `contentId` (`:679-699`, `:723-735`). |
| `delivery.at` | ISO-8601 datetime **with offset** | – | Schedule up to 30 days ahead (`MAX_SCHEDULE_DAYS`, `packages/shared/src/delivery.ts:8`); a time in the past means "now" (`:527-544`). |
| `delivery.throttle` | duration string (`"30m"`, `"24h"`, `"7d"`) \| `{window?, key? (1–255), scope: 'recipient'\|'project'='recipient'}` | – | Allowed window: 1 min to 365 days. An inline send with no template requires `throttle.key` (`:712-721`). |

Response `200` (`Actions.ts:423-429`): `{success:true, data:{emails:[{contact:{id,email}, email: <emailId>|null, status:'queued'|'scheduled'|'throttled', scheduledFor?, nextAllowedAt?}], timestamp}}`.

**`POST /v1/track` body** (`schemas/index.ts:617-622`): `event` string ≥1 ✔; `email` ✔; `subscribed` boolean optional; `data` JSON optional, with the same persistent/non-persistent rule. Reserved system event names are rejected with 422 (`Actions.ts:67-79`). Response `200`: `{success:true, data:{contact:<id>, event:<id>, timestamp}}` (`Actions.ts:102-109`).

**`POST /v1/verify` body:** `{email}` ✔. Response `200`: `{success:true, data:{email, valid, isDisposable, hasMxRecords, suggestedEmail?, reasons:string[]}}` (`Actions.ts:432-483`).

### 3.2 Contacts (`apps/api/src/controllers/Contacts.ts`; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/contacts` | query: `limit` (default 20, max 100), `cursor`, `search` (case-insensitive email substring), `status` = `subscribed`\|`snoozed`\|`unsubscribed`, legacy `subscribed=true\|false`, `sort` = `email`\|`createdAt`, `dir` = `asc`\|`desc` (`:50-82`) | Cursor page `{data: Contact[], total, cursor?, hasMore}` (`apps/api/src/services/ContactService.ts:91-146`) |
| GET | `/contacts/:id` | – | Bare `Contact` (`:143-157`); 404 `CONTACT_NOT_FOUND` |
| POST | `/contacts` | body: `email` ✔, `subscribed?` boolean (new contacts default to `true`), `data?` object. **Upsert by email.** Rate-limited (`contacts:write`) (`:163-187`) | `201` (created) or `200` (updated): `{...Contact, _meta:{isNew, isUpdate}}` |
| PATCH | `/contacts/:id` | body: `email?`, `subscribed?`, `data?` (`:193-208`) | Bare `Contact`. Rate-limited. |
| DELETE | `/contacts/:id` | – | `204` (`:214-228`). Rate-limited. |
| POST | `/contacts/lookup` | body: `emails: string[]` (1–500) (`:367-385`) | `{found: string[], notFound: string[]}` (`ContactService.ts:169-185`) |
| GET | `/contacts/fields` | – | Field list with inferred types (`:89-103`) |
| GET | `/contacts/fields/:field/values` | query `limit` (default 100, max 200) | `{field, values, count, limit}` (`:110-137`) |
| GET | `/contacts/fields/:field/usage` | – | `{usedInSegments, usedInCampaigns, contactCount, canDelete}` (`ContactService.ts:941-946`) |
| DELETE | `/contacts/fields/:field` | – | `{deletedFrom}`; 400 if the field is in use (`:484-504`) |
| POST | `/contacts/import` | multipart field `file`, CSV ≤5 MB (`:16-29`, `:391-419`). ⚠️ No `requireEmailVerified`, which is harmless for API keys. | `202 {message, jobId}` |
| GET | `/contacts/import/:jobId` | – | Job status (`:425-450`) |
| POST | `/contacts/bulk-subscribe` · `/bulk-unsubscribe` · `/bulk-delete` | body is a discriminated union on **`mode`**: `{mode:'ids', contactIds: uuid[1..1000]}` or `{mode:'query', filter?:{search?, status?, subscribed?}, excludeIds?: uuid[≤10000]}` (`schemas/index.ts:114-130`) | `202 {message, jobId}` (`:571-602`) |
| GET | `/contacts/bulk/:jobId` | – | Job status (`:543-568`) |

`Contact` model (`packages/db/prisma/schema.prisma:144-171`): `{id, email, data: Json|null, subscribed: boolean, snoozedUntil: DateTime|null, projectId, createdAt, updatedAt}`. A snoozed contact has `subscribed:false` and a non-null `snoozedUntil`.
⚠️ The docs say bulk endpoints take "up to 1,000 contacts by ID" (`overview.mdx:230-232`) and do not mention that `mode` is required.

### 3.3 Templates (`apps/api/src/controllers/Templates.ts`; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/templates` | query: `page` (1), `pageSize` (20, max 100), `search`, `type` = `TRANSACTIONAL`\|`MARKETING`\|`HEADLESS`, `sort` = `name`\|`createdAt`\|`updatedAt`, `dir` (`:25-38`; `apps/api/src/utils/listSort.ts`) | Page `{data, total, page, pageSize, totalPages}` (`apps/api/src/services/TemplateService.ts:21-54`) |
| GET | `/templates/:id` | – | Bare `Template` |
| POST | `/templates` | body: `name` ✔, `subject` ✔, `body` ✔, `from` ✔ (verified domain), `description?`, `fromName?`, `replyTo?`, `type?` (default `MARKETING` per schema). Liquid syntax is checked (`:65-104`). | `201` bare `Template` |
| PATCH | `/templates/:id` | same fields, all optional | Bare `Template` |
| DELETE | `/templates/:id` | – | `204` |
| POST | `/templates/:id/duplicate` | – | `201` bare `Template` |
| GET | `/templates/:id/usage` | – | Campaigns and workflow steps that reference the template |
| POST | `/templates/bulk-update` | `{ids: uuid[1..1000], delete?: boolean}` (`schemas/index.ts:239-242`) | Result object |

### 3.4 Campaigns (`apps/api/src/controllers/Campaigns.ts`; key `sk_`). ⚠️ These **wrap** responses in `{success:true, data, message?}`

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/campaigns` | query: `page`, `pageSize` (default 20, **no max enforced**, `:85`), `status` (`DRAFT`\|`SCHEDULED`\|`SENDING`\|`SENT`\|`CANCELLED`), `search`, `archived=true`, `sort`, `dir` | **Bare** page `{data, total, page, pageSize, totalPages, archivedCount}` (`apps/api/src/services/CampaignService.ts:272-277`) |
| POST | `/campaigns` | zod `CampaignSchemas.create` (`schemas/index.ts:456-468`): `name` ✔, `subject` ✔, `body` ✔, `from` ✔ email (verified domain), `audienceType` ✔ `ALL`\|`FILTERED`\|`SEGMENT`, `description?`, `fromName?`, `replyTo?`, `type?` (default `MARKETING`), `audienceCondition?` (required when `FILTERED`), `segmentId?` (required when `SEGMENT`) | `201 {success, data: Campaign}` (`:21-60`); created in `DRAFT` |
| GET | `/campaigns/:id` | – | `{success, data}` |
| PUT | `/campaigns/:id` | same fields, all optional (read raw from `req.body`, `:165-166`) | `{success, data}` |
| DELETE | `/campaigns/:id` | – | `{success, message}` |
| POST | `/campaigns/:id/duplicate` · `/archive` · `/unarchive` | – | `{success, data, message}` |
| POST | `/campaigns/:id/send` | body `scheduledFor?` (any `Date`-parseable string; invalid → 400) (`:291-315`) | `{success, data, message}` |
| POST | `/campaigns/:id/cancel` | – | `{success, data, message}` |
| POST | `/campaigns/:id/test` | `{email}` ✔ | `{success, message}` |
| GET | `/campaigns/:id/stats` | – | `{success, data: stats}` |
| GET | `/campaigns/:id/recipients` | query `type` = `bounced` (default)\|`complained`\|`unsubscribed`, `limit` (50, max 100), `cursor` (`:377-395`) | `{success:true, ...cursorPage}` |
| POST | `/campaigns/bulk-update` | `{ids: uuid[1..1000], delete?, archived?}`, one mode per request | Result |

⚠️ `overview.mdx:85` claims dashboard endpoints "return the resource directly — no `success`/`data` wrapper". Campaign endpoints, except the list, do wrap.

### 3.5 Segments (`apps/api/src/controllers/Segments.ts`; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/segments` | – (**no pagination**) | Bare `Segment[]` (`:13-21`) |
| GET | `/segments/:id` | – | Bare `Segment` with cached `memberCount` |
| GET | `/segments/:id/contacts` | query `page` (1), `pageSize` (20, max 100) (`:48-63`) | Page `{data: Contact[], total, page, pageSize, totalPages}` (`apps/api/src/services/SegmentService.ts:68-113`) |
| POST | `/segments` | `name` ✔, `type?` `DYNAMIC` (default)\|`STATIC`, `condition` (object, required when `DYNAMIC`; filter grammar in `schemas/index.ts:144-183`), `description?`, `trackMembership?` (`:70-96`) | `201` bare `Segment` |
| PATCH | `/segments/:id` | `name?`, `description?`, `condition?`, `trackMembership?` | Bare `Segment` |
| DELETE | `/segments/:id` | – | `204`; 409 if an active campaign uses the segment (per docs) |
| POST | `/segments/:id/members` | `{emails: string[] (non-empty), createMissing?, subscribed?}`. **Static segments only.** | Result |
| DELETE | `/segments/:id/members` | `{emails: string[]}` | Result |
| POST | `/segments/:id/compute` | – | Recompute membership (fires `segment.<slug>.entry/exit`) |
| POST | `/segments/:id/refresh` | – | `{memberCount}` |

Filter operators (`schemas/index.ts:146-166`): `equals, notEquals, contains, notContains, greaterThan, lessThan, greaterThanOrEqual, lessThanOrEqual, exists, notExists, within, olderThan, triggered, triggeredWithin, triggeredOlderThan, notTriggered, notTriggeredWithin, memberOfSegment, notMemberOfSegment`. `unit` is one of `days|hours|minutes`. A condition is `{logic: AND|OR, groups: [{filters: [...], conditions?: <nested>}]}`.

### 3.6 Emails (scheduled transactional email) (`apps/api/src/controllers/Emails.ts`; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/emails/:id` | – | Bare `{id, status, sourceType, subject, scheduledFor, sentAt, createdAt, contact:{id,email}}` (`apps/api/src/services/EmailService.ts:123-143`) |
| PATCH | `/emails/:id` | `{delivery:{at: ISO-8601 with offset}}` (strict) (`schemas/index.ts:609-614`) | Bare email; `409 EMAIL_NOT_SCHEDULED` once sending, sent or cancelled |
| POST | `/emails/:id/cancel` | – | Bare email; `409 EMAIL_NOT_SCHEDULED` as above |

`EmailStatus` values: `SCHEDULED, CANCELLED, PENDING, SENDING, SENT, DELIVERED, RECEIVED, OPENED, CLICKED, BOUNCED, COMPLAINED, FAILED` (`schema.prisma`, `enum EmailStatus`).

### 3.7 Events (`apps/api/src/controllers/Events.ts`; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| POST | `/events/track` | `name` ✔, `contactId?` UUID, `emailId?` UUID, `data?` (`:17-49`) | `201` bare `Event`. **This is the secret-key way to track an event**, but it needs a contact ID rather than an email. No idempotency, no rate limit. |
| GET | `/events` | query `eventName?`, `limit` (default 100, **no max**) | `{events}` |
| GET | `/events/stats` | query `startDate?`, `endDate?` | stats object |
| GET | `/events/contact/:contactId` | query `limit` (default 50) | `{events}` |
| GET | `/events/names` | – | `{eventNames}` |
| GET | `/events/:eventName/usage` | – | usage object |
| DELETE | `/events/:eventName` | – | result; 400 if the event name is in use |

### 3.8 Workflows (`apps/api/src/controllers/Workflows.ts`; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/workflows` | query `page`, `pageSize` (20, max 100), `search`, `status` = `active`\|`disabled`, `sort` (`name`\|`createdAt`\|`updatedAt`\|`steps`), `dir` (`:28-47`) | Page object |
| GET | `/workflows/fields` | query `eventName?` | Fields usable in conditions |
| GET | `/workflows/:id` | – | Workflow with steps and transitions |
| POST | `/workflows` | `name` ✔, `eventName` ✔, `description?`, `enabled?` (default false), `allowReentry?`, `throttle?` (`:128-154`) | `201` workflow |
| PATCH | `/workflows/:id` | `name?`, `description?`, `triggerType?` (`EVENT`\|`MANUAL`\|`SCHEDULE`), `triggerConfig?`, `enabled?`, `allowReentry?`, `throttle?` | workflow |
| DELETE | `/workflows/:id` | – | `204` |
| POST | `/workflows/:id/duplicate` | – | `201` workflow |
| POST | `/workflows/bulk-update` | `{ids, delete?}` | result |
| POST/PATCH/DELETE | `/workflows/:id/steps[/:stepId]` (`?splice=true` on delete) | step `{type, name, position, config, templateId?, autoConnect?}` | step / `204` |
| POST/DELETE | `/workflows/:id/transitions[/:transitionId]`, `POST …/transitions/:transitionId/insert-step` | `{fromStepId, toStepId, condition?, priority?}` | transition / step / `204` |
| POST | `/workflows/:id/executions` | `{contactId ✔ UUID, context? JSON}` (`:398-417`) | `201` execution |
| GET | `/workflows/:id/executions` | query `page`, `pageSize` (max 100), `status` (`RUNNING`\|`WAITING`\|`COMPLETED`\|`EXITED`\|`FAILED`\|`CANCELLED`) | Page object |
| GET/DELETE | `/workflows/:id/executions/:executionId` | – | execution (DELETE cancels it) |
| POST | `/workflows/:id/executions/cancel-all` | – | result |

Step types: `TRIGGER, SEND_EMAIL, DELAY, WAIT_FOR_EVENT, CONDITION, EXIT, WEBHOOK, UPDATE_CONTACT`. Step config schemas are in `schemas/index.ts:319-430`.

### 3.9 Domains (`apps/api/src/controllers/Domains.ts`; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/domains/project/:projectId` | **`:projectId` is ignored.** The handler uses the key's project (`:19-28`). `@plunk/mcp` calls `/domains/project/current` (`apps/mcp/src/tools/domains.ts:55`). | Bare `Domain[]` |
| POST | `/domains` | zod `DomainSchemas.create`: **`projectId` ✔ UUID** *and* `domain` ✔ (`schemas/index.ts:433-449`). The handler then uses the key's project anyway (`:38-39`). | `201` domain incl. DNS records |
| GET | `/domains/:id/verify` | – | Verification status (`:102-122`) |
| DELETE | `/domains/:id` | – | `{success:true}` |

⚠️ `POST /domains` with an API key still requires a syntactically valid `projectId` in the body, even though the value is ignored. The project ID is not exposed to API-key callers anywhere, except as `projectId` on returned objects such as `Contact`.

### 3.10 Activity and analytics (read-only; key `sk_`)

| Method | Path | Request | Response |
|---|---|---|---|
| GET | `/activity` | query `limit` (50, max 100), `cursor` (`<epochMs>_<id>`), `types` (comma-separated `ActivityType`), `contactId`, `startDate`, `endDate` (`apps/api/src/controllers/Activity.ts:22-53`) | Cursor page `{data: Activity[], cursor?, hasMore}`, no `total` (`apps/api/src/services/ActivityService.ts:155-167`); `Activity = {id, type, timestamp, contactEmail?, contactId?, metadata}` |
| GET | `/activity/stats` · `/recent-count?minutes≤60` · `/types` · `/upcoming?limit&daysAhead≤90` · `/emails/:id` | – | various |
| GET | `/analytics/overview` · `/timeseries` · `/deliverability` · `/leaderboard?kind=` · `/links` · `/heatmap` · `/audience` · `/events` | common query `from`, `to` (ISO; default last 30 days, max ~92), `tz` (IANA), `stream` = `ALL`\|`TRANSACTIONAL`\|`CAMPAIGN`\|`WORKFLOW` (`apps/api/src/controllers/Analytics.ts:14-66`) | various |

`ActivityType` values: `event.triggered, email.sent, email.delivered, email.received, email.opened, email.clicked, email.bounced, email.complaint, contact.subscribed, contact.unsubscribed, campaign.sent, campaign.scheduled, workflow.started, workflow.completed, workflow.email.scheduled, transactional.scheduled` (`packages/types/src/api/activity.ts`). This feed is a candidate source for a **polling trigger**.
⚠️ The docs list `/analytics/timeseries`, `/top-campaigns`, `/campaign-stats`, `/top-events` (`overview.mdx:333-336`). The code has `overview, timeseries, deliverability, leaderboard, links, heatmap, audience, events`. `top-campaigns`, `campaign-stats` and `top-events` **do not exist** and would 404.

### 3.11 What `@plunk/mcp` (v0.18.0) actually calls

This is a signal of what Plunk itself considers the integrator surface (`apps/mcp/src/tools/*.ts`): `/v1/send`, `/v1/verify`, `/v1/track` (with `pk_`) or the fallback `POST /contacts` + `POST /events/track`; contacts list/get/create/patch/delete (subscribe and unsubscribe are a `PATCH` of `subscribed`, `contacts.ts:203-245`); `GET/POST /templates`; campaigns list/create/get/send/test/cancel/stats; `GET/POST /segments`; `GET /emails/:id`, `PATCH /emails/:id`, `POST /emails/:id/cancel`; `GET /domains/project/current`, `GET /domains/:id/verify`; the full workflow graph API. It sends `Authorization: Bearer`, `Content-Type: application/json` and `User-Agent: plunk-mcp` (`client.ts:74-82`). It treats `204` as empty, parses the error envelope including `errors[]`, and honours `Retry-After` on 429 (`client.ts:92-165`).

## 4. Pagination

| Style | Params | Response | Endpoints |
|---|---|---|---|
| Cursor (keyset on `id`) | `limit` (default 20, max 100), `cursor` | `{data, cursor?, hasMore, total}`; `total` is computed only on the first page, later pages return `0` (`ContactService.ts:136-145`; type `CursorPaginatedResponse` in `packages/types/src/common/pagination.ts`) | `GET /contacts` |
| Cursor (opaque composite) | `limit` ≤100, `cursor` | `{data, cursor?, hasMore}`, no `total` | `GET /activity` (cursor `<epochMs>_<id>`, `ActivityService.ts:159-167`), `GET /campaigns/:id/recipients` (base64url cursor, wrapped as `{success:true, data, cursor, hasMore}`, `CampaignService.ts:1314-1388`) |
| Page / offset | `page` (1-based), `pageSize` (default 20, max 100, except campaigns, which have no cap) | `{data, total, page, pageSize, totalPages}` (`PaginatedResponse`) | `/templates`, `/campaigns` (+`archivedCount`), `/workflows`, `/workflows/:id/executions`, `/segments/:id/contacts` |
| None | sometimes `limit` | whole list | `/segments`, `/domains/project/:id`, `/events` (`limit`, default 100, no max), `/events/contact/:id`, `/events/names`, `/contacts/fields` |

⚠️ `overview.mdx:141-165` says "Most list endpoints use cursor-based pagination" and names only `/segments/:id/contacts` as page-based. In code, cursor pagination covers contacts, activity and campaign recipients only. Templates, campaigns and workflows are page-based.

"Return all" therefore needs two paginator strategies: loop on `cursor` until `hasMore === false`, or loop `page` until `page >= totalPages`.

## 5. Error format

The global handler in `apps/api/src/app.ts:196-396` returns:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "statusCode": 422,
    "requestId": "uuid",
    "errors": [{"field": "email", "message": "Invalid email", "code": "invalid_string", "received": "..."}],
    "details": {},
    "suggestion": "..."
  },
  "timestamp": "ISO-8601"
}
```

- Zod errors → **422** `VALIDATION_ERROR` with `errors[]` (`app.ts:242-286`). Invalid JSON → 400 `VALIDATION_ERROR` (`app.ts:214-239`). `HttpException` subclasses → their own status and `code`, plus optional `details` (`app.ts:289-370`). Anything else → 500 `INTERNAL_SERVER_ERROR` (`app.ts:373-395`). Unknown route → 404 "Unknown route" (`app.ts:187-189`).
- Error codes (`apps/api/src/exceptions/index.ts:5-42`): `UNAUTHORIZED, INVALID_CREDENTIALS, MISSING_AUTH, INVALID_API_KEY, FORBIDDEN, PROJECT_ACCESS_DENIED, PROJECT_DISABLED, EMAIL_VERIFICATION_REQUIRED, RESOURCE_NOT_FOUND, CONTACT_NOT_FOUND, TEMPLATE_NOT_FOUND, CAMPAIGN_NOT_FOUND, WORKFLOW_NOT_FOUND, CONFLICT, IDEMPOTENCY_KEY_REUSED, EMAIL_NOT_SCHEDULED, BAD_REQUEST, VALIDATION_ERROR, INVALID_EMAIL, INVALID_REQUEST_BODY, MISSING_REQUIRED_FIELD, RATE_LIMIT_EXCEEDED, BILLING_LIMIT_EXCEEDED, UPGRADE_REQUIRED, INTERNAL_SERVER_ERROR, DATABASE_ERROR, EXTERNAL_SERVICE_ERROR`.
- Status mapping: `NotFound` → 404 (`details:{resource,id}`); `NotAllowed` → 403; `NotAuthenticated` → 401; `NeedToUpgrade` → 402; `ValidationError` → 422; `ConflictError` → 409; `RateLimitError` → 429 (`details:{retryAfter}`); `BadRequest` → 400 (`exceptions/index.ts:72-169`). A `HttpException` thrown without an error code falls back to `code: INTERNAL_SERVER_ERROR` even for 4xx (`app.ts:291`), e.g. campaign `400 'Segment ID is required…'`.
- ⚠️ **Second error shape.** Many controller paths `return res.status(4xx|5xx).json({error: '<message>'})` directly, with no `success`, `code` or `requestId`. Examples: `Contacts.ts:171, 375, 398, 581`; `Templates.ts:73-85`; `Segments.ts:78, 84, 165`; `Workflows.ts:137, 141, 243`; `Domains.ts:67-93`; `Events.ts:25`. The node's error mapper must handle `body.error` being either an object or a string. The `X-Request-ID` response header is always set (`apps/api/src/middleware/requestId.ts:8-22`) and is the reliable correlation ID. Callers may also send their own `X-Request-ID`.

## 6. Rate limits, idempotency, versioning, misc

**Rate limits** (`apps/api/src/middleware/rateLimit.ts`, `apps/api/src/app/constants.ts:150-172`):
- Token bucket per project per bucket in Redis. Buckets: `track` (`/v1/track`, 100/s sustained, 200 burst), `send` (`/v1/send`, 20/s, 50 burst), `contacts:write` (`POST/PATCH/DELETE /contacts`, 50/s, 100 burst). All are configurable via env vars. No other route is limited.
- Every limited response carries `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset` (seconds until the bucket is full). A 429 adds `Retry-After` (≥1 s) and `error.details.retryAfter` (`rateLimit.ts:133-150`).
- **Opt-in:** `RATE_LIMIT_ENABLED` must be `"true"` (`constants.ts:160`). Self-hosted instances are unlimited by default. If Redis fails, the limiter lets the request through (`rateLimit.ts:151-154`).
- ⚠️ `overview.mdx:167-175` says "API requests — 1000 requests/minute per project". No such global limiter exists in the code. Cloud may have something in front of the app; that is unverifiable from source.
- Separately, `/v1/send` is bounded by the project's billing limits (`BILLING_LIMIT_EXCEEDED` / `UPGRADE_REQUIRED`), and `delivery.throttle` gives per-notification de-duplication (`Actions.ts:291-379`).

**Idempotency** (`apps/api/src/middleware/idempotency.ts`; guide `apps/wiki/content/docs/guides/idempotency.mdx`):
- Optional header `Idempotency-Key`, 1–255 printable ASCII characters. It applies to `POST /v1/send` and `POST /v1/track` only. The key is unique per project and expires after `IDEMPOTENCY_KEY_TTL_HOURS` (default 24) (`constants.ts:148`).
- A reused key gets `409 IDEMPOTENCY_KEY_REUSED` with `details:{key, originalRequest, originalRequestAt, originalStatusCode|null}`. **The original response is not replayed.**
- The claim is released on a 4xx (so the request can be fixed and retried with the same key) and kept on 2xx and 5xx (`idempotency.ts:19-23, 81-90`). One key covers the whole multi-recipient `/v1/send` request.

**Versioning:** only the literal `/v1` prefix on `send`, `track` and `verify` (`Actions.ts:23`). Resource routes are unversioned. There is no version header or negotiation. The OpenAPI document is `"version": "1.0.0"` and covers only 18 operations (`apps/wiki/openapi.json`). The `@plunk/mcp` package version tracks the release (0.18.0, `apps/mcp/package.json`). Compatibility therefore has to be pinned to Plunk releases, e.g. via contract tests against a pinned Docker image.

**Misc:** the JSON body limit is 50 MB (`app.ts:82`). CORS is open (`*`) only on `/v1/*` (`app.ts:98-124`), which is irrelevant for the server-side n8n node. Successful DELETEs return `204` with no body for contacts, templates, segments and workflows, but `200` JSON for campaigns (`{success,message}`) and domains (`{success:true}`).

## 7. Dashboard-only (unusable with an API key)

| Area | Routes | Why |
|---|---|---|
| Session auth | `POST /auth/login, /signup, /logout, /verify-email, /request-verification, /request-password-reset, /reset-password`, `GET /auth/oauth-config` (`apps/api/src/controllers/Auth.ts`) | Cookie/JWT flows |
| OAuth | `/github/outbound|callback`, `/google/outbound|callback` (`apps/api/src/controllers/Oauth/*`) | Browser redirects |
| User and project admin | all of `/users/@me/*`, including project create/update, **key rotation**, billing, security, reset, delete (`apps/api/src/controllers/Users.ts:32-888`) | `isAuthenticated` is cookie-only (`auth.ts:20-24`) |
| Project team | `/projects/:id/setup-state`, `/security`, `/members*` (`apps/api/src/controllers/Projects.ts`) | They use `requireAuth`, but then call `MembershipService.requireAccess(auth.userId!, id)` (`Projects.ts:26, 90, 113, 150, 204, 249`). With an API key `userId` is undefined, so the call fails. ⚠️ Inferred from code, not tested: the result is probably a 404 or 500. |
| Uploads | `POST /uploads/image` (`apps/api/src/controllers/Uploads.ts:46-47`) | Technically `requireAuth` (so `sk_` works), but it is meant for the editor. Optional for the node. |
| Public recipient pages | `GET /contacts/public/:id`, `POST /contacts/public/:id/{subscribe,unsubscribe,snooze}`, `GET|POST /unsubscribe/:id` (`Contacts.ts:234-361`, `apps/api/src/controllers/Unsubscribe.ts`) | **No auth.** Keyed by contact UUID and intended for recipients. An integration should use `PATCH /contacts/:id` instead. |
| Infrastructure webhooks | `POST /webhooks/sns`, `POST /webhooks/incoming/stripe` (`apps/api/src/controllers/Webhooks.ts:116, 712`) | Inbound from AWS and Stripe |
| Diagnostics | `GET /health`, `GET /config` | Unauthenticated, read-only |

## 8. Outbound webhooks (input for the trigger ticket)

Plunk has **no webhook-subscription API**. Webhooks are a `WEBHOOK` **step inside a workflow** (`apps/wiki/content/docs/guides/webhooks.mdx:10-20`). The step config is `{url (http/https), method: GET|POST|PUT|PATCH|DELETE = POST, headers?: Record<string,string>, body?: JSON}` (`packages/shared/src/schemas/index.ts:413-418`).
- Triggering events include system events (`email.sent, email.delivery, email.open, email.click, email.bounce, email.complaint, email.received, contact.subscribed, contact.unsubscribed, segment.<slug>.entry|exit`) and any custom event (`webhooks.mdx:26-55`).
- The default payload is `{contact:{email,subscribed,data}, workflow:{id,name}, execution:{id,startedAt}, event:{…}}` (`webhooks.mdx:146-181`). Requests time out after 10 s, are **not retried**, and are **unsigned**. Authenticity relies on a shared secret header you configure yourself (`webhooks.mdx:570-584`).
- A trigger node could therefore create and enable a workflow on activation (`POST /workflows`, `POST /workflows/:id/steps` with `type: WEBHOOK`, `PATCH /workflows/:id {enabled:true}`) and delete it on deactivation, using the secret key. All of these routes accept `sk_`. An alternative is a polling trigger on `GET /activity` (§3.10).

## 9. Reusable schemas for contract tests

- **`@plunk/types`** (`packages/types/package.json`): `"private": true`, version `0.0.0`, **not published to npm**. It contains TypeScript types only (no zod), e.g. `PaginatedResponse`, `CursorPaginatedResponse`, `ActivityType`, `SendEmailResult`.
- **The zod request schemas are in `@plunk/shared`** (`packages/shared/src/schemas/index.ts`): also `"private": true`, version `0.0.1`, not published. It depends on `@plunk/db` (Prisma enums) and on `zod ^3.23.8` (v3, not v4).
- Response shapes have **no zod schemas at all**. Responses are Prisma rows serialised as-is.
- Licence: the Plunk repo is **AGPL-3.0** (`LICENSE`; `apps/mcp/package.json` `"license": "AGPL-3.0-only"`), while our repo is MIT. Copying or vendoring the schemas into our repo would bring AGPL code in.
- **Conclusion:** neither package can be added as a devDependency. The practical route is to hand-write our own zod contract schemas from this document and verify them against a pinned self-hosted Docker image. Request-side behaviour can additionally be checked by observing the 422 `errors[]` the real API returns.

## 10. Docs vs code disagreements (summary)

1. `/v1/track` with `sk_`: docs say yes (`overview.mdx:208`); code says **no**, 401 (`auth.ts:140-148`). The OpenAPI agrees with the code.
2. Envelope: docs say resource endpoints are unwrapped (`overview.mdx:85`); **campaign** endpoints wrap in `{success,data}` (`Campaigns.ts:56-59` etc.).
3. Pagination: docs say "most" endpoints use cursors (`overview.mdx:143`); only contacts, activity and campaign recipients do (§4).
4. Rate limit: docs say 1000 req/min per project (`overview.mdx:172`); code has opt-in per-route token buckets only (§6).
5. Analytics paths in the docs (`top-campaigns`, `campaign-stats`, `top-events`) do not exist (§3.10).
6. Bulk contact ops: the docs omit the required `mode` discriminator (§3.2).
7. Error envelope: the docs say "All errors include detailed information" (`overview.mdx:108`); many paths return bare `{error: string}` (§5).
8. Legacy host `api.useplunk.com` in `guides/idempotency.mdx` is now `410 Gone`.
9. Example IDs `cnt_…`/`evt_…` in the docs; real IDs are UUIDs.

## 11. Open questions

1. **Cloud rate limits:** is `RATE_LIMIT_ENABLED=true` on `next-api.useplunk.com`, with which values, and is there an edge/global limit (the documented 1000/min)? This needs a Cloud account or a Plunk maintainer.
2. **Is a key-only `pk_` credential useful?** The only `pk_` route is `/v1/track`. Should the credential hold both keys (as `@plunk/mcp` does: secret required, public optional), or should "Track Event" use `sk_` via `POST /contacts` + `POST /events/track`? The two paths differ: the fallback has no idempotency or rate limiting, and the upsert-then-track pair is not atomic.
3. **`/projects/*` with an API key:** the exact failure mode (404 or 500) is unverified. It does not matter for the node, but it should be confirmed against Docker if we document it.
4. **`POST /domains` needs `projectId`:** how does an API-key caller obtain its project ID? Possibly from `projectId` on any returned `Contact`/`Template`. Worth confirming before offering "Add Domain".
5. **Stability of unversioned routes:** Plunk publishes no compatibility promise for the non-`/v1` routes. How much of the node should depend on them? Contract tests against pinned releases mitigate the risk.
6. **Webhook trigger viability:** can a programmatically created `WEBHOOK` workflow fully replace a native subscription, given the lack of retries and signatures? Is creating workflows in a user's project acceptable UX? This belongs to the trigger-design ticket.
7. **Campaign `pageSize` has no upper bound and `GET /events` `limit` has no max.** Should the node impose its own caps?
