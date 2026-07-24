# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Grok2API is a multi-account API gateway for **Grok Build**, **Grok Web**, and **Grok Console**. The Go backend exposes OpenAI-style endpoints (`/v1/*`), Anthropic Messages, and an admin API; the React SPA is the management console. Production serves both from one Go process (static frontend embedded via `frontend.staticPath`).

Providers:

| Provider | Code | Auth | Role |
| --- | --- | --- | --- |
| Grok Build | `grok_build` | Device OAuth / OAuth JSON | Native Responses, billing, dynamic model sync |
| Grok Web | `grok_web` | SSO JSON / line tokens | Chat/Responses/Messages, images, video |
| Grok Console | `grok_console` | SSO JSON / line tokens | Stateless Responses only (`store: false`) |

Public model names have no provider prefix (e.g. `grok-4.5`). Internal routes use `Build/`, `Web/`, `Console/` namespaces. Client API keys use the `g2a_` prefix.

## Local environment constraints (sean.su workstation)

This machine is on a **company intranet** without admin privileges. Assume:

- **No elevated install/config** — do not require admin rights, system-wide installs, or privileged ports.
- **All outbound internet must go through the corporate proxy:**
  - HTTP/HTTPS proxy: `http://172.17.45.120:7890`
  - For shell/tools that need the net, set before running (PowerShell example):

```powershell
$env:HTTP_PROXY = "http://172.17.45.120:7890"
$env:HTTPS_PROXY = "http://172.17.45.120:7890"
$env:http_proxy = "http://172.17.45.120:7890"
$env:https_proxy = "http://172.17.45.120:7890"
# optional if something ignores HTTP(S)_PROXY:
$env:ALL_PROXY = "http://172.17.45.120:7890"
```

  - Also apply proxy awareness to `go`, `pnpm`/`npm`, `docker` pulls, `curl`, package downloads, and any browser automation. Localhost targets (`127.0.0.1`, `localhost`) should usually bypass the proxy when the tool supports `NO_PROXY`.
  - Suggested bypass for local services:

```powershell
$env:NO_PROXY = "localhost,127.0.0.1,::1"
$env:no_proxy = "localhost,127.0.0.1,::1"
```

- **Browser for manual UI testing:** use **Microsoft Edge only**.
  - Always open a **separate, dedicated Edge profile** (new/isolated user-data directory), not the daily work profile. This keeps SSO cookies, admin sessions, and corporate plugins from contaminating test runs.
  - Prefer launching Edge with an isolated profile directory, for example:

```powershell
# Use a project-local profile dir so tests stay isolated
$edgeProfile = Join-Path $PWD ".tmp/edge-profile-grok2api"
New-Item -ItemType Directory -Force -Path $edgeProfile | Out-Null
& "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe" `
  --user-data-dir="$edgeProfile" `
  --no-first-run `
  --new-window `
  "http://127.0.0.1:8000"
```

  - If Edge is installed under `Program Files` instead of `Program Files (x86)`, adjust the path. Do not reuse the default interactive profile.

When verifying admin UI, OAuth/SSO flows, or production-like browser behavior, follow the proxy + isolated Edge profile rules above first to avoid auth/network dead ends.

**Portable runbook (config only, no code changes):** see [docs/local-run.md](docs/local-run.md). It parameterizes proxy/listen/Edge profile so the same steps work on intranet, home, or server hosts. Prefer that checklist before changing gateway source for proxy adaptation.

## Common commands

### Root

```bash
# Run backend (uses root config.yaml; GOCACHE under .gocache)
make run
# or with explicit config
make run CONFIG=/path/to/config.yaml

# Regenerate Swagger from handler annotations (must stay committed)
make swagger
```

### Backend (`backend/`)

```bash
# Start (default config: ../config.yaml or ./config.yaml)
go run ./cmd/grok2api
go run ./cmd/grok2api --config /path/to/config.yaml --listen 0.0.0.0:8000

# Test / lint / build
go test ./...
go test -race ./...
go vet ./...
go build ./cmd/grok2api

# Single package
go test ./internal/application/gateway/

# Single test
go test ./internal/application/gateway/ -run TestSelectorSelectsHealthyAccount -count=1

# Optional integration tests (skipped unless env is set)
TEST_POSTGRES_DSN='postgres://...' go test ./internal/infra/persistence/relational/ -run TestPostgres
TEST_REDIS_ADDRESS='127.0.0.1:6379' go test ./internal/infra/runtime/redis/ -run TestRedis
```

Module path: `github.com/chenyme/grok2api/backend` (Go 1.26).

### Frontend (`frontend/`)

Package manager is **pnpm** (`packageManager: pnpm@11.5.2`).

```bash
pnpm install
pnpm install --frozen-lockfile   # CI-style
pnpm dev                         # http://127.0.0.1:5173, proxies /api /v1 /healthz /readyz → :8000
VITE_DEV_API_TARGET=http://127.0.0.1:9000 pnpm dev
pnpm lint
pnpm build                       # tsc -b && vite build → dist/
pnpm preview
```

There is no frontend unit-test script; verification is lint + build.

### Docker

```bash
cp config.example.yaml config.yaml   # fill secrets + bootstrapAdmin
docker compose up -d
docker compose logs -f grok2api
```

Image builds frontend then backend; entrypoint copies mounted config to `/app/config.yaml` and drops privileges.

### CI parity (`.github/workflows/ghcr-image.yml`)

1. `go test ./...` and `go vet ./...` in `backend/`
2. `make swagger` then `git diff --exit-code` on `backend/docs/*`
3. Frontend: `pnpm install --frozen-lockfile`, `pnpm lint`, `pnpm build`

