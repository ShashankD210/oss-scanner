# syntax=docker/dockerfile:1
# CI pins Node 22.22.2; keep every tracked component, not just the hub bundle.
FROM ghcr.io/astral-sh/uv:0.13.0 AS uv
FROM node:22.22.2-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CI=1 \
    WRANGLER_SEND_METRICS=false \
    PLAYWRIGHT_BROWSERS_PATH=/opt/playwright \
    UV_CACHE_DIR=/opt/uv-cache \
    UV_PYTHON_INSTALL_DIR=/opt/uv-python \
    UV_PYTHON=3.12 \
    UV_MANAGED_PYTHON=true \
    UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1
COPY --from=uv /uv /uvx /usr/local/bin/
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 build-essential pkg-config libffi-dev libssl-dev openssl \
    && rm -rf /var/lib/apt/lists/*
RUN uv python install 3.12

# The scanner supplies a clean remote checkout as its build context, never a user's
# live worktree. Do not mount/copy credentials, real transcripts, or client keys.
COPY . /src
WORKDIR /src/hub
RUN npm ci --no-audit --no-fund \
    && ./node_modules/.bin/playwright install --with-deps chromium

# Wrangler's real build path, without uploading code or resolving live resources.
# Keep production/preview entrypoints and both telemetry workers available offline.
RUN node node_modules/wrangler/bin/wrangler.js deploy --dry-run \
      --outdir /src/.oss-build/hub --var ENVIRONMENT:development \
    && node node_modules/wrangler/bin/wrangler.js deploy --dry-run --env preview \
      --outdir /src/.oss-build/preview \
    && node node_modules/wrangler/bin/wrangler.js deploy --dry-run \
      --config wrangler.oidc-issuer.jsonc --outdir /src/.oss-build/oidc-issuer \
    && node node_modules/wrangler/bin/wrangler.js deploy --dry-run \
      --config wrangler.telemetry-gateway.jsonc --outdir /src/.oss-build/telemetry-gateway

# Include pytest, the optional CDP websocket client, and the local package/build backend.
# Wheels/sdists and compiled bytecode are retained; no tests or live enrollment run here.
WORKDIR /src/collector
RUN uv sync --frozen --all-extras \
    && uv build \
    && uv run --no-sync python -m compileall -q src
WORKDIR /src/client
RUN uv sync --frozen \
    && uv build \
    && uv run --no-sync python -m compileall -q src

# Cache BOTH PEP 723 environments without running the enrollment script or its tests.
# These adjacent lockfiles are image-only build artifacts, not repository changes.
WORKDIR /src
RUN uv lock --script /src/infra/cf/enroll-cert.py \
    && uv sync --frozen --script /src/infra/cf/enroll-cert.py \
    && uv lock --script /src/infra/cf/test_enroll_cert.py \
    && uv sync --frozen --script /src/infra/cf/test_enroll_cert.py

# Offline agents can use the installed environments and retained uv/npm caches.
# Network-dependent provisioning and price synchronization are intentionally not run.
ENV UV_OFFLINE=true \
    UV_PYTHON_DOWNLOADS=never \
    npm_config_offline=true \
    npm_config_audit=false \
    npm_config_fund=false
WORKDIR /src
CMD ["bash"]
