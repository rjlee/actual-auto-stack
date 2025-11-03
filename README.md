Stack usage

- Copy `stack/.env.example` to `stack/.env` and set values:
  - `ACTUAL_API_MAJOR=25` (or desired major)
  - Optional: uncomment port variables to expose services on the host
- Place service-specific `.env` files under `stack/env/` (copy the provided `*.env.example` to `*.env` and fill values):
  - `stack/env/actual-events.env`
  - `stack/env/actual-auto-categorise.env`
  - `stack/env/actual-auto-reconcile.env`
  - `stack/env/actual-investment-sync.env`
  - `stack/env/actual-landg-pension.env`
  - `stack/env/actual-monzo-pots.env`
  - `stack/env/actual-tx-linker.env`
  - Each example includes common keys for that service (ports, scheduling, UI auth, provider creds); leave blank to inherit from the shared `stack/.env`.
- Start from the repo root:
  - `docker compose -f stack/docker-compose.yml --env-file stack/.env up -d`

Single-service runs (profiles)

- Each service is assigned a compose profile:
  - events → actual-events
  - categorise → actual-auto-categorise
  - reconcile → actual-auto-reconcile
  - invest → actual-investment-sync
  - landg → actual-landg-pension
  - monzo → actual-monzo-pots
  - linker → actual-tx-linker
- Run just one service (example: events):
  - `docker compose -f stack/docker-compose.yml --env-file stack/.env --profile events up -d actual-events`
- Bring down only that service:
  - `docker compose -f stack/docker-compose.yml --env-file stack/.env --profile events down`

Notes

- Startup order: `actual-events` defines a healthcheck and other services declare `depends_on: condition: service_healthy`, so they wait for `actual-events` to be healthy before starting.
- Ports are enabled by default and read from `stack/.env` (`EVENTS_HTTP_PORT`, `CATEGORISE_HTTP_PORT`, `INVEST_SYNC_HTTP_PORT`, `LANDG_HTTP_PORT`, `MONZO_HTTP_PORT`). Adjust as needed.
- All images use `api-${ACTUAL_API_MAJOR}` so switching API majors is a one-line change in `stack/.env`.

Data layout

- Each service mounts a single bind under this folder: `./state/<service-name> -> /app/data`.
- The Actual budget cache lives inside that mount at `/app/data/budget` (set via `BUDGET_DIR=/app/data/budget`).
- Example on disk after running:
  - `stack/state/actual-events/...`
  - `stack/state/actual-auto-categorise/budget/...` and model/state in `stack/state/actual-auto-categorise/...`
  - `stack/state/actual-auto-reconcile/budget/...`
  - `stack/state/actual-investment-sync/budget/...`
  - `stack/state/actual-landg-pension/budget/...`
- `stack/state/actual-monzo-pots/budget/...`

Best practices

- Keep one budget cache per service (the default). Do not share budget directories between services to avoid concurrent-write issues.
- Manage app configuration in each repo’s `.env` (under `../<repo>/.env`); the stack only controls image tags, startup order, and where data is persisted.

Environment variables

- Each service loads two env files in this order:
  - The shared stack file: `./.env` (defaults common to all services)
  - The service-specific file in this folder: `./env/<service>.env` (overrides shared for that service)
  - Because the service-specific file is listed last, its values override the shared defaults when keys overlap.
- Variables defined inline under `environment:` in `stack/docker-compose.yml` take precedence over values from any `env_file`.
- Shared keys you can set once in `stack/.env` and reuse across all services:
  - `ACTUAL_SERVER_URL`
  - `ACTUAL_PASSWORD`
  - `ACTUAL_SYNC_ID`
  - `NODE_TLS_REJECT_UNAUTHORIZED` (set to `0` only if you accept insecure certificates)

Example `stack/.env` (fill with your values):

```
ACTUAL_API_MAJOR=25
ACTUAL_SERVER_URL=https://your-actual-server:5006/
ACTUAL_PASSWORD=...
ACTUAL_SYNC_ID=...
NODE_TLS_REJECT_UNAUTHORIZED=0
```
