# 🖥️ Vibe Kanban Host

A modular Docker Compose configuration system for the local vibe-kanban host server (group 2 of the stack) with a bundled executor layer, step-ca TLS and two interchangeable ways to provide git repositories.

## 🚀 Quick Start

### 1. Build the image (once)

The image is source-built (pinned upstream commit, no vendoring) and is not part of `sb build`:

```bash
free -h   # frontend + rust release build need several GB RAM
docker build -t localhost/vibe-kanban-host:0.1.45 images/vibe-kanban-host
```

### 2. Build Configurations

```bash
sb build
```

This creates all combinations in `build/` based on [`stackbuilder.toml`](stackbuilder.toml).

### 3. Choose Your Configuration

```bash
# Development with port forwarding
cd build/forwarding/

# Production with Let's Encrypt SSL
cd build/letsencrypt/

# Production with Step CA SSL, repos cloned by a sidecar (DEFAULT)
cd build/step-ca/step-ca-repo-sync/

# Production with Step CA SSL, repos bind-mounted from the host
cd build/step-ca/step-ca-repo-local/
```

### 4. Configure Environment

```bash
cp .env.example .env
# Edit .env: VK_SHARED_API_BASE, VK_SHARED_RELAY_API_BASE, domains, repo vars
```

### 5. Deploy

```bash
docker compose up -d
```

Access: `http://localhost:3000` (forwarding mode); `https://host.vibe-kanban.af` (step-ca, via nginx-proxy).

## 🧩 Product Parts

Per [`DEPLOYMENT.md`](../DEPLOYMENT.md) (stack-conventions §20) this component is group 2 (ring-1) together with `app/relay/`:

- **vibe-kanban-host** (`host-server`): the local vibe-kanban server with embedded `local-web` UI. It runs agent sessions in git worktrees, logs in against the remote (`VK_SHARED_API_BASE`) and keeps a tunnel open to the relay (`VK_SHARED_RELAY_API_BASE`). Without it the remote still serves boards, but there are no workspaces or agents.
- **Executors**: coding-agent CLIs the host spawns. They are not in the upstream image, so our image adds Node 22 and `opencode-ai` (no upstream patch).
- **Relay pairing**: the host generates the one-time pairing code (host UI → Settings → Remote Access); the remote web consumes it. Without pairing the host stays offline for browsers. Relay and remote live in [`../relay`](../relay) and [`../remote`](../remote).

## 📁 Project Structure

- [`components/`](components/) — Source compose components
  - [`base/`](components/base/) — Host service, data and worktree volumes
  - [`environments/`](components/environments/) — `forwarding`, `letsencrypt`, `step-ca`
  - [`extensions/`](components/extensions/) — `step-ca-trust`, `repo-sync`, `repo-local`
- [`images/vibe-kanban-host/`](images/vibe-kanban-host/) — Self-contained Dockerfile (clones upstream at a pinned commit, builds frontend + Rust `server`, adds the executor layer)
- [`build/`](build/) — Generated configurations (auto-generated, ready to deploy)
- [`stackbuilder.toml`](stackbuilder.toml) — Build configuration for stackbuilder

## 🔧 Available Configurations

### Image

- `localhost/vibe-kanban-host:0.1.45`, upstream `BloopAI/vibe-kanban` @ `7c3807084daad0d9d315064f60fa869de254f027` (see `## app/host` in [`UPSTREAM.md`](../../UPSTREAM.md)).
- Multi-stage: Node 24 frontend build, Rust 1.93 server build, Debian bookworm runtime with git, ssh, tini, Node 22 and `opencode-ai`.
- Runs as uid 10001 (`appuser`); working directory `/repos`; state in `XDG_DATA_HOME=/data`; worktrees in `/var/tmp/vibe-kanban`.

### Environments

- **forwarding**: publishes the UI (`3000`) and preview proxy (`3001`) ports to the host.
- **letsencrypt**: joins the external `letsencrypt-network`, nginx-proxy / ACME variables.
- **step-ca**: joins the external `step-ca-network`, same variables. Built as two combos (below).

### Extensions

- **step-ca-trust** (`implemented`): entrypoint wrapper started as root; waits for the step-ca root CA (read-only `step-ca-data` volume, `certs` subpath), builds a CA bundle, sets `SSL_CERT_FILE`, then drops to uid 10001 via `setpriv`. Needed because the host calls the remote and relay by public https domain.
- **repo-sync** (`implemented`, DEFAULT): sidecar `vibe-kanban-repo-sync` clones a repo into the named volume `vibe-kanban-host-repos` (mounted at `/repos`) and fast-forward pulls it every `REPO_SYNC_INTERVAL` seconds. It chowns the clone to uid 10001.
- **repo-local** (`implemented`, alternative): bind-mounts `${REPOS_PATH:?}` at `/repos` to use existing working copies in place.

`repo-sync` and `repo-local` are mutually exclusive: both mount `/repos`, never list both. The upstream has no repos-dir variable; the host discovers repositories under its working directory (`/api/filesystem/git-repos?path=/repos`, `POST /api/repos`).

**Bind-mount exception:** `repo-local` deliberately breaks the no-bind-mount policy (stack-conventions §4), because working copies must be used in place. The stack never changes host permissions: `REPOS_PATH` must be writable by uid 10001 (chown/ACL on your side, see P-020).

Combos for the `step-ca` environment: `step-ca-repo-sync` = `step-ca-trust` + `repo-sync`; `step-ca-repo-local` = `step-ca-trust` + `repo-local`. `forwarding` and `letsencrypt` have no extensions yet; they get one repo extension only after the step-ca gates pass. Other catalog items (extra executor CLIs, multiple hosts) are `available`, not implemented.

