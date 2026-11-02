# Deployment Model — Vibe Kanban

Upstream: https://github.com/BloopAI/vibe-kanban @ `7c3807084daad0d9d315064f60fa869de254f027`
(verified from upstream source, 2026-11-02). Stack = two independently
deployable groups, each an own component under `app/` (own `stackbuilder.toml`,
`components/{base,environments,extensions}`, `build/`, `images/`):

- Group 1 `app/remote/` — cloud server (API + web + Electric sync + DB). Today's
  `app/` content moves here unchanged (remediation R1).
- Group 2 `app/relay/` + `app/host/` — relay-server and a headless local host
  with agents. Deployed as its own compose projects, separate from group 1.

## Product parts

| part | group | purpose | image source |
|------|-------|---------|--------------|
| remote-server | 1 | `crates/remote` axum API (`:8081`, `/v1/health`), embeds remote web (build arg `VITE_RELAY_API_BASE_URL` = public relay URL); runs DB migrations (incl. relay tables) | source-built (`app/remote/images/`) |
| electric | 1 | Electric sync service in front of the remote DB (needs `wal_level=logical`) | published |
| remote-db | 1 | Postgres for remote (database extension pair, never base) | extension |
| relay-server | 2 | `crates/relay-tunnel` bin `relay-server` (`:8082`, `/health`), WebSocket tunnel broker host↔browser | source-built (`app/relay/images/`) |
| host-server | 2 | root `Dockerfile` bin `server` (axum, `HOST`/`PORT`/`PREVIEW_PROXY_PORT`, `/health`), embeds `local-web` UI, runs agent sessions in worktrees | source-built (`app/host/images/`, executors layer added) |
| executors | 2 | coding-agent CLIs the host spawns (many are started via `npx` ⇒ Node needed). NOT in the upstream image (only git, ssh, tini, wget) | added inside the host image (no upstream patch) |

Not parts: Tauri desktop app, npx-cli, mobile docs, `review` worker (external,
`REVIEW_WORKER_BASE_URL`), Caddy (our environments overlays replace it).

## Interaction map

- host-server → remote-server: login (`/api/auth/local/login` on host proxies to remote
  `/v1/...`), token refresh, org/project data. Base URL is runtime env
  `VK_SHARED_API_BASE` (env first, compile-time fallback — no rebuild needed).
- host-server → relay-server: after login and `relay_enabled` (default true),
  `relay_registration.rs` opens `GET /v1/relay/connect` and reconnects forever.
  Base URL = runtime env `VK_SHARED_RELAY_API_BASE`.
- browser (remote web) → relay-server: `/v1/relay/create/{host_id}`,
  `/v1/relay/h/{host_id}/s/{session}/…`; origin = build-time
  `VITE_RELAY_API_BASE_URL`. Relay also needs wildcard `*.relay.<domain>` routing.
- relay-server → **remote DB** (and shares `VIBEKANBAN_REMOTE_JWT_SECRET`): it reads/writes
  tables `hosts`, `auth_sessions`, `relay_browser_sessions`, `users`. Schema
  and migrations live ONLY in `crates/remote/migrations` (FKs to `users`,
  `organizations`, `auth_sessions`) — the relay has NO own schema. Absent
  remote DB ⇒ relay cannot start.
- remote-server → electric → remote-db (realtime sync). Absent electric ⇒ web UI
  has no live data.
- Pairing (SPAKE2): the one-time code is generated on the HOST
  (`POST /api/relay-auth/server/enrollment-code`, host UI Settings → Remote
  Access; refused over relay), entered in the remote web (Remote Access → Link a
  host → client-side `/api/relay-auth/client/pair`; verify at gate D which server
  serves the client route).
  Without pairing a host stays offline for browsers.
- Host absent ⇒ remote still works (boards) but no workspaces/agents; relay
  absent ⇒ host works locally, no remote access.

## Functional base

Group 1: remote-server, electric (+ remote-db). Group 2: relay-server,
host-server (with executors). Each part is exercised by the gate (C).
Mandatory infra:

- Database: postgresql+pgrust pair — remote-db for group 1. **The relay shares
  this DB** (hypothesis-confirmed from source), so group 2 needs network
  reach to group 1's DB: `RELAY_DATABASE_URL`/`SERVER_DATABASE_URL` env +
  external docker network or published DB port. Own relay DB is NOT possible
  without patching upstream (schema owned by remote) — rejected.
