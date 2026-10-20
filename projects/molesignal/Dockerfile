# Keep source, toolchains, dependency caches, and test binaries for offline auditing.
FROM node:20-bookworm-slim AS node
FROM rust:1.96-bookworm

COPY --from=node /usr/local/ /usr/local/
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl git python3 build-essential clang cmake pkg-config \
    protobuf-compiler libssl-dev libcurl4-openssl-dev \
    && rm -rf /var/lib/apt/lists/*
# Only enable pnpm: the Node image's Yarn links point outside /usr/local.
RUN corepack enable pnpm && corepack prepare pnpm@9 --activate

ENV CARGO_BUILD_JOBS=2 \
    CARGO_PROFILE_DEV_DEBUG=1 \
    CARGO_PROFILE_TEST_DEBUG=1 \
    CARGO_INCREMENTAL=0 \
    NODE_OPTIONS=--max-old-space-size=8192
WORKDIR /src
COPY . /src

# The Rust server embeds web/dist. Install dev dependencies for offline web tests.
RUN pnpm -C web install --frozen-lockfile && pnpm -C web build
# Install the checkout's toolchain and fetch both registry and pinned git dependencies.
RUN rustup show active-toolchain && cargo fetch --locked
# Building online also fetches native build inputs such as the V8 archive.
RUN cargo build --locked --workspace
RUN cargo test --locked --workspace --lib --bins --no-run

# Preserve failing-test logs without making a test assertion abort image creation.
RUN cargo test --offline --locked --workspace --lib --bins > /tmp/molesignal-rust-tests.log 2>&1 \
    || (cat /tmp/molesignal-rust-tests.log; echo 'WARNING: Rust tests failed; see /tmp/molesignal-rust-tests.log')
RUN pnpm -C web test:run > /tmp/molesignal-web-tests.log 2>&1 \
    || (cat /tmp/molesignal-web-tests.log; echo 'WARNING: web tests failed; see /tmp/molesignal-web-tests.log')

CMD ["bash"]