## 🔧 Environment Variables

### Base Configuration

- `COMPOSE_PROJECT_NAME`: Project name (default: vibe-kanban-host)
- `IMAGE`: Docker image (default: localhost/vibe-kanban-host:0.1.45)
- `LOG_MAX_SIZE` / `LOG_MAX_FILE` / `LOG_COMPRESS`: json-file log rotation (default: 10m / 3 / true)
- `RUST_LOG`: Log level (default: info)
- `HOST`: Bind address (default: 0.0.0.0)
- `PORT`: UI/API port (default: 3000)
- `PREVIEW_PROXY_PORT`: Preview proxy port (default: 3001)
- `VK_SHARED_API_BASE`: Public URL of the remote (required)
- `VK_SHARED_RELAY_API_BASE`: Public URL of the relay (required)
- `TZ`: Timezone (default: UTC)

### Forwarding Configuration

- `HOST_UI_PORT`: Published UI port (default: 3000)
- `PREVIEW_PUBLISH_PORT`: Published preview port (default: 3001)

### Let's Encrypt / Step CA Configuration

- `VIRTUAL_PORT`: Port for nginx-proxy (default: 3000)
- `VIRTUAL_HOST`: Domain for nginx-proxy (e.g. `host.vibe-kanban.af`)
- `VIRTUAL_PROTO`: Backend protocol (default: http)
- `LETSENCRYPT_HOST`: Domain for the certificate
- `LETSENCRYPT_EMAIL`: Email for certificate registration

### Step CA Trust Configuration

- `STEP_CA_ROOT_CA_SRC`: Root CA path inside the container (default: /step-ca-root/root_ca.crt)
- `STEP_CA_ROOT_CA_FILE`: Resulting CA bundle path (default: /etc/ssl/step-ca/root_ca.crt)
- `STEP_CA_WAIT_TIMEOUT`: Seconds to wait for the root CA (default: 60)

### repo-sync Configuration

- `REPO_SYNC_URL`: Repository URL to clone (required)
- `REPO_SYNC_BRANCH`: Branch to track (default: main)
- `REPO_SYNC_DIR_NAME`: Directory under `/repos` (default: basename of the URL)
- `REPO_SYNC_TOKEN` / `REPO_SYNC_USER`: https credentials (default: empty / x-access-token)
- `REPO_SYNC_SSH_KEY`: Private key for ssh URLs (default: empty)
- `REPO_SYNC_INTERVAL`: Seconds between pulls (default: 300)
- `REPO_SYNC_IMAGE`: Sidecar image (default: alpine/git:latest)

### repo-local Configuration

- `REPOS_PATH`: Absolute host path mounted at `/repos` (required)

## 🌐 Networks

- **forwarding**: `vibe-kanban-host-network` (internal bridge)
- **letsencrypt**: `letsencrypt-network` (external)
- **step-ca**: `step-ca-network` (external)

Volumes: `vibe-kanban-host-data`, `vibe-kanban-host-worktrees`, and `vibe-kanban-host-repos` (repo-sync only). `step-ca-data` is external (step-ca-trust).

Domains in the step-ca setup: `vibe-kanban.af` (remote, `:8081`), `relay.vibe-kanban.af` (relay, `:8082`), `host.vibe-kanban.af` (this host, `:3000`).

## 🔒 Security

⚠️ **Production Checklist:**

- Keep `.env` out of git; it is gitignored in `build/`.
- Prefer `REPO_SYNC_TOKEN` (read-only token) over embedding credentials in the URL.
- `repo-local` exposes a host directory to the container: mount only what agents may modify.
- The step-ca-trust wrapper runs as root only to install the CA, then drops to uid 10001.

## ✅ Testing Status

Step-ca variant, per [`DEPLOYMENT.md`](../DEPLOYMENT.md):

- Stage C (host starts, logs in, relay connect): passed.
- Stage G (`repo-sync`): **passed** — repo listed, `POST /api/repos` ok, uid 10001 can write, fast-forward sync verified.
- Stage H (`repo-local`): **passed** — repo listed and registered, uid 10001 can write.
- Browser → relay → host session is unverified: it needs a wildcard cert for `*.relay.<domain>` (remote/relay work is paused).

## 🆘 Troubleshooting

**Build Issues:**
- The image build needs several GB RAM; check `free -h` and retry later or lower parallelism instead of changing the host.
- `sb build` removes tracked `build/**/.gitkeep` (P-013); restore with `git ls-files -d | xargs touch`.

**Repo Issues:**
- `repo-local`: host cannot write into `/repos` → fix ownership/ACL for uid 10001 on the host (P-020).
- `repo-sync`: check `docker logs vibe-kanban-repo-sync` ("clone failed" / "pull failed, will retry").
- Switching variants: both share project `vibe-kanban-host`; `docker compose down` in one variant removes `vibe-kanban-host-network` for the others (P-021). Run `down` before switching, then `up -d` in the new variant.

**SSL Issues:**
- **Let's Encrypt**: Verify domain DNS and letsencrypt-manager.
- **Step CA**: Check step-ca-manager; if the host fails to reach the remote or relay, confirm `step-ca-trust` is in the combo and the root CA exists in `step-ca-data`.

## 📝 Notes

- The `build/` directory is automatically generated and should not be edited manually.
- Environment variables in generated files use `$VARIABLE_NAME` format for proper interpolation.
- Each generated configuration includes a complete `docker-compose.yml` and `.env.example`.
- Use `sb build` to regenerate all configurations after modifying components.
- Verify repos: `GET /api/filesystem/git-repos?path=/repos`, then `POST /api/repos {"path":"/repos/<name>"}`.
