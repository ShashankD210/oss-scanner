# Scanner image for https://github.com/starkware-libs/proving (main): the stwo prover/verifier, the stwo-cairo AIR,
# prover and Rust verifier, stwo-circuits, proving-utils and the Cairo verifier in stwo_cairo_verifier/.
#
# The audit runs offline, so this build installs the nightly toolchain pinned in rust-toolchain.toml and Scarb at the
# version in .tool-versions, fetches every crate and Scarb package, then compiles the workspace and every test binary in
# the dev profile (debug assertions and overflow checks on) and builds the Cairo verifier. threat_model.md, next to this
# file, says how to run all of it.
FROM rust:1-trixie
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential clang jq python3 \
    && rm -rf /var/lib/apt/lists/*

# cargo-nextest: the slow-test jobs and .config/nextest.toml's "ci" profile use it.
ARG NEXTEST_VERSION=0.9.113
RUN curl -fsSL "https://get.nexte.st/${NEXTEST_VERSION}/linux" | tar -xzf - -C "${CARGO_HOME}/bin" \
    && cargo nextest --version

# Scarb, at the version .tool-versions pins, for stwo_cairo_verifier/.
ARG SCARB_VERSION=2.18.0
ENV PATH=/root/.local/bin:$PATH
RUN curl --proto '=https' --tlsv1.2 -sSf https://docs.swmansion.com/scarb/install.sh | sh -s -- -v "${SCARB_VERSION}" \
    && scarb --version

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

ENV RUST_BACKTRACE=1 \
    CARGO_TERM_COLOR=never \
    CARGO_PROFILE_DEV_DEBUG=line-tables-only \
    CARGO_PROFILE_TEST_DEBUG=line-tables-only

# rust-toolchain.toml pins a nightly (the prover's SIMD backend uses portable_simd); install it, then every crate in
# Cargo.lock for every target and feature, so later builds with other features or profiles stay offline.
RUN rustup toolchain install \
    && cargo --version \
    && cargo fetch --locked

# The workspace and every test binary, default features. A step that fails fails the build, and primary_contact gets
# the log.
RUN cargo build --locked --workspace --bins \
    && cargo nextest run --locked --workspace --no-run

# The Cairo verifier: fetch its Scarb dependencies and build it, with the qm31_opcode feature CI lints with.
RUN cd stwo_cairo_verifier \
    && scarb fetch \
    && scarb build --features=qm31_opcode

# Run the core stwo tests and the Cairo verifier's tests so the log shows the image works, without letting a failure
# abort the build.
RUN timeout 1800 cargo nextest run --locked -p stwo --features prover --no-fail-fast \
    || echo "WARNING: cargo nextest reported failures"
RUN (cd stwo_cairo_verifier && timeout 1200 scarb test -p stwo_verifier_core) \
    || echo "WARNING: scarb test reported failures"

# Everything is fetched. From here on cargo must not try to reach a registry.
ENV CARGO_NET_OFFLINE=true
