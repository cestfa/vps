# CLAUDE.md

Context and conventions for AI-assisted work on this repo.

## What this repo is

A self-hosted Docker stack for a personal VPS. All services run behind Caddy (reverse proxy + automatic TLS). One [.env](file:///home/ceest/dev/vps/.env) file drives the entire stack.

## Repo structure

```
compose.yaml          # Root: includes all apps via `include:`
.env.example          # Template — copy to .env and fill
.gitattributes        # Enforces LF line endings
.gitignore
CLAUDE.md
README.md
apps/
  <service>/
    compose.yaml
    Caddyfile         # caddy only
    config.json       # honey only
    config.toml       # mediaflow-proxy only
```

Data is persisted under `$DOCKER_DATA_DIR` (default `/opt/docker/data`), never in the repo.

## Network

All services share `vps_network` (defined in root [compose.yaml](file:///home/ceest/dev/vps/compose.yaml)). Services talk to each other by container name.

Only Caddy and AdGuard expose host ports (`80`, `443`, `53`, `853`). All other services rely on Docker network service resolution — never use `ports:` or `expose:`.

WireGuard clients use `WG_DEFAULT_DNS=10.8.0.1` (the WireGuard gateway IP) so all DNS from VPN clients is routed through AdGuard Home on the VPS. AdGuard must be configured to listen on `0.0.0.0` (not just `127.0.0.1`) for this to work — the default post-wizard config already does this.

Beszel Agent uses `network_mode: host` and communicates with the hub via a Unix socket volume (`beszel-socket`).

**Never define a separate network inside a per-app compose fragment.** Everything shares `vps_network` via the root compose default.

## Dispatcharr

Dispatcharr runs as a single container `dispatcharr` in All-in-One (`aio`) mode (`DISPATCHARR_ENV=aio`). It manages the Django web application, background celery workers, and broker internally. It persists data at `$DOCKER_DATA_DIR/dispatcharr`.

## MediaFlow + Warp

MediaFlow routes all traffic through Warp (`caomingjun/warp`) so debrid add-ons (Torrentio, etc.) aren't blocked by debrid providers.

- Warp exposes an HTTP proxy on port `1080`
- Per-domain proxy routing is configured explicitly inside `apps/mediaflow-proxy/config.toml`
- Warp requires `cap_add: NET_ADMIN` and the `src_valid_mark` sysctl
- MediaFlow uses `depends_on: [warp]` — starts after Warp container starts
- `warp-data` volume persists the registration across restarts

## Plex

Plex runs purely behind Caddy (`vps_network`, no `ports:`/`expose:`) since the VPS already has a public IP — there's no router/NAT for Plex's own UPnP-based remote access to configure, so `ADVERTISE_IP=https://${PLEX_HOSTNAME}:443` tells Plex its externally reachable URL instead. `PLEX_CLAIM` is a one-time token (https://plex.tv/claim, 4-minute expiry) only needed on first launch to link the server to an account; it can be left blank afterwards.

## Caddy

The Caddyfile defines a `(common)` snippet applied to every vhost:

```
(common) {
    encode zstd gzip
}
```

HTTP/3 (QUIC) is enabled globally. All subdomains are derived from [.env](file:///home/ceest/dev/vps/.env) variables and injected via `environment:` in the Caddy compose fragment.

## Conventions

- `logging` is handled globally by `daemon.json`; custom `stop_grace_period` is used only when overriding the default `10s`.
- `restart: always` on critical services (Caddy, Warp, AdGuard, MediaFlow, Dispatcharr, WG-Easy)
- `restart: unless-stopped` on personal tools (Ghostfolio, Wallos, Honey, Stirling-PDF, etc.)
- No hardcoded secrets — [.env](file:///home/ceest/dev/vps/.env) only
- `image: ...:latest` is intentional; Watchtower handles updates
- No `deploy.resources` limits — let the host scheduler manage allocation
- **Alphabetical Sorting:** Keep [.env](file:///home/ceest/dev/vps/.env)/[.env.example](file:///home/ceest/dev/vps/.env.example) subdomains, root [compose.yaml](file:///home/ceest/dev/vps/compose.yaml) includes, [apps/caddy/compose.yaml](file:///home/ceest/dev/vps/apps/caddy/compose.yaml) environment variables, [apps/caddy/Caddyfile](file:///home/ceest/dev/vps/apps/caddy/Caddyfile) reverse proxy blocks, and [apps/honey/config.json](file:///home/ceest/dev/vps/apps/honey/config.json) services list sorted alphabetically.

## Healthchecks

Healthchecks are added where a reliable, officially-documented or universally-standard probe exists. Do not invent endpoints — verify against the upstream repo or Docker Hub before adding.

| Service | Test command | Notes |
|---|---|---|
| `caddy` | `caddy validate --config /etc/caddy/Caddyfile` | Uses caddy binary itself — no curl/wget needed |
| `adguard` | `wget -qO- http://localhost:80` | Port 80 = post-setup; port 3000 during wizard |
| `dispatcharr` | `curl -fs http://localhost:9191/health/` | Official `/health/` endpoint |
| `ghostfolio` | `curl -f http://localhost:3333/api/v1/health` | Official unauthenticated endpoint |
| `ghostfolio-db` | `pg_isready -U $POSTGRES_USER -d $POSTGRES_DB` | Universal postgres standard |
| `ghostfolio-redis` | `redis-cli -a $REDIS_PASSWORD --no-auth-warning ping \| grep PONG` | Auth-aware; REDIS_PASSWORD exposed via `environment:` |
| `aiometadata-redis` | `redis-cli ping` | No auth on this instance |
| `beszel` | `/beszel health --url http://localhost:8090` | Official binary subcommand (distroless image) |
| `beszel-agent` | `/agent health` | Official binary subcommand (distroless image) |
| `filebrowser` | `wget --no-verbose --tries=1 --spider http://localhost:80/` | Alpine image — wget available |
| `plex` | `curl -fs http://localhost:32400/identity` | Official unauthenticated identity endpoint |
| `warp`, `honey`, `uptime-kuma`, `watchtower`, `wallos`, `aiostreams`, `stirling-pdf`, `aiometadata`, `wg-easy` | — | Already healthy via image built-in HEALTHCHECK |
| `mediaflow-proxy` | — | No healthcheck: `/health` endpoint exists but tool availability in minimal Rust image is unverified |

When adding a healthcheck to a service that has dependencies (e.g. Redis, Postgres), also upgrade `depends_on` to use `condition: service_healthy` so the dependent service waits for a ready backing service.

## What to avoid

- `ports:` on internal services
- Separate networks in per-app compose files
- Docker socket mounted read-write except on Watchtower
- Data persisted inside containers
- **Running `docker compose` locally** — this repo is deployed on a remote VPS. Never run `docker compose up`, `docker compose down`, `docker compose pull`, or any other `docker compose` command from this machine. All stack operations must be performed on the VPS.

## Common operations

```bash
# Start
docker compose up -d

# Status + memory
docker compose ps
docker stats --no-stream

# Reload Caddy
docker compose exec -w /etc/caddy caddy caddy reload

# Logs
docker compose logs -f <service>

# Update all images
docker compose pull && docker compose up -d

# Verify Warp tunnel
docker compose exec warp curl -s https://cloudflare.com/cdn-cgi/trace | grep warp
```
