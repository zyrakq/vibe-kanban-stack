# 🎯 Vibe Kanban Remote Server

A modular Docker Compose configuration system for self-hosting the Vibe Kanban
Cloud (remote) variant — multi-user Kanban for AI coding agents with local
email/password authentication (optional GitHub/Google OAuth), PostgreSQL
persistence, and ElectricSQL real-time sync to the browser client.

## 🚀 Quick Start

### 0. Build the source image (once)

No upstream Docker image is published, so the remote-server image is built
from source (Rust nightly + pnpm; 10–15 min, 4+ GB RAM, BuildKit required):

```bash
docker build -t localhost/vibe-kanban-remote-server:0.1.45 images/vibe-kanban-remote-server
```

### 1. Build Configurations

```bash
sb build
```

This will create all environments in the `build/` directory based on
[`stackbuilder.toml`](stackbuilder.toml).

### 2. Choose Your Configuration

```bash
# For development with port forwarding
cd build/forwarding/postgresql/   # or pgrust

# For production with Let's Encrypt SSL
cd build/letsencrypt/postgresql/

# For production with Step CA SSL
cd build/step-ca/postgresql/
```

### 3. Configure Environment

```bash
cp .env.example .env
# Edit .env: set the secrets and the local-auth login
openssl rand -base64 48   # for VIBEKANBAN_REMOTE_JWT_SECRET / ELECTRIC_ROLE_PASSWORD / POSTGRES_PASSWORD
```

### 4. Deploy

```bash
docker compose up -d
```

Access: `http://localhost:8081` (forwarding mode) or `https://<VIRTUAL_HOST>`
(proxy modes). First boot: the server runs migrations and creates the
`electric_sync` database role, then Electric starts. Log in on the login page
with `SELF_HOST_LOCAL_AUTH_EMAIL` / `SELF_HOST_LOCAL_AUTH_PASSWORD`.

## 📁 Project Structure

- [`components/`](components/) — Source compose components
  - [`base/`](components/base/) — Base three-service configuration
  - [`environments/`](components/environments/) — Environment-specific configurations
- [`images/`](images/) — Source-built image definitions (NOT part of `sb build`)
- [`build/`](build/) — Generated configurations (auto-generated, ready to deploy)
- [`stackbuilder.toml`](stackbuilder.toml) — Build configuration for stackbuilder

## 🔧 Available Configurations

### Environments

- **forwarding**: publishes port 8081 directly to the host
- **letsencrypt**: joins `letsencrypt-network`; nginx-proxy terminates TLS and
  routes to `vibe-kanban:8081`
- **step-ca**: joins `step-ca-network`; same wiring with the step-ca stack

### Database variants

- **postgresql**: `postgres:alpine` via the `database/postgresql` extension
- **pgrust**: same, built for the pgrust toolchain

## 🔧 Environment Variables

### Base Configuration

- `COMPOSE_PROJECT_NAME`: Project name for Docker Compose (default: vibe-kanban)
- `IMAGE`: Prebuilt remote-server image (default: localhost/vibe-kanban-remote-server:0.1.45)
- `LOG_MAX_SIZE` / `LOG_MAX_FILE` / `LOG_COMPRESS`: json-file log rotation
- `RUST_LOG` / `SERVER_LISTEN_ADDR` / `ELECTRIC_URL`: server runtime knobs
  (defaults: `info,remote=info`, `0.0.0.0:8081`, `http://vibe-kanban-electric:3000`)
- `TZ`: Timezone (default: UTC)

### Secrets (all required)

- `VIBEKANBAN_REMOTE_JWT_SECRET`: JWT signing secret (`openssl rand -base64 48`)
- `POSTGRES_PASSWORD`: PostgreSQL password (database extension)
- `ELECTRIC_ROLE_PASSWORD`: Password for the `electric_sync` role the server
  creates on first startup

### Authentication (local auth — required)

The server refuses to start without at least one auth provider. This stack
wires local auth (there is no OAuth extension; upstream has no generic OIDC):

- `SELF_HOST_LOCAL_AUTH_EMAIL` / `SELF_HOST_LOCAL_AUTH_PASSWORD`: the login
  credential — a single shared account. Set both in `.env`; log in on the web
  login page with them. There is no CLI bootstrap and no first-run setup page:
  the env vars ARE the mechanism.

