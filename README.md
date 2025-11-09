# actual-auto-stack

Docker Compose bundle for the `actual-*` sidecars. Brings the services up against a shared Actual Budget server with consistent configuration, health checks, and persistent state.

## Features

- Central `.env` for shared Actual credentials, image tags, and port assignments.
- Per-service overrides in `env/*.env` plus include snippets for optional services.
- Health-aware startup using container `HEALTHCHECK`s and `depends_on` conditions.
- Persistent volumes under `state/<service>` with isolated budget caches.

## Requirements

- Docker Engine and Docker Compose v2.20+ (required for `include` support).
- Access to an Actual Budget server (`ACTUAL_SERVER_URL`, `ACTUAL_PASSWORD`, `ACTUAL_SYNC_ID`).
- Image tags published by [`actual-auto-ci`](https://github.com/rjlee/actual-auto-ci) (set via `ACTUAL_IMAGE_TAG`).

## Installation

```bash
git clone https://github.com/rjlee/actual-auto-stack.git
cd actual-auto-stack
```

### Docker quick start

```bash
cp .env.example .env
for f in env/*.env.example; do cp "$f" "${f%.example}"; done
# (Optional) enable extra services
for f in includes/*.yml.example; do cp "$f" "${f%.example}"; done

docker compose --env-file .env up -d
```

## Configuration

- `.env` – shared defaults (Actual credentials, image tag, ports).
- `env/actual-auto-auth.env` – optional environment overrides consumed by the shared auth gateway (use it to define landing-page shortcuts like `/investment/` when you enable extra UIs).
- `env/*.env` – service-specific overrides (copy from `.env.example` siblings). For example, `env/actual-auto-categorise.env` controls the auth headers (`CATEGORISE_LOGIN_NAME`, `CATEGORISE_AUTH_COOKIE_NAME`) that Traefik sends to the shared gateway, and the root `.env` file now holds the auth gateway defaults (image tag, session secret, cookie name).
- `includes/*.yml` – optional Compose fragments; copy the `.example` files you need and reference them with `docker compose -f docker-compose.yml -f includes/<service>.yml up`.

Precedence: values supplied via `docker compose ... --env-file` > per-service env files > `.env`.

| Setting                       | Description                                                | Default                                 |
| ----------------------------- | ---------------------------------------------------------- | --------------------------------------- |
| `ACTUAL_IMAGE_TAG`            | Image tag applied to all services (matches API version)    | unset (`latest`)                        |
| `ACTUAL_SERVER_URL`           | Actual Budget server URL                                   | required                                |
| `ACTUAL_PASSWORD`             | Actual server password                                     | required                                |
| `ACTUAL_SYNC_ID`              | Budget sync ID                                             | required                                |
| `ENABLE_EVENTS`               | Toggle event-driven behaviour across services              | unset                                   |
| `EVENTS_URL`                  | Shared `actual-events` endpoint for SSE subscribers        | unset                                   |
| `EVENTS_AUTH_TOKEN`           | Bearer token for secured SSE endpoints                     | unset                                   |
| `STACK_HTTP_PORT`             | Host port exposed by the shared Traefik proxy              | `3000`                                  |
| `EVENTS_HTTP_PORT`            | Internal port for `actual-events` (still defaults to 3000) | `3000`                                  |
| `AUTH_FORWARD_IMAGE`          | Auth gateway image tag (defaults to GHCR latest)           | `ghcr.io/rjlee/actual-auto-auth:latest` |
| `CATEGORISE_LOGIN_NAME`       | Login heading injected via Traefik header                  | `Actual Auto Categorise`                |
| `CATEGORISE_AUTH_COOKIE_NAME` | Cookie name shared between the UI and auth gateway         | `categorise-auth`                       |
| `STACK_HOME_TITLE`            | Heading shown on the authenticated home page (`/`)         | `Actual Automation Stack`               |
| `STACK_NAV_LINKS`             | Extra stack links (format `Label|/path;Another|https://…`) | unset                                   |
| `STACK_INCLUDE_DEFAULT_LINKS` | Set to `false` to drop built-in Traefik/Categorise links   | `true`                                  |
| `STACK_INCLUDE_CATEGORISE_LINK` | Toggle the Categorise shortcut without disabling defaults | `true`                                  |

The Compose file automatically exports `STACK_LINK_*` / `STACK_LABEL_*` pairs for bundled services (e.g. Traefik dashboard, Categorise UI). Add your own by defining similar variables in `.env` or optional include files so they appear on the auth home page.

Refer to each service README for additional keys exposed via `env/<service>.env`.

## Usage

### Common commands

```bash
# Launch core services
docker compose --env-file .env up -d

# Include optional services
docker compose --env-file .env -f docker-compose.yml -f includes/actual-monzo-pots.yml up -d

# Check status / logs
docker compose --env-file .env ps
docker compose --env-file .env logs -f actual-events

# Stop everything
docker compose --env-file .env down
```

Each service stores data under `state/<service>`. Budget caches live in `state/<service>/budget`; other artefacts (tokens, models, mappings) share the same root.

### Accessing the UIs

- `http://<host>:${STACK_HTTP_PORT:-3000}/` – Authenticated home page with shortcuts to enabled services (links auto-generated from `STACK_LINK_*`, `STACK_NAV_LINKS`, and the defaults above).
- `http://<host>:${STACK_HTTP_PORT:-3000}/dashboard/` – Traefik dashboard (lists routers/services and is secured by the shared auth gateway).
- `http://<host>:${STACK_HTTP_PORT:-3000}/categorise/` – Actual Auto Categorise UI.
- Add additional path-based routers for optional UIs using the include files (e.g., `/investment/`, `/landg/`, `/monzo/`). Set `STACK_LINK_*`/`STACK_LABEL_*` so the home page picks them up automatically.

## Image tags

- Set `ACTUAL_IMAGE_TAG` (e.g. `26.1.0`) to pin every service to a specific `@actual-app/api` release.
- Leave `ACTUAL_IMAGE_TAG` empty to follow `latest` (highest supported API version published by `actual-auto-ci`).

Individual service READMEs describe additional tags and compatibility notes.

## License

MIT © contributors.
