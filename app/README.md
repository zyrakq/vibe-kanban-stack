# 🎯 Vibe Kanban Remote Server

A modular Docker Compose configuration system for self-hosting the Vibe Kanban
Cloud (remote) variant — multi-user Kanban for AI coding agents with GitHub /
Google OAuth or bootstrap local authentication, PostgreSQL persistence, and
ElectricSQL real-time sync to the browser client.

## 🚀 Quick Start

### 0. Build the source image (once)

No upstream Docker image is published, so the remote-server image is built
from source (Rust nightly + pnpm; 10–15 min, 4+ GB RAM, BuildKit required):

```bash
docker build -t localhost/vibe-kanban-remote-server:0.1.45 images/vibe-kanban-remote-server
```

The Dockerfile is fully self-contained: it clones the upstream repo at the
pinned commit `7c380708` itself and builds with an empty context. The release
tag `v0.1.45-20260919085201` does not compile (fixed three commits after the
tag), hence the pinned commit.

### 1. Build Configurations

```bash
sb build
```

This will create all combinations in the `build/` directory based on
[`stackbuilder.toml`](stackbuilder.toml).

### 2. Choose Your Configuration

```bash
# For development with port forwarding
cd build/forwarding/postgresql/

# For production with Let's Encrypt SSL
cd build/letsencrypt/postgresql/

# For production with Step CA SSL
cd build/step-ca/postgresql/
```

Swap `postgresql` for `pgrust` in any path to run the database on the
pgrust image instead (see Extensions below).

### 3. Configure Environment

```bash
cp .env.example .env
# Edit .env: set the secrets and one auth method
openssl rand -base64 48   # for VIBEKANBAN_REMOTE_JWT_SECRET
```

### 4. Deploy

```bash
docker compose up -d
```

Access: `http://localhost:8081` (forwarding mode) or `https://<VIRTUAL_HOST>`
(proxy modes). First boot: the database initializes, the server runs
migrations and creates the `electric_sync` role, then Electric starts.

## 📁 Project Structure

- [`components/`](components/) — Source compose components
  - [`base/`](components/base/) — Base app + Electric configuration
  - [`environments/`](components/environments/) — Environment-specific configurations
  - [`extensions/`](components/extensions/) — Extension components (database pair)
- [`images/`](images/) — Source-built image definitions (NOT part of `sb build`)
- [`build/`](build/) — Generated configurations (auto-generated, ready to deploy)
- [`stackbuilder.toml`](stackbuilder.toml) — Build configuration for stackbuilder

## 🔧 Available Configurations

### Environments

- **forwarding**: publishes `VIBEKANBAN_PORT` (default 8081) directly to the host
- **letsencrypt**: joins `letsencrypt-network`; nginx-proxy terminates TLS and
  routes to `vibe-kanban-app:8081`
- **step-ca**: joins `step-ca-network`; same wiring with the step-ca stack

### Extensions

- **postgresql**: the `vibe-kanban-db` service on `postgres:alpine` — the
  default database combo
- **pgrust**: the same service on `malisper/pgrust:v0.2` — alternative
  Postgres-compatible engine. Caveat: pgrust may not support logical
  replication (`wal_level=logical`, required by Electric); if the Stage A2
  gate fails, `postgresql` stays the default combo

## 🔧 Environment Variables

### Base Configuration

