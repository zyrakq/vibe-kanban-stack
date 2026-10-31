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
docker build -t localhost/vk-remote-server:0.1.45 images/vk-remote-server
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
cd build/forwarding/

# For production with Let's Encrypt SSL
cd build/letsencrypt/

# For production with Step CA SSL
cd build/step-ca/
```

### 3. Configure Environment

```bash
cp .env.example .env
# Edit .env: set the three secrets and one auth method
openssl rand -base64 48   # for VIBEKANBAN_REMOTE_JWT_SECRET
```

### 4. Deploy

```bash
docker compose up -d
```

Access: `http://localhost:8081` (forwarding mode) or `https://<VIRTUAL_HOST>`
(proxy modes). First boot: the server runs migrations and creates the
`electric_sync` database role, then Electric starts.

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
  routes to `remote-server:8081`
- **step-ca**: joins `step-ca-network`; same wiring with the step-ca stack

## 🔧 Environment Variables

### Base Configuration

- `COMPOSE_PROJECT_NAME`: Project name for Docker Compose (default: vibe-kanban)
- `PROJECT_NAME`: Volume/network prefix convention (default: vk)
- `IMAGE`: Prebuilt remote-server image (default: localhost/vk-remote-server:0.1.45)
- `LOG_MAX_SIZE` / `LOG_MAX_FILE` / `LOG_COMPRESS`: json-file log rotation
- `SERVER_PUBLIC_BASE_URL`: Public URL for OAuth callbacks (default: http://localhost:8081)
- `TZ`: Timezone (default: UTC)

### Secrets (all required)

- `VIBEKANBAN_REMOTE_JWT_SECRET`: JWT signing secret (`openssl rand -base64 48`)
- `DB_PASSWORD`: PostgreSQL password for the `remote` user
- `ELECTRIC_ROLE_PASSWORD`: Password for the `electric_sync` role the server
  creates on first startup

### Auth Configuration (at least one method)

- `GITHUB_OAUTH_CLIENT_ID` / `GITHUB_OAUTH_CLIENT_SECRET`: GitHub OAuth app
  (callback `https://<domain>/v1/oauth/github/callback`)
- `GOOGLE_OAUTH_CLIENT_ID` / `GOOGLE_OAUTH_CLIENT_SECRET`: Google OAuth
  (redirect `https://<domain>/v1/oauth/google/callback`)
- `SELF_HOST_LOCAL_AUTH_EMAIL` / `SELF_HOST_LOCAL_AUTH_PASSWORD`: bootstrap
  local admin credential (single shared login)

### Optional Configuration

- `LOOPS_EMAIL_API_KEY`: transactional email (empty = invitation emails off)
- `LOOPS_INVITE_TEMPLATE_ID` / `LOOPS_REVIEW_READY_TEMPLATE_ID` /
  `LOOPS_REVIEW_FAILED_TEMPLATE_ID`: Loops template IDs (upstream defaults)
- `AZURE_STORAGE_ACCOUNT_NAME` / `AZURE_STORAGE_ACCOUNT_KEY` /
  `AZURE_STORAGE_CONTAINER_NAME` / `AZURE_STORAGE_ENDPOINT_URL` /
  `AZURE_STORAGE_PUBLIC_ENDPOINT_URL`: issue attachment blob storage
- `VITE_RELAY_API_BASE_URL`: build-arg passthrough only — inert at runtime;
  rebuild the image with it to enable relay (out of scope here)

### Let's Encrypt / Step CA Configuration

- `VIRTUAL_PORT`: Port for nginx-proxy (default: 8081)
- `VIRTUAL_HOST`: Domain for nginx-proxy
- `VIRTUAL_PROTO`: Protocol nginx-proxy expects (default: http)
- `LETSENCRYPT_HOST`: Domain for SSL certificate
- `LETSENCRYPT_EMAIL`: Email for certificate registration

## 🌐 Networks

- **forwarding**: `vk-network` (internal bridge)
- **letsencrypt**: `vk-network` + `letsencrypt-network` (external)
- **step-ca**: `vk-network` + `step-ca-network` (external)

## 🔒 Security

⚠️ **Production Checklist:**

- Generate all three secrets with `openssl rand -base64 48`; compose refuses
  to start when they are empty
- Configure OAuth with exact callback URLs, or set a strong
  `SELF_HOST_LOCAL_AUTH_PASSWORD`
- Set `SERVER_PUBLIC_BASE_URL` to the real public URL (OAuth callbacks break
  otherwise)
- Relay/tunnel support is intentionally out of scope (needs wildcard TLS)

## 🆘 Troubleshooting

**Build Issues:**
- Image build OOMs → add swap or build on a host with 4+ GB free RAM
- Build requires BuildKit (cache mounts); Docker 23+ has it by default

**Runtime Issues:**
- **ElectricSQL fails to connect**: the `electric_sync` role is created by the
  remote-server on first startup. Verify the server is healthy and
  `ELECTRIC_ROLE_PASSWORD` matches, then `docker compose restart electric`
- **Database connection refused**: check `docker compose logs remote-db`;
  the server retries, but `docker compose restart remote-server` helps after
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
- No generic OIDC: upstream auth is GitHub/Google OAuth or bootstrap local
  auth only, so there is no kanidm `oidc` extension for this stack
