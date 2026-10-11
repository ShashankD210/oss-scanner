FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers \
    BSKY_NOT_BLOCKED_LIVE=0

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git zip firefox-esr \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src
# Both browser manifests use the shared plain JavaScript source; the build also
# creates archives. Install the real Chromium binary, not just headless-shell,
# so offline browser experiments can load the extension.
RUN npm ci --include=dev \
    && npx --no-install playwright install --with-deps chromium \
    && npm run build \
    && test -s dist/chrome/manifest.json \
    && test -s dist/firefox/manifest.json \
    && test -s "dist/bsky-not-blocked-chrome-$(node -p 'require("./package.json").version').zip" \
    && test -s "dist/bsky-not-blocked-firefox-$(node -p 'require("./package.json").version').zip"

# All dependency/browser downloads happen above. Do not run live fixtures or E2E
# scripts here: they navigate real bsky.app and call external public services.
ENV npm_config_offline=true
CMD ["bash"]
