# vibe-kanban host

The local vibe-kanban server (`vibe-kanban-host`, image `localhost/vibe-kanban-host:0.1.45`, source-built).
It discovers git repositories under `/repos` (its working directory) and exposes them via
`/api/filesystem/git-repos` and `POST /api/repos`. The upstream has no repos-dir env variable.

## Repo extensions (choose exactly one; mutually exclusive — both mount `/repos`)

| Extension | Role | `/repos` source |
|-----------|------|-----------------|
| `repo-sync` | DEFAULT | named volume `vibe-kanban-host-repos`, filled by sidecar `vibe-kanban-repo-sync` (clone + periodic fast-forward pull) |
| `repo-local` | alternative | bind mount of host path `REPOS_PATH` |

Both are tested (stack-test stages G and H, step-ca variant).

### repo-sync env

| Variable | Default | Description |
|----------|---------|-------------|
| `REPO_SYNC_URL` | required | repo URL to clone |
| `REPO_SYNC_BRANCH` | `main` | branch to track |
| `REPO_SYNC_DIR_NAME` | basename of URL | directory name under `/repos` |
| `REPO_SYNC_TOKEN` / `REPO_SYNC_USER` | empty / `x-access-token` | https credentials |
| `REPO_SYNC_SSH_KEY` | empty | private key for ssh URLs |
| `REPO_SYNC_INTERVAL` | `300` | seconds between pulls |
| `REPO_SYNC_IMAGE` | `alpine/git:latest` | sidecar image |

The sidecar chowns the clone to uid 10001, so the host can write to it.

### repo-local env

| Variable | Default | Description |
|----------|---------|-------------|
| `REPOS_PATH` | required | host directory mounted at `/repos` |

**Bind-mount exception:** this is a deliberate exception to the no-bind-mount policy — existing working
copies must be used in place. The stack never changes host permissions: `REPOS_PATH` must be writable by
uid 10001 (chown/ACL by you).

## Notes

- Switching variants: both share compose project `vibe-kanban-host`; run `down` in one variant before `up` in the other.
- Verify: `GET /api/filesystem/git-repos?path=/repos` lists the repo; `POST /api/repos {"path":"/repos/<name>"}` registers it.
