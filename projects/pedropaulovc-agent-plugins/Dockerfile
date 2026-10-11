# Node 24 matches CI and supports the memory usage scanner's native TypeScript.
# Keep the Rust toolchain and dependency caches available for offline audit builds.
FROM rust:bookworm AS rust-toolchain
FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_HOME=/usr/local/cargo \
    RUSTUP_HOME=/usr/local/rustup \
    PATH=/usr/local/cargo/bin:$PATH \
    RUSTFLAGS="-C force-frame-pointers=yes" \
    CARGO_PROFILE_RELEASE_DEBUG=2 \
    CARGO_PROFILE_RELEASE_STRIP=false
COPY --from=rust-toolchain /usr/local/cargo /usr/local/cargo
COPY --from=rust-toolchain /usr/local/rustup /usr/local/rustup
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash build-essential ca-certificates git pkg-config python3 \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src
# Root adapter dependencies are locked. The separate OpenCode runtime manifest
# has no committed npm lock; resolve/install it online and retain its image lock.
# Neither manifest needs lifecycle scripts (CI also disables root scripts).
RUN npm ci --ignore-scripts \
    && npm install --prefix .opencode --ignore-scripts --no-audit --no-fund

# Fetch all locked workspace dependencies, including dev dependencies, online.
# Compile all owned hooks and test targets; do not run tests during preparation.
RUN cargo fetch --locked \
    && cargo build --offline --locked --workspace --all-targets \
    && cargo build --offline --locked --release --workspace \
    && for hook in command-chain-separator mediocrity-detector memory-to-repo playwright-cli-headed unrelated-issue-detector windows-bash-guard; do \
         install -m 0755 "target/release/$hook" "plugins/$hook/hooks/bin/$hook"; \
       done
# Node adapters, watch-pr-monitor.mjs and record-memory-usage.ts execute directly;
# this repository has no npm build script or watch-pr backend to compile.
# Do not use cross-build wrappers: they update the Git index and need Windows tools.
