# syntax=docker/dockerfile:1.6
#
# Source-built image for the BloopAI/vibe-kanban remote (Cloud) variant.
# Fully self-contained: clones the upstream repo at a pinned commit and builds
# with an EMPTY build context (stack-conventions §19 — no vendored source).
#
# Built from commit 7c3807084daad0d9d315064f60fa869de254f027 (3 commits after
# the v0.1.45-20260919085201 tag): the tag itself does not compile — its
# remote-web destructures getMatchedRoutes as an object while the lockfile's
# router-core returns a tuple; fixed immediately after the tag.
#
# Self-hosted build (FEATURES empty) strips the private billing crate from
# crates/remote/Cargo.toml, so no SSH / private-repo access is needed and no
# --mount=type=ssh is used.
#
# Build standalone:
#   docker build -t localhost/vibe-kanban-remote-server:<tag> images/vibe-kanban-remote-server
#
# Requires BuildKit (cache mounts). Heavy: Rust nightly release build + pnpm
# frontend build, ~10-15 min, 4+ GB RAM.

ARG GITHUB_REPO=BloopAI/vibe-kanban
ARG GIT_COMMIT=7c3807084daad0d9d315064f60fa869de254f027
ARG APP_NAME=remote
ARG FEATURES=""
ARG VITE_RELAY_API_BASE_URL=""
ARG VITE_PUBLIC_REACT_VIRTUOSO_LICENSE_KEY=""
ARG POSTHOG_API_KEY=""
ARG POSTHOG_API_ENDPOINT=""
ARG SENTRY_DSN_REMOTE=""

# --- source stage: fetch upstream source at a pinned commit (no vendoring) ---
FROM alpine:3.20 AS source
ARG GITHUB_REPO
ARG GIT_COMMIT
RUN apk add --no-cache git \
 && git clone --filter=blob:none "https://github.com/${GITHUB_REPO}.git" /repo \
 && git -C /repo checkout --detach "${GIT_COMMIT}" \
 && rm -rf /repo/.git

# --- frontend builder (remote-web SPA, embedded at /srv/static) ---
FROM node:20-alpine AS fe-builder
ARG VITE_RELAY_API_BASE_URL
ARG VITE_PUBLIC_REACT_VIRTUOSO_LICENSE_KEY
WORKDIR /repo
ENV VITE_RELAY_API_BASE_URL=${VITE_RELAY_API_BASE_URL} \
    VITE_PUBLIC_REACT_VIRTUOSO_LICENSE_KEY=${VITE_PUBLIC_REACT_VIRTUOSO_LICENSE_KEY}

RUN corepack enable

# Copy manifests first so `pnpm install` is cached independently of source
COPY --from=source /repo/pnpm-lock.yaml /repo/pnpm-workspace.yaml /repo/package.json ./
COPY --from=source /repo/patches/ patches/
COPY --from=source /repo/packages/local-web/package.json packages/local-web/package.json
COPY --from=source /repo/packages/remote-web/package.json packages/remote-web/package.json
COPY --from=source /repo/packages/ui/package.json packages/ui/package.json
COPY --from=source /repo/packages/web-core/package.json packages/web-core/package.json

RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile

COPY --from=source /repo/packages/remote-web/ packages/remote-web/
COPY --from=source /repo/packages/local-web/tailwind.new.config.js packages/local-web/tailwind.new.config.js
COPY --from=source /repo/packages/public/ packages/public/
COPY --from=source /repo/packages/ui/ packages/ui/
COPY --from=source /repo/packages/web-core/ packages/web-core/
COPY --from=source /repo/shared/ shared/

RUN pnpm -C packages/remote-web build

# --- rust builder (remote server binary) ---
FROM rust:1.93-slim-bookworm AS builder
ARG APP_NAME
ARG FEATURES
ARG POSTHOG_API_KEY
ARG POSTHOG_API_ENDPOINT
ARG SENTRY_DSN_REMOTE

ENV CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse \
    CARGO_NET_GIT_FETCH_WITH_CLI=true \
    CARGO_TARGET_DIR=/app/target \
    POSTHOG_API_KEY=${POSTHOG_API_KEY} \
    POSTHOG_API_ENDPOINT=${POSTHOG_API_ENDPOINT} \
    SENTRY_DSN_REMOTE=${SENTRY_DSN_REMOTE}

RUN apt-get update \
 && apt-get install -y --no-install-recommends pkg-config libssl-dev ca-certificates git \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=source /repo/ .

# rust-toolchain.toml pins nightly-2025-12-04; rustup in the rust image
# auto-installs it on first cargo invocation.
RUN cargo --version >/dev/null

RUN mkdir -p /app/bin

# Self-hosted build (FEATURES empty): strip the private billing dependency and
# its feature flag so cargo never fetches the private git repo; drop the
# crate-local Cargo.lock that references it.
RUN if [ -z "${FEATURES}" ]; then \
      sed -i '/^billing = {.*vibe-kanban-private.*/d' crates/remote/Cargo.toml; \
      sed -i '/^# private crate for billing/d' crates/remote/Cargo.toml; \
      sed -i 's/^vk-billing = \["dep:billing"\]/vk-billing = []/' crates/remote/Cargo.toml; \
      rm -f crates/remote/Cargo.lock; \
    fi

RUN --mount=type=cache,id=cargo-registry,target=/usr/local/cargo/registry \
    --mount=type=cache,id=cargo-git,target=/usr/local/cargo/git \
    --mount=type=cache,id=remote-target,target=/app/target \
    cargo build ${FEATURES:+--locked} --release --manifest-path crates/remote/Cargo.toml ${FEATURES:+--features ${FEATURES}} \
 && cp /app/target/release/${APP_NAME} /app/bin/${APP_NAME}

# --- runtime ---
FROM debian:bookworm-slim AS runtime
ARG APP_NAME

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates libssl3 wget git \
 && rm -rf /var/lib/apt/lists/* \
 && useradd --system --create-home --uid 10001 appuser

WORKDIR /srv

COPY --from=builder /app/bin/${APP_NAME} /usr/local/bin/${APP_NAME}
COPY --from=fe-builder /repo/packages/remote-web/dist /srv/static

USER appuser

ENV SERVER_LISTEN_ADDR=0.0.0.0:8081 \
    RUST_LOG=info

EXPOSE 8081

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["wget","--spider","-q","http://127.0.0.1:8081/v1/health"]

ENTRYPOINT ["/usr/local/bin/remote"]
