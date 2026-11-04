# Upstream Provenance

## app — https://github.com/BloopAI/vibe-kanban

| date | upstream commit | image tag / digest | note |
|------|-----------------|--------------------|------|
| 2026-10-31 | 7c3807084daad0d9d315064f60fa869de254f027 | localhost/vk-remote-server:0.1.45 | initial dockerization (remote variant, no relay); source-built via app/images/vk-remote-server |
| 2026-11-01 | 7c3807084daad0d9d315064f60fa869de254f027 | localhost/vibe-kanban-remote-server:0.1.45 | conformance rework: renamed image, db extension pair, env layering |
| 2026-11-02 | 7c3807084daad0d9d315064f60fa869de254f027 | - | gap analysis only (DEPLOYMENT.md); relay + host groups planned, no images built yet |

## app/host — https://github.com/BloopAI/vibe-kanban

| date | upstream commit | image tag / digest | note |
|------|-----------------|--------------------|------|
| 2026-11-03 | 7c3807084daad0d9d315064f60fa869de254f027 | localhost/vibe-kanban-host:0.1.45 | no repos-dir env; cwd /repos autodiscovered; /api/filesystem/git-repos, POST /api/repos |
| 2026-11-04 | 7c3807084daad0d9d315064f60fa869de254f027 | localhost/vibe-kanban-host-opencode:0.1.45 (FROM base) | opencode executor: npx opencode-ai@1.4.7, inherits env, merges OPENCODE_CONFIG_CONTENT; opencode data in /data/opencode |
