# Scanner image for https://github.com/freenet/freenet-core (main): the freenet node (crates/core), its fdev developer
# tool, and the test contracts and freenet-ping app the integration tests deploy.
#
# The audit runs offline, so this build installs the toolchain rust-toolchain.toml pins (with the wasm32 target that
# contracts compile to), fetches every crate in Cargo.lock, then builds the workspace, the ping contract and app, and
# every test binary the way CI's Linux test job does. threat_model.md, next to this file, says how to run all of it.

FROM rust:1-trixie
ENV DEBIAN_FRONTEND=noninteractive
# clang and cmake cover native build scripts. CI links with mold; this image keeps Rust's default linker.
RUN apt-get update && apt-get install -y --no-install-recommends \
      clang cmake pkg-config \
    && rm -rf /var/lib/apt/lists/*

# cargo-nextest, which CI runs the suite with. .config/nextest.toml holds the per-test timeouts.
RUN case "$(uname -m)" in x86_64) platform=linux ;; aarch64) platform=linux-arm ;; *) exit 1 ;; esac \
    && curl -fsSL "https://get.nexte.st/latest/${platform}" | tar -xzf - -C "${CARGO_HOME}/bin" \
    && cargo nextest --version

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# The environment CI's Linux test job sets. RUST_MIN_STACK matches CI because some simulation tests recurse deeply.
ENV RUST_BACKTRACE=1 \
    CARGO_TERM_COLOR=never \
    RUST_LOG=error \
    RUST_MIN_STACK=268435456 \
    FREENET_BACKOFF_BASE_SECS=5 \
    CARGO_TARGET_DIR=/src/target \
    PATH=/src/target/debug:$PATH

# The toolchain and targets rust-toolchain.toml names, then every crate in Cargo.lock for every target, so the test
# harness can still compile contracts to wasm32 once the network is gone.
RUN rustup toolchain install && rustc --version && cargo fetch --locked

# The workspace (freenet and fdev land in target/debug, which is on PATH), then the ping contract and app, as CI's
# Build step does. A step that fails fails the build, and primary_contact gets the log.
RUN cargo build --locked \
    && make -C apps/freenet-ping -f run-ping.mk build

# Every test binary, with the feature set CI tests with. freenet-ping-app's tests are left out: they pull in
# freenet-test-network, which still links a second freenet-stdlib, and current linkers reject the duplicate symbols
# (https://github.com/freenet/freenet-core/issues/5950). The ping app itself is built above.
RUN cargo nextest run --locked --workspace --exclude freenet-ping-types --exclude freenet-ping-app \
      --no-default-features --features trace,websocket,redb,wasmtime-backend,testing --no-run

# Run the suite so the log shows the image works and the wasm test contracts get compiled, without letting a failure
# abort the build.
RUN timeout 2400 cargo nextest run --locked --workspace --exclude freenet-ping-types --exclude freenet-ping-app \
      --no-default-features --features trace,websocket,redb,wasmtime-backend,testing \
      --profile ci -E 'not test(blocked_peers)' \
    || echo "WARNING: cargo nextest reported failures"

# Everything is fetched. From here on cargo must not try to reach a registry.
ENV CARGO_NET_OFFLINE=true
