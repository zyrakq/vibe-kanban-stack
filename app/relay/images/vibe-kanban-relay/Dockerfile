# syntax=docker/dockerfile:1.6
# Source-built vibe-kanban relay-server (crates/relay-tunnel), pinned commit.
# Build: docker build -t localhost/vibe-kanban-relay:0.1.45 app/relay/images/vibe-kanban-relay
# Memory: rust release build needs several GB RAM; run `free -h` first.
ARG GITHUB_REPO=BloopAI/vibe-kanban
ARG GIT_COMMIT=7c3807084daad0d9d315064f60fa869de254f027

FROM alpine:3.20 AS source
ARG GITHUB_REPO
ARG GIT_COMMIT
RUN apk add --no-cache git \
 && git clone --filter=blob:none "https://github.com/${GITHUB_REPO}.git" /repo \
 && git -C /repo checkout --detach "${GIT_COMMIT}" \
 && rm -rf /repo/.git

FROM rust:1.93-slim-bookworm AS builder
ENV CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse
ARG CARGO_BUILD_JOBS=2
ENV CARGO_BUILD_JOBS=${CARGO_BUILD_JOBS}
ENV CARGO_TARGET_DIR=/app/target
RUN apt-get update \
 && apt-get install -y --no-install-recommends pkg-config libssl-dev ca-certificates \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=source /repo/ /app/
RUN mkdir -p /app/bin
RUN --mount=type=cache,id=cargo-registry,target=/usr/local/cargo/registry \
    --mount=type=cache,id=cargo-git,target=/usr/local/cargo/git \
    --mount=type=cache,id=relay-target,target=/app/target \
    cargo build --release --manifest-path crates/relay-tunnel/Cargo.toml \
 && cp /app/target/release/relay-server /app/bin/relay-server

FROM debian:bookworm-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates libssl3 wget \
 && rm -rf /var/lib/apt/lists/* \
 && useradd --system --create-home --uid 10001 appuser
COPY --from=builder /app/bin/relay-server /usr/local/bin/relay-server
USER appuser
ENV RELAY_LISTEN_ADDR=0.0.0.0:8082 \
    RUST_LOG=info
EXPOSE 8082
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["wget","--spider","-q","http://127.0.0.1:8082/health"]
ENTRYPOINT ["/usr/local/bin/relay-server"]
