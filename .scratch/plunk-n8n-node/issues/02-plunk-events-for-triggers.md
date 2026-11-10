# Plunk events for triggers

Type: research
Status: resolved
Blocked by: —
Map: [../map.md](../map.md)

## Question

What can an n8n trigger listen to in Plunk? Which events exist (email sent/delivered/opened/clicked/bounced/complained, contact subscribed/unsubscribed, custom events, inbound email), how are they delivered (webhook workflow step, dedicated webhooks, inbound-email webhook), what do payloads look like, is there signature verification, and can webhooks be registered/unregistered via API (needed for n8n's webhook lifecycle) or only by hand in the UI? If no push mechanism fits, which list endpoints would allow polling?

## Answer

Full findings: branch `research/plunk-events-for-triggers`, file `claudedocs/research/02-plunk-events-for-triggers.md` (Plunk `next` @ `37e5f81`).

- **Only push mechanism**: the WEBHOOK step inside a Plunk workflow. `Webhooks.ts` is inbound only (SES via SNS, Stripe). No project-level webhooks, no signing secret in the schema.
- **Event catalogue** (all via `EventService.trackEvent`, which starts matching workflows): `email.sent`, `email.delivery`, `email.open` (bot opens filtered), `email.click`, `email.bounce`, `email.complaint`, `email.received` (inbound); `contact.subscribed`, `contact.unsubscribed` (snooze carries `reason`); `segment.<slug>.entry/.exit` (dynamic tracked segments, 5-min recompute); custom events. **Not existing**: contact created/updated, campaign-level events, `email.failed`.
- **Lifecycle is automatable with `sk_`**: create workflow (`allowReentry: true`) + WEBHOOK step with random secret header + enable; checkExists = `GET /workflows/:id`; delete = disable, cancel executions, `DELETE` (409 while executions are active). `@plunk/mcp` builds workflows the same way. These endpoints are **not** in Plunk's published OpenAPI.
- **Payload**: method configurable (default POST, JSON). Default body `{contact{email,subscribed,data}, workflow{id,name}, execution{id,startedAt}, event{…}}`, with no contact id and no event name; addable via Liquid-templated headers/URL (renders strings only). Authenticity only via the shared-secret header.
- **Delivery constraints**: synchronous, 10 s timeout, no retry; non-2xx still counts as completed; SSRF guard blocks private/loopback (not configurable), so n8n on localhost or a Docker bridge is unreachable and local testing needs a public tunnel; only contact-bearing events start workflows; at-least-once SNS means duplicates and no ordering guarantee.
- **Polling fallback**: `GET /events?eventName=&limit=` (newest first, no cursor/since, uncapped limit); `GET /activity` (cursor + `startDate`, max 100/page, `startDate` ignored once a cursor is passed); `GET /contacts` (newest first, cursor; the only way to catch new contacts); `GET /campaigns?status=`.
- **Docs vs code**: docs say non-2xx fails the step (it doesn't); docs say bounce/complaint emits `contact.unsubscribed` with `reason: "bounce"` (it doesn't).

Open questions handed on:
- Are `/workflows`, `/events`, `/activity` a stable public API? → new ticket "Ask Plunk maintainers about API stability".
- Custom vs default payload, dedup key (`emailId` + event + count), workflow-per-project caps on Cloud → Trigger design.
- End-to-end tunnel setup → Local Plunk test environment.
- Reporting the two doc/code mismatches upstream → folded into the maintainers ticket.
