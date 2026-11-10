# Research 02: Plunk events for triggers

Ticket: [.scratch/plunk-n8n-node/issues/02-plunk-events-for-triggers.md](../../.scratch/plunk-n8n-node/issues/02-plunk-events-for-triggers.md)

**Source read:** `useplunk/plunk`, branch `next`, commit `37e5f81` (2026-11-09). Every claim cites `path:line` at that commit. Where the code and the wiki disagree, the code wins, and the disagreement is listed in [Docs vs code](#docs-vs-code-disagreements).

"Plunk workflow" means Plunk's own automation feature. "n8n workflow" means the user's workflow in n8n.

## TL;DR

1. **Plunk has one outbound push mechanism: the `WEBHOOK` step of a Plunk workflow.** There are no project-level webhook subscriptions, no signing secret, and no webhook model in the schema (`packages/db/prisma/schema.prisma:847-856`; a search for webhook, secret or signing in the schema finds only `Project.secret` = the sk API key, line 43). `apps/api/src/controllers/Webhooks.ts` handles **inbound** traffic only: AWS SNS for SES events (`:109-118`) and Stripe (`:712`).
2. **Every event goes through one function, `EventService.trackEvent`** (`apps/api/src/services/EventService.ts:23-48`). It stores an `Event` row, then starts every enabled Plunk workflow whose trigger `eventName` matches. So anything that is a Plunk event can be pushed to n8n, including system events, custom events, inbound email and segment entry/exit.
3. **The webhook lifecycle can be automated with the secret API key.** `/workflows` CRUD, steps and enable/disable all use `requireAuth`, which accepts `Authorization: Bearer sk_*` (`apps/api/src/middleware/auth.ts:253-307`). Plunk's own MCP server builds Plunk workflows, including WEBHOOK steps, this way (`apps/mcp/src/tools/workflows.ts:172-175, 672-676`). So n8n's `checkExists/create/delete` can map to *create a Plunk workflow "trigger(event) → WEBHOOK(n8n URL)"*, *GET it*, and *disable + delete it*. Caveat: `/workflows`, `/events` and `/activity` are **not in the published OpenAPI spec** (`apps/wiki/openapi.json`), so they are source-public but not documented as stable.
4. **Security comes from a shared secret only.** Header values are templated and sent as-is, so the trigger can generate a random secret, put it in a custom header, and verify it. Nothing is HMAC-signed.
5. **Hard constraints for the trigger:**
   - The WEBHOOK step refuses private and loopback IPs (SSRF guard, not configurable), which matters for the local Docker test setup.
   - It is called **synchronously and with no retry** (10 s timeout).
   - It only fires for events that **have a contact**.
   - The Plunk workflow must have `allowReentry: true`, or each contact fires only once.
6. **Polling fallback:**
   - `GET /events?eventName=&limit=` returns the newest first, has no cursor and no `since` parameter, and has no server cap on `limit`.
   - `GET /activity?startDate=&types=&cursor=` is a unified feed with a keyset cursor and at most 100 per page.
   - `GET /contacts` is newest first with a cursor. It is the only way to detect new contacts, because **there is no `contact.created` event**.

## Event catalogue

`trackEvent` call sites (from a search for `trackEvent(` in `apps/api/src`, excluding tests) are the complete list of event producers.

| Event name (exact) | Fires when | Producer (code) | `event` payload fields | Has contact? → can be pushed by a webhook |
|---|---|---|---|---|
| `email.sent` | SES accepted the send | `apps/api/src/jobs/email-processor.ts:376-386` | base fields¹ + `sentAt` | yes |
| `email.delivery` | SES Delivery notification | `Webhooks.ts:420, 484-492, 680` | base¹ + `deliveredAt` | yes |
| `email.open` | SES Open; **opens marked `isBotEvent: "Likely"` are dropped** (`Webhooks.ts:497-500`) | `Webhooks.ts:494-518` | base¹ + `openedAt` (first open), `opens`, `isFirstOpen` | yes |
| `email.click` | SES Click | `Webhooks.ts:520-537` | base¹ + `link`, `clickedAt` (first click), `clicks`, `isFirstClick` | yes |
| `email.bounce` | SES Bounce: Permanent, Transient, unknown, or account suppression | `Webhooks.ts:539-617`; suppression `:457-480` | base¹ + `bounceType`, `bounceSubType`, `diagnosticCode`, `feedbackId`, then `bouncedAt` (permanent/unknown) **or** `transientBounce: true` **or** `suppressedByProvider: true` | yes |
| `email.complaint` | SES Complaint, or account suppression | `Webhooks.ts:619-648`; `:457-480` | base¹ + `complainedAt`, `complaintFeedbackType`, `complaintSubType`, `userAgent`, `feedbackId`, `arrivalDate` (or `suppressedByProvider: true`) | yes |
| `email.received` | Inbound mail to a verified domain (SES receiving through SNS) | `Webhooks.ts:207-381`, data `:339-358` | `messageId, from, fromHeader, to, subject, timestamp, recipients[], hasContent, body` (sanitised HTML), `spamVerdict, virusVerdict, spfVerdict, dkimVerdict, dmarcVerdict, processingTimeMillis` | yes (the sender is upserted as a **subscribed** contact, `:301-310`) |
| `contact.subscribed` | Subscription flips to true through the API, the dashboard, a public subscribe page, bulk subscribe, or an UPDATE_CONTACT step; also when a snooze expires | `ContactService.ts:319-325, 401-407, 599-612, 1021`; snooze expiry `:806-809`; `WorkflowExecutionService.ts:1255-1262` | `{}` usually; `{reason: "snooze_expired"}` on snooze expiry | yes |
| `contact.unsubscribed` | Subscription flips to false through the API, dashboard, unsubscribe page, bulk, or an UPDATE_CONTACT step; also on snooze | `ContactService.ts:319-325, 401-407, 621-632, 1071`; snooze `:723-737` | `{}` usually; snooze: `{reason: "snooze", duration, snoozedUntil}` | yes |
| `segment.<slug>.entry` / `.exit` | A **dynamic** segment with `trackMembership: true` is recomputed (cron **every 5 min**, `apps/api/src/app.ts:491-502`, or manually through `POST /segments/:id/compute`, `controllers/Segments.ts:200-214`) | `SegmentService.ts:575-587, 606-618`; guard `:454-469` | `{segmentId, segmentName}` | yes |
| custom (`purchase`, `user.signup`, …) | `POST /v1/track` (pk key, upserts the contact) or `POST /events/track` (sk key) | `controllers/Actions.ts:57-110`; `controllers/Events.ts:17-49` | whatever `data` was sent. Non-persistent values arrive wrapped as `{value, persistent: false}` (`WorkflowEntryService.ts:161-165`) | `/v1/track`: always. `/events/track`: **only if `contactId` is given**, otherwise it cannot be pushed |

¹ Base fields on every outbound email event: `subject, from, fromName, messageId, emailId, templateId, campaignId, sourceType` (`Webhooks.ts:438-447`; `email.sent` builds the same set at `email-processor.ts:376-386`). `sourceType` is one of `TRANSACTIONAL | CAMPAIGN | WORKFLOW | INBOUND` (`schema.prisma:930-935`).

### Events that do not exist (a trigger cannot listen to them)

- **`contact.created` / `contact.deleted` / contact updated.** Neither create path tracks an event (`ContactService.ts:203-227`, `:417-430`). Poll `GET /contacts` instead (below).
- **Campaign-level events** (campaign sent, scheduled, finished). Campaign mails emit per-recipient `email.*` events with `campaignId` set. `campaign.sent` and `campaign.scheduled` exist only as **activity feed types** that are derived from rows, not as events (`packages/types/src/api/activity.ts:8-26`). Poll `GET /campaigns?status=SENT` or `GET /activity?types=campaign.sent`.
- **`email.failed`.** A send failure only sets `status = FAILED` (`email-processor.ts:391-401`). No event is tracked.
- **Workflow started/completed.** These are activity feed types only (`activity.ts:21-23`).
- **Reserved names** cannot be faked through `/v1/track`: anything starting with `email.`, `contact.subscribed/unsubscribed`, and `segment.*.entry|exit` (`EventService.ts:328-346`, enforced in `Actions.ts:67-79`). `/events/track` does **not** check this (`Events.ts:17-49`).

### Preconditions per event

- `email.open` and `email.click` need tracking to be on. Project `tracking` is `ENABLED | DISABLED | MARKETING_ONLY` (`schema.prisma:71, 821-825`; `EmailService.ts:387-398`). On a self-hosted instance they also need the SES configuration set with Opens and Clicks published to SNS (`apps/wiki/content/docs/self-hosting/email-setup.mdx:50-57`).
- All SES-derived events (`delivery/open/click/bounce/complaint/received`) need the instance's SNS → `/webhooks/sns` loop to work (`Webhooks.ts:116-141`). That is given on Cloud and is up to the operator when self-hosted.
- `email.received` needs a verified domain with the inbound MX record (`apps/wiki/content/docs/guides/receiving-emails.mdx:29-37`). It is dropped silently when the inbound billing cap is hit (`Webhooks.ts:291-299`). It fans out to every project that verified the domain (`receiving-emails.mdx:168`).

## Delivery mechanism: the Plunk workflow WEBHOOK step

### Path from event to HTTP request

1. `trackEvent` creates an `Event` row and calls `triggerWorkflows` (`EventService.ts:30-45`).
2. `triggerWorkflows` loads enabled `EVENT` workflows. It caches them in Redis for 5 min, and the cache is invalidated on create, update and delete (`EventService.ts:352-392`; `WorkflowService.ts:223-225, 336-344, 390-393`). It matches `triggerConfig.eventName === eventName` **exactly**, with no wildcards (`EventService.ts:398`).
3. **If there is no `contactId`, nothing starts.** The event is only logged (`EventService.ts:400-406`).
4. `WorkflowEntryService.enter` applies the entry rules (`WorkflowEntryService.ts:42-101`):
   - `allowReentry: false` (the **default**, `schema.prisma:394`): once per contact, ever (`:50-57`).
   - `allowReentry: true`: refused while another run for the same contact is `RUNNING` (`:79-81`).
   - With a throttle: once per window and key (`:59-78`).
5. The steps then run **inline, in the same call stack**. `processStepExecution` → `processNextSteps` → `processStepExecution(next)` (`WorkflowExecutionService.ts:52, 1352-1354`), with no queue hop for TRIGGER→WEBHOOK. The HTTP call therefore happens inside the SNS handler, inside `/v1/track`, and so on. A slow n8n endpoint adds latency to Plunk's own request, for example the `/v1/track` response.

### The HTTP request (`WorkflowExecutionService.ts:1098-1182`)

- **Config schema** (`packages/shared/src/schemas/index.ts:413-418`): `{url: string (URL), method: GET|POST|PUT|PATCH|DELETE = POST, headers?: Record<string,string>, body?: JSON}`. The dashboard dialog only exposes url, method and headers (`apps/web/src/components/workflow-steps/WebhookStepDialog.tsx:25-26, 40-49`). `body` is reachable through the API.
- **Templating:** `{{…}}` is rendered in the URL, in header values and in body strings. The method is not templated (`:1134-1138`). The scope is `id, email`, the contact data keys, the execution context, `data.*`, `event.*` (the trigger payload), and `unsubscribeUrl/subscribeUrl/manageUrl` (`:1122-1132`).
- **Method and headers:** `Content-Type: application/json` plus the rendered custom headers. GET sends no body (`:1158-1165`). There is no User-Agent or signature header from Plunk.
- **Default body** (used when no `body` is configured, `:1140-1155`):

  ```json
  {
    "contact":   {"email": "…", "subscribed": true, "data": {…}},
    "workflow":  {"id": "<plunk workflow id>", "name": "…"},
    "execution": {"id": "<execution id>", "startedAt": "<ISO>"},
    "event":     { …trigger event data, see catalogue… }
  }
  ```

  The default body contains **no contact ID, event name, event ID or emailId at the top level.** The event name is not in the body at all, so the trigger must know it from its own config or from the URL. The contact ID can be added with a templated header or query, for example `X-Plunk-Contact-Id: {{id}}`. A custom `body` **replaces** the default payload entirely (wiki `webhooks.mdx:87`).
- **Transport:** a 10 s timeout per hop (`:1069`). Up to 5 redirects are followed manually and each is re-validated (`:1043, 1072-1080`). Only `http:` and `https:` are allowed (`:1049-1051`).
- **SSRF guard (not configurable):** the hostname is resolved, and loopback, RFC1918, CGNAT 100.64/10, link-local, multicast and the private IPv6 ranges are rejected (`:1060-1064, 903-918, 985-1036`). **n8n on `localhost`, or on a Docker bridge network (172.16/12) next to a self-hosted Plunk, is unreachable.** Both the local test setup and private self-hosted deployments need a publicly resolvable URL for n8n, such as a tunnel or a public reverse proxy.

### Retries and failure semantics

- **There is no retry.** The webhook runs synchronously (see above). A thrown error marks the step and the execution `FAILED` and sends an operator ntfy notification (`WorkflowExecutionService.ts:264-319`). `startWorkflowForContact` then swallows the error (`EventService.ts:470-472`). The BullMQ `workflow` queue has `attempts: 3` (`QueueService.ts:75-86`), but it is not used on this path. Even if a job were retried, a `FAILED` execution is skipped (`WorkflowExecutionService.ts:89-94`).
- **What throws:** timeout, DNS failure, private IP, a bad scheme, or too many redirects.
- **A non-2xx response does not fail the step.** The step completes with output `{statusCode, success: false, response}` (`:1175-1181`), and the run completes.
- SNS-side retries do not help either. The SNS handler always answers 200, even on processing errors (`Webhooks.ts:701-705`). Note that SNS is at-least-once, and duplicates are only partly deduplicated: bounce and complaint timestamps are idempotent (`Webhooks.ts:422-428`), but the event is tracked again. So **n8n can receive duplicate email events.** Deduplicate on `event.emailId` + event name (+ `opens`/`clicks` count).

### Ordering and loss

- SNS gives no ordering guarantee (`Webhooks.ts:29-44`). For example, `email.open` can arrive after `email.complaint`.
- With `allowReentry: true` and no throttle, a second event for the **same contact** that arrives while the first run is still `RUNNING` is refused with `in-progress` and **dropped** (`WorkflowEntryService.ts:79-81`). This is a narrow window, because a TRIGGER→WEBHOOK run takes about the webhook round-trip. Concurrent duplicate clicks or opens can still lose events.

## Can the trigger manage webhooks through the API? Yes, with the secret key.

All routes in `apps/api/src/controllers/Workflows.ts` use `requireAuth, requireEmailVerified`. `requireAuth` accepts `Authorization: Bearer sk_*` (`auth.ts:253-307`), and `requireEmailVerified` passes API keys (`auth.ts:367-369`).

| Lifecycle need | Call | Notes |
|---|---|---|
| create | `POST /workflows` `{name, description?, eventName, enabled?: false, allowReentry: true}` (`Workflows.ts:128-154`) | Creates the workflow and its TRIGGER step automatically (`WorkflowService.ts:180-221`). The response does **not** include steps (`apps/mcp/src/tools/workflows.ts:678-680`). |
| | `POST /workflows/:id/steps` `{type: "WEBHOOK", name, position: {x,y}, config: {url, method: "POST", headers: {"X-Plunk-Trigger-Secret": "<random>"}}}` (`Workflows.ts:230-256`) | `type, name, position, config` are all required (`:242-244`). `autoConnect` defaults to true and links the step from the leaf TRIGGER step (`WorkflowService.ts:595-618`). |
| | `PATCH /workflows/:id` `{enabled: true}` (`Workflows.ts:160-184`) | Invalidates the trigger cache (`WorkflowService.ts:336-344`). |
| checkExists | `GET /workflows/:id` (`Workflows.ts:108-122`) → 404 when it is gone (`WorkflowService.ts:130`) | Returns the steps, so the code can also check that the WEBHOOK URL still matches the current n8n URL. |
| delete | `PATCH {enabled: false}` → `POST /workflows/:id/executions/cancel-all` (`Workflows.ts:488`) → `DELETE /workflows/:id` (`:190-204`, returns 204) | DELETE returns **409 while executions are RUNNING or WAITING** (`WorkflowService.ts:376-384`), so the trigger disables and cancels first. |
| event picker (loadOptions) | `GET /events/names` (`Events.ts:110-119`) | Only names that have already occurred. Merge with the static system list. Segment names must be slugified (`slugifySegmentName`). |

**Design consequences:**

- Use one Plunk workflow per selected event name, because matching is exact.
- Name the Plunk workflow recognisably, for example `n8n: <n8n workflow> (<event>)`, and store `workflowId` and the secret in `staticData`.
- Changing the event of a running workflow can return 409 (`WorkflowService.ts:265-280`), so delete and recreate instead.

Each create and delete of a Plunk workflow sends an operator ntfy notification (`WorkflowService.ts:233-236, 396`). This is harmless for the user, but Plunk Cloud operators will see one notification per n8n activation.

The wiki itself only describes setting this up by hand in the dashboard (`apps/wiki/content/docs/guides/webhooks.mdx:57-144`). Automating it through the API is supported by the code and by `@plunk/mcp`, but it is not documented as a public contract.

## Verification and security

- **There is no signature, no timestamp and no Plunk-issued secret** on outbound requests (`WorkflowExecutionService.ts:1158-1165`, and no signing code anywhere in `executeWebhook`).
- The recommended mechanism is a **shared-secret header** (wiki `webhooks.mdx:575-584`). The trigger generates a random secret at create time, sets it as a custom header on the WEBHOOK step, and rejects any request where it does not match. The secret is stored in the Plunk workflow config and is visible to anyone with dashboard or sk access to the project.
- Plunk *verifies* its own inbound SNS traffic (signature plus topic owner, `Webhooks.ts:120-141`) and Stripe (`:722-736`). That is not relevant to n8n.

## Polling fallback

Use polling for events with no contact, for the events that do not exist (new contacts, campaign status), or as a simpler, no-registration alternative. All of these endpoints accept the sk key (`requireAuth`), and **none of them are in the OpenAPI spec.**

| Endpoint | Order / cursor | Filters | Use for | Caveats |
|---|---|---|---|---|
| `GET /events?eventName=&limit=` (`Events.ts:55-66` → `EventService.ts:80-96`) | `createdAt desc`. **No cursor, no `since`.** | one `eventName` | any event, including contact-less custom events. Rows: `id, name, data, contactId, emailId, createdAt, contact.email` (`schema.prisma:673-703`) | `limit` defaults to 100 and has **no server cap**. The poller keeps `lastCreatedAt` + the IDs seen at that timestamp, and must size `limit` above the event rate per poll interval or it misses events. |
| `GET /activity?startDate=&types=&contactId=&cursor=&limit=` (`Activity.ts:22-53` → `ActivityService.ts:95-166`) | Newest first. Keyset cursor `"<ms>_<id>"` (`:129-136, 159-161`). `hasMore`. | `types` (comma list of `ActivityType`), `contactId`, `startDate`/`endDate` | a unified feed: events, email rows, workflow rows, `campaign.sent` | Max 100 per page (`:53`). **`startDate` is ignored once a cursor is passed** (`:121-125`), so the poller must stop paging when the timestamp is ≤ the last poll. The activity type names differ from the event names (`email.opened` vs `email.open`, `activity.ts:8-26`). The code itself warns that cross-source cursor pagination can miss items (`ActivityService.ts:86-88`). |
| `GET /contacts?limit=&cursor=&sort=createdAt&dir=desc&status=` (`Contacts.ts:50-82` → `ContactService.ts:91-144`) | `createdAt desc, id desc`, cursor = `id` | `status=subscribed|snoozed|unsubscribed`, `search` | **new contact** (`contact.created` does not exist) | Stop when `createdAt` ≤ the last seen. There is no `updatedAt` sort, so contact updates cannot be polled efficiently. |
| `GET /campaigns?status=SENT&sort=updatedAt&dir=desc&page=&pageSize=` (`Campaigns.ts:74-103`) | page-based | `status` ∈ `DRAFT|SCHEDULED|SENDING|SENT|CANCELLED` (`schema.prisma:827-833`) | **campaign sent / status changed** | page/pageSize, not a cursor |
| `GET /events/contact/:contactId?limit=` (`Events.ts:89-104`) | `createdAt desc` | contact | per-contact history | Not a trigger feed. |

## Mapping onto n8n's two trigger patterns

- **Webhook trigger with lifecycle methods** (`webhookMethods.default.{checkExists, create, delete}`, with state in `getWorkflowStaticData('node')`). This fits every event that has a contact, which is all system events, segment events, and custom events from `/v1/track`. Use the recipe in the API table above. Verify the secret header in `webhook()` and return the body (`contact`, `workflow`, `execution`, `event`) plus the configured event name.
- **Polling trigger** (`polling: true` + `poll()`, with the cursor in staticData). Use it for new contacts, campaign status, custom events without a contact, and as an option for users whose n8n is not publicly reachable. The SSRF guard makes webhooks impossible for them.

## Docs vs code disagreements

| Topic | Wiki says | Code does |
|---|---|---|
| Non-2xx response | "a non-2xx response or timeout fails the workflow step" (`webhooks.mdx:574`) | Only a timeout or network error throws. A non-2xx completes the step with `success: false` (`WorkflowExecutionService.ts:1175-1181`). |
| Unsubscribe on bounce/complaint | `contact.unsubscribed` carries `reason: "bounce"|"complaint"` when caused by a bounce or complaint (`webhooks.mdx:459-471`) | Bounce, complaint and suppression handlers set `subscribed: false` directly through Prisma and **track no `contact.unsubscribed` event** (`Webhooks.ts:468-471, 562-565, 601-604, 626-629`). The only `reason` values that are emitted are `snooze` and `snooze_expired` (`ContactService.ts:733-737, 807-809`). |
| Contact event data | "carry no event data by default … `{}`" (`webhooks.mdx:457`) | It is `{}` except for the snooze reasons above. The wiki does not mention snooze. |
| Event name examples | The schema comment shows `"email.opened", "email.clicked"` (`schema.prisma:677`) | The real names are `email.open` and `email.click` (`Webhooks.ts:420`). The activity feed uses `email.opened`. |

## Open questions

1. **Stability contract.** Are `/workflows*`, `/events` and `/activity` meant to be public API? They accept sk keys, and `@plunk/mcp` relies on `/workflows`, but they are absent from `openapi.json`. Ask the Plunk maintainers before treating webhook auto-registration as a supported contract. This is relevant to verified-node review.
2. **Reachability in the test environment.** For the local Docker Plunk, n8n must be reachable at a public IP (tunnel) because of the SSRF guard. Confirm this end to end, including whether a tunnel hostname that resolves to a public IP passes.
3. **Are the docs or the code wrong** on the two disagreements above (non-2xx and bounce-induced `contact.unsubscribed`)? Possibly raise an upstream issue or PR.
4. **Is the default payload enough,** or should the trigger set a custom `body`, for example to add the event name, the contact `id` and the event `id`? Templating is Liquid (`packages/shared/src/template/index.ts:176-178`) and always returns a **string**, so a custom body cannot carry nested objects as JSON objects. Whether a Liquid `json` filter is available was **not verified**. The safer route is the default body plus templated headers or query (`{{id}}`).
5. **Plan limits.** Does Plunk Cloud cap the number of workflows per project? No limit was found in `WorkflowService.create`, but `BillingLimitService` was not read in full.
6. **Duplicates and ordering.** Should the trigger offer built-in deduplication for SNS redeliveries, keyed on `emailId` + event (+ counter)?

## Citations (files read, `useplunk/plunk@37e5f81`)

- `apps/api/src/controllers/Webhooks.ts`: inbound SNS/SES and Stripe handler, email event producers
- `apps/api/src/services/EventService.ts`: `trackEvent`, workflow triggering, reserved names, event queries
- `apps/api/src/services/WorkflowExecutionService.ts`: step engine, `executeWebhook`, `safeFetch`, SSRF guard
- `apps/api/src/services/WorkflowEntryService.ts`: re-entry and throttle rules
- `apps/api/src/services/WorkflowService.ts`, `apps/api/src/controllers/Workflows.ts`: workflow CRUD API
- `apps/api/src/middleware/auth.ts`: sk/pk/JWT auth
- `apps/api/src/controllers/{Actions,Events,Activity,Contacts,Campaigns}.ts`; `apps/api/src/services/{ContactService,SegmentService,ActivityService,EmailService,QueueService}.ts`; `apps/api/src/jobs/{email-processor,workflow-processor-queue,segment-count-processor}.ts`; `apps/api/src/app.ts`
- `packages/db/prisma/schema.prisma`; `packages/shared/src/schemas/index.ts`; `packages/types/src/api/activity.ts`
- `apps/web/src/components/workflow-steps/WebhookStepDialog.tsx`
- `apps/mcp/src/tools/workflows.ts`, `apps/mcp/src/config.ts`: first-party API-key use of `/workflows`
- `apps/wiki/content/docs/guides/{webhooks,receiving-emails}.mdx`, `apps/wiki/content/docs/self-hosting/email-setup.mdx`, `apps/wiki/content/docs/concepts/workflows.mdx`, `apps/wiki/openapi.json` (servers: `https://next-api.useplunk.com`; paths: `/v1/*`, `/contacts`, `/templates`, `/campaigns`, `/segments`, `/emails` only)