### Per-Environment Configuration

- `SERVER_PUBLIC_BASE_URL`: Public URL for auth callbacks (forwarding default:
  `http://localhost:8081`; proxy modes default to `https://<VIRTUAL_HOST>`)
- `VIBEKANBAN_PORT`: Host port (forwarding only, default: 8081)
- `VIRTUAL_PORT` / `VIRTUAL_HOST` / `VIRTUAL_PROTO` / `LETSENCRYPT_HOST` /
  `LETSENCRYPT_EMAIL`: nginx-proxy + certificate wiring (proxy modes)

## 🔌 Optional integrations (README-only — not in compose env)

These are genuinely read by the server at runtime but are optional third-party
integrations, so they are NOT wired into the compose files. To enable one,
add the entries to your `.env` AND to the `vibe-kanban` service
`environment:` block in your generated `docker-compose.yml`, then
`docker compose up -d`:

- **GitHub OAuth** — `GITHUB_OAUTH_CLIENT_ID`, `GITHUB_OAUTH_CLIENT_SECRET`
  (callback `https://<domain>/v1/oauth/github/callback`)
- **Google OAuth** — `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`
  (redirect `https://<domain>/v1/oauth/google/callback`)
- **Loops transactional email** — `LOOPS_EMAIL_API_KEY` (unset = no-op mailer,
  invitation/review emails skipped); optional `LOOPS_INVITE_TEMPLATE_ID`,
  `LOOPS_REVIEW_READY_TEMPLATE_ID`, `LOOPS_REVIEW_FAILED_TEMPLATE_ID`
  (upstream defaults apply)
- **Azure Blob attachments** — `AZURE_STORAGE_ACCOUNT_NAME` (unset = feature
  disabled), `AZURE_STORAGE_ACCOUNT_KEY`, `AZURE_STORAGE_CONTAINER_NAME`,
  `AZURE_STORAGE_ENDPOINT_URL`, optional `AZURE_STORAGE_PUBLIC_ENDPOINT_URL`
- **Relay/tunnel** — `VITE_RELAY_API_BASE_URL` is a build-arg passthrough only
  (inert at runtime); rebuild the image with it to enable relay (out of scope)

## 🌐 Networks

- **forwarding**: `vibe-kanban-network` (internal bridge)
- **letsencrypt**: `vibe-kanban-network` + `letsencrypt-network` (external)
- **step-ca**: `vibe-kanban-network` + `step-ca-network` (external)

## 🔒 Security

⚠️ **Production Checklist:**

- Generate all three secrets with `openssl rand -base64 48`; compose refuses
  to start when they are empty
- Set a strong `SELF_HOST_LOCAL_AUTH_PASSWORD` (it is the login password)
- Set `SERVER_PUBLIC_BASE_URL` to the real public URL (auth callbacks break
  otherwise)
- Relay/tunnel support is intentionally out of scope (needs wildcard TLS)

## 🆘 Troubleshooting

**Build Issues:**
- Image build OOMs → build on a host with 4+ GB free RAM (do not add swap)
- Build requires BuildKit (cache mounts); Docker 23+ has it by default

**Runtime Issues:**
- **ElectricSQL fails to connect**: the `electric_sync` role is created by
  `vibe-kanban` on first startup. Verify the app is healthy and
  `ELECTRIC_ROLE_PASSWORD` matches, then `docker compose restart vibe-kanban-electric`
- **Database connection refused**: check `docker compose logs vibe-kanban-db`;
  the server retries, but `docker compose restart vibe-kanban` helps after
  a slow first DB boot

**SSL Issues:**
- **Let's Encrypt**: verify domain DNS and letsencrypt-manager
- **Step CA**: check step-ca-manager and virtual network config

## 📝 Notes

- The `build/` directory is automatically generated and should not be edited manually
- Environment variables in generated files use `$VARIABLE_NAME` format for proper interpolation
- Each generated configuration includes a complete `docker-compose.yml` and `.env.example`
- Use `sb build` to regenerate all configurations after modifying components
- The image in `images/` is built standalone (empty build context, fetches
  upstream source itself) — never add a `build:` section to compose
- No generic OIDC: upstream auth is GitHub/Google OAuth or local auth only,
  so there is no kanidm `oidc` extension for this stack