- `COMPOSE_PROJECT_NAME`: Project name for Docker Compose (default: vibe-kanban)
- `IMAGE`: Prebuilt remote-server image (default: localhost/vibe-kanban-remote-server:0.1.45)
- `LOG_MAX_SIZE` / `LOG_MAX_FILE` / `LOG_COMPRESS`: json-file log rotation
- `RUST_LOG`: Server log filter (default: info,remote=info)
- `SERVER_LISTEN_ADDR`: Server listen address (default: 0.0.0.0:8081)
- `ELECTRIC_URL`: Electric endpoint the server proxies (default: http://vibe-kanban-electric:3000)
- `TZ`: Timezone (default: UTC)

### Secrets (both required)

- `VIBEKANBAN_REMOTE_JWT_SECRET`: JWT signing secret (`openssl rand -base64 48`)
- `ELECTRIC_ROLE_PASSWORD`: Password for the `electric_sync` role the server
  creates on first startup

### Auth Configuration (at least one method)

- `GITHUB_OAUTH_CLIENT_ID` / `GITHUB_OAUTH_CLIENT_SECRET`: GitHub OAuth app
  (callback `https://<domain>/v1/oauth/github/callback`)
- `GOOGLE_OAUTH_CLIENT_ID` / `GOOGLE_OAUTH_CLIENT_SECRET`: Google OAuth
  (redirect `https://<domain>/v1/oauth/google/callback`)
- `SELF_HOST_LOCAL_AUTH_EMAIL` / `SELF_HOST_LOCAL_AUTH_PASSWORD`: bootstrap
  local admin credential (single shared login)

### Database Configuration (postgresql / pgrust combos)

- `POSTGRES_DB`: Database name (default: vibe-kanban)
- `POSTGRES_USER`: Database user (default: vibe-kanban)
- `POSTGRES_PASSWORD`: Database password (required)
- `DB_IMAGE`: Override the database image (default: `postgres:alpine` or
  `malisper/pgrust:v0.2` per combo)
- `COMPOSE_EXTENDS` / `COMPOSE_FILE`: internal chain that layers
  `docker-compose.db.yml` onto the merged compose — leave as generated

### Forwarding Configuration

- `VIBEKANBAN_PORT`: Host port for direct access (default: 8081)
- `SERVER_PUBLIC_BASE_URL`: Public URL for OAuth callbacks
  (default: http://localhost:8081)

### Let's Encrypt / Step CA Configuration

- `VIRTUAL_PORT`: Port for nginx-proxy (default: 8081)
- `VIRTUAL_HOST`: Domain for nginx-proxy
- `VIRTUAL_PROTO`: Protocol nginx-proxy expects (default: http)
- `LETSENCRYPT_HOST`: Domain for the SSL certificate
- `LETSENCRYPT_EMAIL`: Email for certificate registration
- `SERVER_PUBLIC_BASE_URL`: derived as `https://${VIRTUAL_HOST}` by the
  overlay — set it only to override

### Optional Configuration (not in .env.example — set in `.env` to use)

- `LOOPS_EMAIL_API_KEY`: transactional email (empty = invitation emails off).
  Upstream template IDs, if you have your own Loops account:
  `LOOPS_INVITE_TEMPLATE_ID=cmhvy2wgs3s13z70i1pxakij9`,
  `LOOPS_REVIEW_READY_TEMPLATE_ID=cmj47k5ge16990iylued9by17`,
  `LOOPS_REVIEW_FAILED_TEMPLATE_ID=cmj49ougk1c8s0iznavijdqpo`
- `AZURE_STORAGE_ACCOUNT_NAME` / `AZURE_STORAGE_ACCOUNT_KEY` /
  `AZURE_STORAGE_CONTAINER_NAME` / `AZURE_STORAGE_ENDPOINT_URL` /
  `AZURE_STORAGE_PUBLIC_ENDPOINT_URL`: issue attachment blob storage
- `ELECTRIC_IMAGE` / `PG_PROXY_PORT` / `LOGICAL_PUBLISHER_HOST` / `AUTH_MODE` /
  `ELECTRIC_INSECURE` / `ELECTRIC_MANUAL_TABLE_PUBLISHING` /
  `ELECTRIC_USAGE_REPORTING` / `ELECTRIC_FEATURE_FLAGS`: Electric tuning —
  defaults match upstream; override only if you know why
- `VITE_RELAY_API_BASE_URL`: dead build arg at runtime — the image accepts it
  as a build-arg passthrough only; relay/tunnel support needs wildcard TLS and
  is out of scope (rebuild the image with it to experiment)

## 🌐 Networks

- **forwarding**: `vibe-kanban-network` (internal bridge)
- **letsencrypt**: `vibe-kanban-network` + `letsencrypt-network` (external)
- **step-ca**: `vibe-kanban-network` + `step-ca-network` (external)

## 🔒 Security

⚠️ **Production Checklist:**

- Generate both secrets with `openssl rand -base64 48`; compose refuses to
  start when they are empty
- Configure OAuth with exact callback URLs, or set a strong
  `SELF_HOST_LOCAL_AUTH_PASSWORD`
- The public base URL follows the environment (http for forwarding, https for
  proxy modes) — override `SERVER_PUBLIC_BASE_URL` only when it differs
- Relay/tunnel support is intentionally out of scope (needs wildcard TLS)

## 🆘 Troubleshooting

**Build Issues:**
- Image build OOMs → build on a host with 4+ GB free RAM; never modify the
  host to add swap
- Build requires BuildKit (cache mounts); Docker 23+ has it by default
- Build fails at the pinned tag → do not "upgrade" to the release tag; it does
  not compile (see Quick Start step 0)

**Runtime Issues:**
- **ElectricSQL fails to connect**: the `electric_sync` role is created by
  `vibe-kanban-app` on first startup. Verify the app is healthy and
  `ELECTRIC_ROLE_PASSWORD` matches, then
  `docker compose restart vibe-kanban-electric`
- **Database connection refused**: check `docker compose logs vibe-kanban-db`;
  the app retries, but `docker compose restart vibe-kanban-app` helps after a
  slow first database boot

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
- No generic OIDC: upstream auth is GitHub/Google OAuth or bootstrap local
  auth only, so there is no kanidm `oidc` extension for this stack
- Staged test status: `docker compose config` validation passed for all six
  variants; the image build and Stage A browser gate (base + postgresql) are
  the next run — treat the stack as untested until then