- OIDC: **REFUSED** — remote supports only GitHub/Google OAuth and bootstrap
  local auth (`SELF_HOST_LOCAL_AUTH_EMAIL/PASSWORD`); no generic OIDC provider
  exists upstream; supporting Kanidm requires patching upstream. Base auth =
  local bootstrap auth with random credentials.
- S3: attempted — remote accepts S3-API storage via `R2_ACCESS_KEY_ID`,
  `R2_SECRET_ACCESS_KEY`, `R2_REVIEW_ENDPOINT`, `R2_REVIEW_BUCKET`
  (aws-sdk-s3, custom endpoint ⇒ RustFS). Used only for code-review payloads;
  issue attachments are Azure-only. Gate observes startup with R2 configured.

REFUSED items: OIDC (reason above). Nothing else dropped.

## Base stage plan

Naming: services `<stack>-<role>` per group (`vibe-kanban-remote`,
`vibe-kanban-electric`, `vibe-kanban-db`, `vibe-kanban-relay`,
`vibe-kanban-host`). Secrets random in runtime `.env`.

- Stage A — group 1 remote (+db, electric), step-ca variant first.
  Gate: healthy `/v1/health`; web opens; local login with bootstrap creds;
  create org + project + issue, reload, data persists (Electric live).
- Stage B — group 2 relay. Gate: `GET /health` → `{"status":"ok"}` and
  authenticated `/v1/relay/connect` rejects without token (401). (The
  `/api/relay-auth/*` path is NOT served by the relay — it is host API.)
  Requires Stage A DB reachable and same JWT secret.
- Stage C — group 2 host. Gate: container healthy (`/health` on PORT), preview
  proxy port bound, `POST /api/auth/local/login` succeeds,
  `/api/auth/status` logged in, relay log "connect" with `relay_enabled`.
- Stage D — functional gate scenario (pairing E2E): log in host, generate code
  on host UI, remote web → Link a host with code, host listed online, open
  its workspaces, start a task with an installed executor and see output via
  the relay. Also verifies `VITE_RELAY_API_BASE_URL` build arg.
- Stage E — pgrust swap for remote-db (default if clean), re-run A+D.
- Stage F — S3 (RustFS `R2_*` fragment), re-run A.
- OIDC: no stage (refused).

## Extension catalog

| extension | type | status | dependencies |
|-----------|------|--------|--------------|
| letsencrypt / forwarding / step-ca env (per component) | infra | implemented (group 1) / available (group 2) | environments |
| database postgresql, pgrust | infra | implemented (group 1) | — |
| oidc | infra | refused | — |
| s3-storage (R2_* review payloads) | infra | available (mandatory attempt, stage F) | RustFS |
| azure-attachments (issue attachments) | product | available | Azure account |
| github-oauth / google-oauth login | product | available | OAuth apps |
| github-app (PR review, webhooks) | product | available | GitHub App, review worker |
| loops-email (invites) | product | available | Loops API key |
| extra executor CLIs in host image | product | available | node, agent API keys |
| analytics (PostHog/Sentry build args) | product | available | keys |
| multiple hosts / host scaling | product | available | host group |

## Open hypotheses

- Relay shares remote DB (high, source+migrations). Gate B/D must observe the
  relay reading `hosts` rows created by host registration.
- Wildcard `*.relay.<domain>` routing/cert is needed for browser sessions
  (medium; docs) — step-ca/letsencrypt overlay must provide it; gate D observes.
- Executors needing Node/`npx` and credentials (medium): D runs one task with
  the cheapest installable executor; image layer decides which.
- `VK_SHARED_API_BASE` runtime override sufficient (high,
  `local-deployment/src/lib.rs`); gate C observes remote features enabled.
- Pairing code is generated on the host UI, not in remote UI (high, docs +
  route table) — scenario D follows this.
- R2 usable for RustFS without path-style quirks (low); gate F observes.
- Relay DB credentials: dedicated Postgres role with only needed privileges
  is not provided upstream (low); plain remote DB owner used.

## Remediation (gap vs current stack)

- R1: `app/` → `app/remote/`; add `app/relay/`, `app/host/` (each own
  `stackbuilder.toml`, `components/`, `build/`, `images/`); `.gitkeep` in `app/`.
- R2: rename services/containers to `<stack>-<role>` (`vibe-kanban` →
  `vibe-kanban-remote`), `container_name` = service name.
- R3: add images `vibe-kanban-relay` (crates/relay-tunnel) and
  `vibe-kanban-host` (root Dockerfile + Node/executors layer), pinned commit.
- R4: cross-group wiring: shared JWT secret and DB URL via env.
