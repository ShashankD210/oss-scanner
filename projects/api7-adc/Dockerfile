# syntax=docker/dockerfile:1
FROM rust:1.95-bookworm AS rust
FROM node:24-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_HOME=/usr/local/cargo \
    RUSTUP_HOME=/usr/local/rustup \
    PATH=/usr/local/cargo/bin:$PATH \
    NX_DAEMON=false \
    NX_NO_CLOUD=true \
    NODE_OPTIONS=--no-experimental-strip-types

COPY --from=rust /usr/local/cargo /usr/local/cargo
COPY --from=rust /usr/local/rustup /usr/local/rustup
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential clang cmake pkg-config git python3 \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# Use the package-manager version declared by the checkout, retaining dev tools.
RUN npm install --global "pnpm@$(node -p "require('./package.json').packageManager.split('@')[1].split('+')[0]")" \
    && pnpm install --frozen-lockfile \
    && pnpm exec nx build cli --skip-nx-cache \
    && node dist/apps/cli/main.cjs --help

# Fetch and compile the Rust port, including test-only dependencies, while online.
RUN cd rust \
    && cargo fetch --locked \
    && cargo build --workspace --locked \
    && cargo run --locked -p adc-differ --example gen_fixtures \
    && cargo test --workspace --locked --no-run

RUN <<'EOF'
cat > /usr/local/bin/oss-scanner-test <<'SCRIPT'
#!/bin/bash
set -euo pipefail
cd /src
unset E2E
pnpm exec nx run-many --target=test \
    --projects=sdk,differ,cli,backend-apisix,backend-api7,converter-openapi \
    --parallel=2 --skip-nx-cache
cd /src/rust
cargo test --workspace --locked --offline
SCRIPT
chmod +x /usr/local/bin/oss-scanner-test
EOF

# Backend E2E suites need live gateway services; unit suites run offline.
WORKDIR /src
CMD ["bash"]
