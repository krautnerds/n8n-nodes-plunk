# Local Plunk test environment

Type: task
Status: open
Blocked by: —
Map: [../map.md](../map.md)

## Question

Bring up a self-hosted Plunk locally via Docker (the repo's `docker-compose.yml` / `.env.self-host.example`) so later tickets can verify API behaviour and trigger delivery against a real instance. Record: how to start/stop it, URLs, where the API keys are stored (never in the repo), and any limits of the self-hosted setup (e.g. email sending without SES).

## Notes from research

- Plunk's webhook SSRF guard blocks private/loopback targets and can't be disabled, so trigger delivery to a local n8n needs a **public tunnel** (e.g. cloudflared/ngrok). Verify the setup end to end.
- Confirm what `/projects/*` returns when called with an API key (inferred to fail, untested).
