FROM rust:1-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_TERM_COLOR=never \
    CARGO_NET_RETRY=5
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential clang cmake pkg-config git python3 \
      libasound2-dev libudev-dev libxkbcommon-dev libwayland-dev libx11-dev \
      libxcursor-dev libxi-dev libxrandr-dev libxcb1-dev libvulkan-dev \
      libssl-dev mesa-vulkan-drivers \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN cargo fetch --locked
RUN cargo build --locked --all-targets
RUN cargo test --locked --all-targets --no-fail-fast || echo "WARNING: cargo test reported failures"
