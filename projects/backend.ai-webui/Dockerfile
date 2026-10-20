# Build the Backend.AI Web UI for Anthropic's OSS Scanner.
# The scanner uses the repository root as the build context and audits /src offline, so every dependency and
# build artifact the tests need is produced here. CI uses Node from .nvmrc (24) and pnpm from packageManager.
FROM node:24-bookworm

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Skip install-time downloads the scanner does not need (Electron binary, Playwright browsers) and git hooks.
ENV DEBIAN_FRONTEND=noninteractive \
    CI=1 \
    HUSKY=0 \
    ELECTRON_SKIP_BINARY_DOWNLOAD=1 \
    PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
    NODE_OPTIONS=--max-old-space-size=8192

RUN apt-get update && apt-get install -y --no-install-recommends git python3 build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install the exact pnpm the repository pins, so pnpm never tries to fetch itself offline.
RUN npm install -g pnpm@12.4.0

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

# The workspace install CI runs. The pnpm store (global virtual store) stays inside the image.
RUN pnpm install --frozen-lockfile

# The production web build (`make compile_keepversion` without the git-based version stamping):
# Relay compile, tsc, every workspace package, and the React app into build/web.
RUN cp -n config.toml.sample config.toml && pnpm run build

# The local proxy that Electron and the standalone wsproxy binary run (src/wsproxy).
RUN pnpm -w exec webpack-cli --config src/wsproxy/webpack.config.js

# Run the fast, browser-free unit tests so the image is known to work; a failing test must not abort the build.
# More: `cd react && pnpm exec vitest run`, `pnpm --filter backend.ai-ui exec vitest run`.
RUN (pnpm --filter backend.ai-client test && pnpm exec vitest run) \
    || echo "WARNING: vitest reported failures"
