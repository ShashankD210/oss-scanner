# OIBus: three npm projects (backend/, frontend/, launcher/) on the Node version pinned in their package.json
# `engines` field. Mirrors .github/workflows/tests.yml.
FROM node:26-trixie
ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    PLAYWRIGHT_BROWSERS_PATH=/ms-playwright
# build-essential and python3 build the native modules (better-sqlite3, argon2, isolated-vm) when no prebuild
# matches; unixodbc-dev is needed by the optional odbc dependency.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 build-essential unixodbc-dev \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN cd backend && npm ci
RUN cd frontend && npm ci
RUN cd launcher && npm ci
# Frontend tests run in Vitest browser mode, so Chromium must be fetched while the network is up.
RUN cd frontend && npx playwright install --with-deps chromium

# The frontend build outputs to backend/dist/frontend, where the backend serves it from.
RUN cd frontend && npm run build
RUN cd backend && npm run build
RUN cd launcher && npm run build

# A data folder for a local instance: `cd /src/backend && node dist/backend/src/index.js --config /src/data-folder`
# then serves the UI and API on http://127.0.0.1:2223 (first login admin / pass).
RUN mkdir -p /src/data-folder

# Smoke-test the image; failures are logged, not fatal.
RUN cd backend && (npm test || echo "WARNING: backend tests reported failures")
RUN cd launcher && (npm test || echo "WARNING: launcher tests reported failures")
