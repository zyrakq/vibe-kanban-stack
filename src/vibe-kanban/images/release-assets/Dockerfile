# Build stage
FROM node:24-alpine AS builder

# Install build dependencies including wget for downloading
RUN apk add --no-cache \
    curl \
    build-base \
    perl \
    llvm-dev \
    clang-dev \
    wget \
    tar \
    gzip

# Allow linking libclang on musl
ENV RUSTFLAGS="-C target-feature=-crt-static"

# Install Rust
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Release download arguments
ARG VERSION=0.0.161
ARG GITHUB_REPO=BloopAI/vibe-kanban
ARG POSTHOG_API_KEY
ARG POSTHOG_API_ENDPOINT

ENV VITE_PUBLIC_POSTHOG_KEY=$POSTHOG_API_KEY
ENV VITE_PUBLIC_POSTHOG_HOST=$POSTHOG_API_ENDPOINT

# Set working directory
WORKDIR /app

# Download and extract release archive
RUN echo "Downloading release v${VERSION}..." && \
    DOWNLOAD_URL="https://github.com/${GITHUB_REPO}/archive/refs/tags/v${VERSION}.tar.gz" && \
    echo "URL: ${DOWNLOAD_URL}" && \
    wget -O release.tar.gz "${DOWNLOAD_URL}" && \
    tar -xzf release.tar.gz --strip-components=1 && \
    rm release.tar.gz

# Install pnpm and dependencies
RUN npm install -g pnpm && pnpm install

# Build application
RUN npm run generate-types
RUN cd frontend && pnpm run build
RUN cargo build --release --bin server

# Runtime stage
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

# Copy binary from builder
COPY --from=builder /app/target/release/server /usr/local/bin/server

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