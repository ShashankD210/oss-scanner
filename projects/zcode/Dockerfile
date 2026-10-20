FROM node:24.14.0-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CI=1 \
    HUSKY=0 \
    TURBO_TELEMETRY_DISABLED=1 \
    NODE_OPTIONS=--max-old-space-size=8192 \
    PNPM_CONFIG_WORKSPACE_CONCURRENCY=1

# Keep native build tools and Electron/Xvfb libraries available to the offline
# auditor, including for new regression tests and patched builds.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git python3 python3-venv build-essential \
      pkg-config openssh-client procps file jq unzip zip xz-utils \
      dbus dbus-x11 xauth xvfb xdg-utils \
      fonts-noto-cjk fonts-noto-color-emoji \
      libasound2 libatk-bridge2.0-0 libatk1.0-0 libatspi2.0-0 \
      libcairo2 libcups2 libdbus-1-3 libdrm2 libgbm1 libglib2.0-0 \
      libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 \
      libsecret-1-0 libx11-6 libx11-xcb1 libxcb-dri3-0 \
      libxcomposite1 libxdamage1 libxext6 libxfixes3 libxkbcommon0 \
      libxkbfile1 libxrandr2 libxrender1 libxshmfence1 libxss1 libxtst6 \
    && rm -rf /var/lib/apt/lists/*

RUN corepack enable && corepack prepare pnpm@10.33.2 --activate

COPY . /src
WORKDIR /src

# Resolve all dependencies and native install hooks while network is available.
# Keep development dependencies and the package store for offline investigations.
RUN pnpm install --frozen-lockfile

# Use the repository's build owners, including the Agent and first-party plugin
# runtimes, rather than replacing product code or requiring cloud credentials.
RUN pnpm prepare:desktop-runtime
RUN pnpm build:bootstrap

# Confirm the main runtime and application bundles are available to the scanner.
RUN test -s apps/zcode-cli/packages/cli/dist/zcode.cjs \
    && test -s packages/server/dist/entry-http.js \
    && test -s packages/desktop/out/main/index.js \
    && test -s packages/web/dist/index.html

CMD ["bash"]
