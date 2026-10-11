FROM rust:1-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_TERM_COLOR=never \
    CARGO_NET_RETRY=5 \
    CARGO_INCREMENTAL=0 \
    CARGO_PROFILE_DEV_DEBUG=line-tables-only
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential clang cmake pkg-config git python3 \
      libasound2-dev libudev-dev libxkbcommon-dev libwayland-dev libx11-dev \
      libxcursor-dev libxi-dev libxrandr-dev libxcb1-dev libvulkan-dev \
      libssl-dev mesa-vulkan-drivers libgtk-3-dev libwebkit2gtk-4.1-dev \
      libsoup-3.0-dev libjavascriptcoregtk-4.1-dev libxdo-dev \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# rust-toolchain.toml pins the compiler; the first cargo call installs it. Fetch every
# dependency now, because the scan runs without network access.
RUN cargo fetch --locked
# The demo apps under apps/ and xtask are left out of the image. They are the least important code, and their binaries and
# examples make target/ several times larger. Build and test everything else, in one layer. Compile every test binary so
# that tests run offline. Run the quick tests under a time limit. The GPU tests of cranpose-render-wgpu render on the CPU
# and take many minutes, so they are compiled here and left for the scan to run. A failing test does not fail the build.
ENV SKIP_APPS="--exclude desktop-app --exclude desktop-app-platform --exclude cranpose-native-demo --exclude coroflow-demo --exclude coroflow-demo-android --exclude coroflow-demo-web --exclude xtask"
RUN cargo build --locked --workspace $SKIP_APPS \
 && cargo test --locked --workspace $SKIP_APPS --no-run \
 && (timeout 900 cargo test --locked --workspace $SKIP_APPS --exclude cranpose-render-wgpu --no-fail-fast || echo "WARNING: cargo test reported failures or timed out") \
 && (timeout 300 cargo test --locked -p cranpose-render-wgpu --lib || echo "WARNING: cargo test reported failures or timed out")
