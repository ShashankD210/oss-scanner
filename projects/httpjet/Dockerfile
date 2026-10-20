# Builds httpjet (a Rust, io_uring-based web server) from the public repository checkout.
FROM debian:bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_HOME=/opt/cargo \
    RUSTUP_HOME=/opt/rustup \
    PATH=/opt/cargo/bin:$PATH
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl build-essential clang cmake pkg-config git python3 perl \
    && rm -rf /var/lib/apt/lists/*

# httpjet declares rust-version 1.99 (edition 2024).
RUN curl -fsSL https://sh.rustup.rs | sh -s -- -y --profile minimal --default-toolchain 1.99.0

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Debug info and frame pointers for the scanner's debuggers. Dependencies are fetched here, while the network is
# available, so the later offline audit can still rebuild and run tests.
ENV CARGO_PROFILE_RELEASE_DEBUG=1 \
    RUSTFLAGS="-C force-frame-pointers=yes"
RUN cargo fetch --locked \
    && cargo build --release --locked -p httpjet

# Pre-compile the workspace tests so they can be run offline; a failure here must not abort the build.
RUN cargo test --workspace --no-run --locked || echo "WARNING: test build reported failures"
RUN cargo test --workspace --locked -- --test-threads=2 || echo "WARNING: cargo test reported failures"
