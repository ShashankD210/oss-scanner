# Rendu: Tauri 2 (Rust) + React/TypeScript/Vite desktop Markdown viewer. Mirrors .github/workflows/ci.yml.
FROM node:22-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    CARGO_HOME=/usr/local/cargo \
    RUSTUP_HOME=/usr/local/rustup \
    PATH=/usr/local/cargo/bin:$PATH
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential curl wget file git pkg-config \
      libwebkit2gtk-4.1-dev libxdo-dev libssl-dev libayatana-appindicator3-dev librsvg2-dev \
    && rm -rf /var/lib/apt/lists/*
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal --default-toolchain stable

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Frontend dependencies and build
RUN npm ci
RUN npx tsc -b && npx vite build
# Rust dependencies (fetched now; the audit runs offline) and build
RUN cd src-tauri && cargo fetch --locked && cargo build --locked
# Smoke tests; failures are logged, not fatal.
RUN npm run test:run || echo "WARNING: vitest reported failures"
RUN cd src-tauri && (cargo test --lib || echo "WARNING: cargo test reported failures")
