# syntax=docker/dockerfile:1
# Match CI: repository workflows avoid 24.16.0 for Playwright installation.
FROM node:24.15.0-bookworm
ENV CI=1 \
    NEXT_TELEMETRY_DISABLED=1 \
    WRANGLER_SEND_METRICS=false \
    PLAYWRIGHT_BROWSERS_PATH=/opt/playwright \
    E2E_DEPENDENCIES_MODE=mock \
    NEXT_PUBLIC_APP_URL=http://localhost:3000
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates build-essential python3 git \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
# Include dev/workspace dependencies and the SQL.js postinstall assets.
RUN npm ci \
    && npx --no-install playwright install --with-deps chromium-headless-shell
# The action has its own lockfile and is not in the root npm workspace.
RUN npm --prefix action ci \
    && npm --prefix action run build
# Direct Next build: no Cloudflare preview/deployment or GitHub credentials.
RUN npm run build:next
# A local-only browser smoke uses the already-built Next output. Setting
# E2E_BASE_URL bypasses Playwright's OpenNext rebuild/webServer command.
COPY <<'EOF' /usr/local/bin/scanner-auth-e2e
#!/bin/sh
set -eu
cd /src
npm run start -- -H 127.0.0.1 -p 3000 &
server_pid=$!
trap 'kill "$server_pid" 2>/dev/null || true' EXIT HUP INT TERM
node --input-type=module <<'JS'
const url = 'http://127.0.0.1:3000';
let ready = false;
for (let i = 0; i < 60; i++) {
  try {
    const response = await fetch(url, { signal: AbortSignal.timeout(1000) });
    if (response.ok) { ready = true; break; }
  } catch {}
  await new Promise(resolve => setTimeout(resolve, 500));
}
if (!ready) throw new Error('local Next server did not become ready');
JS
E2E_BASE_URL=http://127.0.0.1:3000 npm run test:e2e -- e2e/common/stateless-mode/authentication-token-refresh.spec.ts
EOF
RUN chmod +x /usr/local/bin/scanner-auth-e2e
