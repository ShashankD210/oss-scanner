# syntax=docker/dockerfile:1
FROM rust:1-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_TERM_COLOR=never \
    RUST_BACKTRACE=1
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential clang pkg-config libssl-dev cmake \
      libdbus-1-dev libxcb1-dev libxcb-randr0-dev libxcb-shm0-dev libxcb-composite0-dev python3 \
    && rm -rf /var/lib/apt/lists/*

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

# Fetch every crate up front so later builds work offline.
RUN cargo fetch --locked

# Build all targets in debug mode (retains debug info and assertions).
RUN cargo build --locked --all-targets

# Compile tests so they are usable offline.
RUN cargo test --locked --no-run

# Run the unit tests as a smoke check. Failures do not abort the build.
RUN (timeout 1800 cargo test --locked --lib || echo "WARNING: selfware unit tests reported failures")
