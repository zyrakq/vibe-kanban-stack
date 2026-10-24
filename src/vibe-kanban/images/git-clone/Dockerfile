# Stage 1: Rust Base with Type Generation
FROM rust:alpine AS rust-base

# Install all Rust build dependencies once
RUN apk add --no-cache \
    build-base \
    perl \
    llvm-dev \
    clang-dev \
    git

# Allow linking libclang on musl
ENV RUSTFLAGS="-C target-feature=-crt-static"

# Git repository arguments
ARG GIT_REPO=https://github.com/BloopAI/vibe-kanban.git
ARG GIT_BRANCH=main
ARG GIT_TAG=""

# Set working directory
WORKDIR /app

# Clone repository
RUN if [ -n "$GIT_TAG" ]; then \
    echo "Cloning tag ${GIT_TAG}..." && \
    git clone --depth 1 --branch ${GIT_TAG} ${GIT_REPO} . ; \
    else \
    echo "Cloning branch ${GIT_BRANCH}..." && \
    git clone --depth 1 --branch ${GIT_BRANCH} ${GIT_REPO} . ; \
    fi

# Generate TypeScript types from Rust
RUN cargo run --bin generate_types

# Stage 2: Frontend Builder
FROM node:24-alpine AS frontend-builder

# PostHog configuration
ARG POSTHOG_API_KEY
ARG POSTHOG_API_ENDPOINT

ENV VITE_PUBLIC_POSTHOG_KEY=$POSTHOG_API_KEY
ENV VITE_PUBLIC_POSTHOG_HOST=$POSTHOG_API_ENDPOINT

# Set working directory
WORKDIR /app

# Copy repository with generated types from rust-base
COPY --from=rust-base /app /app

# Install pnpm and dependencies
RUN npm install -g pnpm && pnpm install

# Build frontend
RUN cd frontend && pnpm run build

# Stage 3: Backend Builder (reuse rust-base layer)
FROM rust-base AS backend-builder

# Copy built frontend artifacts from frontend-builder
COPY --from=frontend-builder /app/frontend/dist /app/frontend/dist

# Build Rust server binary
RUN cargo build --release --bin server

# Stage 4: Runtime
FROM alpine:latest AS runtime

# Install runtime dependencies
RUN apk add --no-cache \
    ca-certificates \
    tini \
    libgcc \
    wget

# Create app user for security
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Copy binary from backend builder
COPY --from=backend-builder /app/target/release/server /usr/local/bin/server

# Create repos directory and set permissions
RUN mkdir -p /repos && \
    chown -R appuser:appgroup /repos

# Switch to non-root user
USER appuser

# Set runtime environment
ENV HOST=0.0.0.0
ENV PORT=3000
EXPOSE 3000

# Set working directory
WORKDIR /repos

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider "http://${HOST:-localhost}:${PORT:-3000}" || exit 1

# Run the application
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["server"]
