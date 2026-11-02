# syntax=docker/dockerfile:1.6
# Source-built vibe-kanban host (local server + web UI), pinned commit, plus
# executor layer (opencode-ai via npm, Node 22).
# Build: docker build -t localhost/vibe-kanban-host:0.1.45 app/host/images/vibe-kanban-host
# Memory: frontend + rust release build need several GB RAM; run `free -h` first.
# Findings (upstream @7c38070): state dir = ProjectDirs("ai","bloop","vibe-kanban")
#   data_dir -> $XDG_DATA_HOME/vibe-kanban, so XDG_DATA_HOME=/data and volume on /data;
#   worktree base = /var/tmp/vibe-kanban on Linux (volume);
#   process runs as non-root appuser (uid 10001): dirs pre-created+chowned so named volumes inherit ownership.
ARG GITHUB_REPO=BloopAI/vibe-kanban
ARG GIT_COMMIT=7c3807084daad0d9d315064f60fa869de254f027

FROM alpine:3.20 AS source
ARG GITHUB_REPO
ARG GIT_COMMIT
RUN apk add --no-cache git \
 && git clone --filter=blob:none "https://github.com/${GITHUB_REPO}.git" /repo \
 && git -C /repo checkout --detach "${GIT_COMMIT}" \
 && rm -rf /repo/.git

FROM node:24-alpine AS fe-builder
ARG POSTHOG_API_KEY=""
ARG POSTHOG_API_ENDPOINT=""
WORKDIR /app
ENV PNPM_HOME=/pnpm
ENV PATH=${PNPM_HOME}:${PATH}
ENV VITE_PUBLIC_POSTHOG_KEY=${POSTHOG_API_KEY}
ENV VITE_PUBLIC_POSTHOG_HOST=${POSTHOG_API_ENDPOINT}
ENV NODE_OPTIONS=--max-old-space-size=4096
RUN corepack enable
RUN pnpm config set store-dir /pnpm/store
COPY --from=source /repo/pnpm-lock.yaml /repo/pnpm-workspace.yaml /repo/package.json ./
COPY --from=source /repo/patches/ patches/
COPY --from=source /repo/packages/local-web/package.json packages/local-web/package.json
COPY --from=source /repo/packages/ui/package.json packages/ui/package.json
COPY --from=source /repo/packages/web-core/package.json packages/web-core/package.json
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile
COPY --from=source /repo/packages/local-web/ packages/local-web/
COPY --from=source /repo/packages/public/ packages/public/
COPY --from=source /repo/packages/ui/ packages/ui/
COPY --from=source /repo/packages/web-core/ packages/web-core/
COPY --from=source /repo/shared/ shared/
RUN pnpm -C packages/local-web build

FROM rust:1.93-slim-bookworm AS builder
ARG POSTHOG_API_KEY=""
ARG POSTHOG_API_ENDPOINT=""
ARG SENTRY_DSN=""
ARG VK_SHARED_API_BASE=""
ENV CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse
ENV CARGO_NET_GIT_FETCH_WITH_CLI=true
ENV CARGO_TARGET_DIR=/app/target
ENV POSTHOG_API_KEY=${POSTHOG_API_KEY}
ENV POSTHOG_API_ENDPOINT=${POSTHOG_API_ENDPOINT}
ENV SENTRY_DSN=${SENTRY_DSN}
ENV VK_SHARED_API_BASE=${VK_SHARED_API_BASE}
WORKDIR /app
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    build-essential ca-certificates git libclang-dev libssl-dev pkg-config \
 && rm -rf /var/lib/apt/lists/*
COPY --from=source /repo/ /app/
COPY --from=fe-builder /app/packages/local-web/dist packages/local-web/dist
RUN --mount=type=cache,id=cargo-registry,target=/usr/local/cargo/registry \
    --mount=type=cache,id=cargo-git,target=/usr/local/cargo/git \
    --mount=type=cache,id=workspace-target,target=/app/target \
    cargo build --locked --release --bin server \
 && cp /app/target/release/server /usr/local/bin/server

FROM debian:bookworm-slim AS runtime
ARG NODE_MAJOR=22
ARG EXECUTOR_NPM_PACKAGES="opencode-ai@latest"
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    ca-certificates git openssh-client tini wget curl gnupg \
 && curl -fsSL https://deb.nodesource.com/setup_${NODE_MAJOR}.x | bash - \
 && apt-get install -y --no-install-recommends nodejs \
 && npm install -g ${EXECUTOR_NPM_PACKAGES} \
 && rm -rf /var/lib/apt/lists/* /root/.npm \
 && useradd --system --create-home --uid 10001 appuser
WORKDIR /repos
COPY --from=builder /usr/local/bin/server /usr/local/bin/server
RUN mkdir -p /repos /data /var/tmp/vibe-kanban \
 && chown -R appuser:appuser /repos /data /var/tmp/vibe-kanban
USER appuser
ENV HOST=0.0.0.0
ENV PORT=3000
ENV XDG_DATA_HOME=/data
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/bin/sh", "-c", "wget --spider -q http://127.0.0.1:${PORT:-3000}/health"]
ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/server"]
