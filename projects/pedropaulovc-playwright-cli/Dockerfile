FROM node:22-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    PLAYWRIGHT_BROWSERS_PATH=/ms-playwright \
    PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
    npm_config_audit=false \
    npm_config_fund=false

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 procps xvfb xauth \
    && rm -rf /var/lib/apt/lists/*

# Retain the entire default-branch fork checkout, including tests and skill docs.
COPY . /src
WORKDIR /src

# This repository is a plain-JavaScript CLI launcher with no build script.
# The committed lockfile still describes the upstream package/dependency graph.
# Ignore that stale lock without editing it: install the current manifest's
# aliased fork runtime/test packages, including their compiled JavaScript.
# Published runtime code imports the bare name playwright-core, but its manifest
# declares only @pedropaulovc/playwright-core. Install the genuine matching fork
# package under the imported name without saving or rewriting source manifests.
# The dependency's browser installer, not an upstream/global CLI, provisions
# all bundled engines plus the Chrome channel used by CLI defaults/tests.
RUN npm install --include=dev --no-save --package-lock=false \
      playwright-core@npm:@pedropaulovc/playwright-core@1.59.0-next.9 \
    && node node_modules/playwright/cli.js install --with-deps chromium firefox webkit chrome \
    && ln -s /src/playwright-cli.js /usr/local/bin/playwright-cli \
    && chmod -R a+rX /ms-playwright

# No credentials, user browser profiles, extensions, or remote endpoints are
# provisioned. Tests and synthetic CLI interactions run after the online build,
# in the scanner's isolated container with networking disabled.
CMD ["bash"]