If you change public API swagger annotations under `cmd/grok2api` or `internal/transport/http`, run `make swagger` and commit `backend/docs/docs.go`, `swagger.json`, `swagger.yaml`.

## Configuration split

Root `config.yaml` (from `config.example.yaml`, gitignored) holds **startup** config only:

- `server` listen / timeouts / swagger
- `secrets` (`jwtSecret`, `credentialEncryptionKey` — latter is permanent AES key for stored credentials)
- `bootstrapAdmin` (only until first admin exists)
- `database` sqlite | postgres
- `runtimeStore` memory | redis
- `media` local path
- `frontend.staticPath`

Runtime knobs (provider URLs/UA, batch concurrency, routing cooldowns, media limits, audit buffers, client-key defaults, egress, etc.) live in the DB and are edited via admin **Settings**. Most apply hot; fields marked restart-required do not. Multi-instance uses Redis for settings change notification, rate limits, concurrency leases, sticky sessions, locks, and quota recovery queues.

Recommended combos: SQLite+Memory (single node), Postgres+Redis (multi-node). Media binaries are on disk; DB stores metadata only.

Do not commit `config.yaml`, OAuth/SSO exports, HAR/pcap, or credential dumps (see `.gitignore`).

## Architecture

```
Client / Admin SPA
       │
       ▼
  Gin HTTP (transport/http)
   /healthz /readyz
   /v1/*           client API key (g2a_)
   /api/admin/v1/* admin JWT cookies/tokens
   static SPA fallback
       │
       ▼
  Application services
   gateway | account | model | clientkey | settings | media | audit | ...
       │
       ▼
  Domain models + repository interfaces
       │
       ├── relational (GORM SQLite/Postgres)
       ├── runtime (memory | redis)
       ├── provider adapters (cli=Build, web, console)
       └── egress manager (HTTP/SOCKS proxy pool, TLS client)
```

### Backend layout

| Path | Role |
| --- | --- |
| `cmd/grok2api` | Process entry; swagger title annotations |
| `internal/cli` | Flags (`--config`, `--listen`), signal handling |
| `internal/app` | DI / lifecycle: DB, providers, services, HTTP server, startup recovery |
| `internal/application/*` | Use cases (gateway is the request orchestrator) |
| `internal/domain/*` | Pure domain types and rules |
| `internal/repository` | Persistence/runtime interfaces only |
| `internal/infra/*` | Config, GORM repos, providers, runtime stores, security, media, egress |
| `internal/transport/http/*` | Gin handlers + middleware; thin protocol layer |
| `internal/pkg/*` | Shared utilities (batch pools, caches, signed URL policy) |

Dependency direction: **Transport → Application → Domain**. Infrastructure implements `repository` and `provider` interfaces; domain must not import HTTP/DB/provider packages.

### Request path (inference)

1. `transport/http/inference` parses OpenAI/Anthropic-shaped bodies and calls `gateway.Service`.
2. Gateway resolves public model → enabled internal route(s), checks client-key model permissions and provider capability surfaces.
3. `selector` picks a leased account from the provider pool (priority, concurrency, quota/billing, cooldown, sticky/prompt-cache affinity).
4. Provider adapter (`infra/provider/{cli,web,console}`) talks upstream via egress.
5. Failures classified in `gateway/failure.go`; retries stay **within the chosen provider** after source selection.
6. Audit/usage finalized asynchronously; media jobs go through `application/media` + local store.

Core gateway files: `service.go`, `selector.go`, `attempt.go`, `failure.go`, `image.go`, `video.go`.

### Providers

Adapters register static `Definition`s (catalog kind, quota kind, conversation/media surfaces, credential surface). `provider.Registry` is the capability switchboard (`SupportsConversation`, `Responses`, `Quota`, `ImageGeneration`, etc.).

- **Build (`cli`)**: OAuth refresh, remote model list, billing quota, stored responses.
- **Web**: SSO (no auto-renew), remote quota windows, Statsig headers, images/video.
- **Console**: SSO, static catalog, always stateless Responses; no `previous_response_id` / get-delete / compact ownership.

Web→Console SSO reuse and Web→Build conversion are first-class account workflows under `application/account`.

### Frontend layout

Feature-sliced SPA:

- `src/app` — router, auth boundary, deferred page loading
- `src/features/*` — pages (accounts, models, client-keys, audits, settings, media, docs, dashboard)
- `src/entities/*` — shared DTOs / API helpers
- `src/shared/api` — fetch client, admin session refresh, typed decoders
- `src/components/ui` — shadcn/Radix primitives

Alias `@` → `src/`. All admin traffic goes through `shared/api`; TanStack Query owns server state. Frontend never reads YAML; public runtime info comes from backend-controlled endpoints.

## Working notes specific to this repo

- Prefer matching existing package boundaries: new persistence methods go on `repository` interfaces + `infra/persistence/relational`; new upstream protocol logic stays in the relevant provider adapter, not in handlers.
- Account credentials are AES-encrypted at rest; tests should use the security cipher helpers or already-encrypted fixtures rather than inventing plaintext token storage.
- Media post-processing errors (`provider.MediaPostProcessingError`) must not rotate accounts or penalize health — generation already succeeded upstream.
- Console must remain stateless: do not invent response ownership rows for Console routes.
- Startup phases gate `/v1` traffic (`service_reconciling`) until credential/quota recovery finishes; readiness is multi-component JSON on `/readyz`.
- Chinese UI/API error strings and comments are intentional product language; keep new user-facing messages consistent with surrounding code.
