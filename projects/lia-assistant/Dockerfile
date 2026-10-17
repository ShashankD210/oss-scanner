# LIA (https://github.com/jgouviergmail/LIA-Assistant) for the OSS Scanner.
#
# One image holds the whole monorepo at /src with every dependency installed, so the audit runs offline:
#   - apps/api: the FastAPI + LangGraph backend (Python 3.14), its development lockfile installed in apps/api/.venv;
#   - apps/web: the Next.js 16 frontend (Node.js 24, pnpm 10), its node_modules and a production build;
#   - a local PostgreSQL 17 with pgvector and a local Redis, started on demand by `lia-services start` at the
#     addresses apps/api/.env.test names, so the integration tests and a running backend need no network;
#   - `lia-api` applies the migrations and serves the backend on http://127.0.0.1:8000 with the test settings;
#     `lia-web` serves the built frontend on http://127.0.0.1:3000, proxying /api/v1 to that backend.
# The base image, Node.js and pnpm are the versions the project's own images pin (apps/api/Dockerfile.prod,
# apps/web/Dockerfile.prod).

FROM python:3.14.7-slim-trixie@sha256:51dafde81dbdb6ebde285137a295cf18a47ca95234fe388a343719cb97305b3d

# TEST_DATABASE_URL and TEST_REDIS_URL point the test suites at the bundled services. Without them, the test
# configuration reads /.dockerenv as the development Compose network and looks for a host named `redis`.
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_ROOT_USER_ACTION=ignore \
    NEXT_TELEMETRY_DISABLED=1 \
    TIKTOKEN_CACHE_DIR=/opt/tiktoken-cache \
    BROWSER_CHROMIUM_EXECUTABLE=/usr/bin/chromium \
    TEST_DATABASE_URL=postgresql+asyncpg://test:test@127.0.0.1:5432/test_db \
    TEST_REDIS_URL=redis://127.0.0.1:6379/15 \
    PATH=/src/apps/api/.venv/bin:$PATH

# Build tools; ffmpeg (audio conversion on the voice paths); Chromium (the browsing tool); PostgreSQL with pgvector
# and Redis (the two services the backend needs).
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git xz-utils build-essential pkg-config procps \
      ffmpeg chromium \
      postgresql postgresql-contrib postgresql-17-pgvector redis-server \
    && rm -rf /var/lib/apt/lists/*

# Node.js from its official release, checked against its published digest, then the pnpm the workspace declares.
ARG NODE_VERSION=24.21.0
ARG NODE_SHA256_AMD64=fd8e59d5a511510f6a298afb548f18c7d2b1be404d8b4a27d94fbe49f56cb2d6
ARG NODE_SHA256_ARM64=6ad1325edbdb5649c379b75a237147a666c95d4f9ae8d340fef2d1575d289ad2
RUN set -eux; \
    case "$(dpkg --print-architecture)" in \
      amd64) node_arch=x64; sha="${NODE_SHA256_AMD64}" ;; \
      arm64) node_arch=arm64; sha="${NODE_SHA256_ARM64}" ;; \
      *) echo "unsupported architecture: $(dpkg --print-architecture)" >&2; exit 1 ;; \
    esac; \
    file="node-v${NODE_VERSION}-linux-${node_arch}.tar.xz"; \
    curl -fsSL --retry 5 --retry-all-errors -o "/tmp/${file}" "https://nodejs.org/dist/v${NODE_VERSION}/${file}"; \
    echo "${sha}  /tmp/${file}" | sha256sum -c -; \
    tar -xJf "/tmp/${file}" -C /usr/local --strip-components=1 --no-same-owner; \
    rm -f "/tmp/${file}"; \
    npm install -g pnpm@10.34.6; \
    node --version; pnpm --version

WORKDIR /src

# Backend: the development lockfile (runtime and test tools), hash-checked, in the virtualenv the project's
# tasks call (apps/api/.venv), installed exactly as CI installs it.
COPY apps/api/requirements-dev.lock.txt /tmp/requirements-dev.lock.txt
RUN python -m venv /src/apps/api/.venv \
    && URLLIB3_NO_OVERRIDE=1 /src/apps/api/.venv/bin/pip install --require-hashes --no-binary urllib3-future \
         -r /tmp/requirements-dev.lock.txt \
    && rm /tmp/requirements-dev.lock.txt
# tiktoken downloads an encoding on first use: fetch the two the backend counts tokens with.
RUN python -c "import tiktoken; [tiktoken.get_encoding(name) for name in ('o200k_base', 'cl100k_base')]"

# Frontend: the workspace manifests, then the frozen lockfile, as apps/web/Dockerfile.prod installs it.
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc /src/
COPY patches /src/patches
COPY apps/web/package.json /src/apps/web/package.json
RUN pnpm install --frozen-lockfile --filter "@lia-assistant/web..."

# scanner contract: the checkout lives inside the image, at /src
COPY . /src

# The scanner deletes the repository's .dockerignore before this build: restore it, so /src is the clean checkout
# (a guard test reads it). Give git an identity, as a developer's machine has: tests that create commits need one.
RUN { [ -f /src/.dockerignore ] || git -C /src checkout -- .dockerignore; } \
    && git config --system user.name "OSS Scanner" \
    && git config --system user.email "oss-scanner@example.invalid"

# The frontend's production build, same-origin as in production: the browser calls /api/v1 on the frontend, which
# proxies it to the local backend. The standalone copy (a deployment artefact that duplicates the server output)
# and the build cache are dropped: together they are 3.4 GB. The backend does not need the bundle, so a failure
# is reported, not fatal.
RUN cd /src/apps/web \
    && (API_URL_SERVER=http://127.0.0.1:8000 NEXT_PUBLIC_API_URL= NODE_OPTIONS=--max-old-space-size=8192 \
          pnpm run build \
        && rm -rf .next/standalone .next/cache \
        || echo "WARNING: the Next.js production build failed; see its output above")

# The test database apps/api/.env.test names (role test/test, database test_db), with the extensions the
# deployed database creates (infrastructure/docker/postgres-init.sql).
RUN set -eux; \
    pg_ctlcluster 17 main start; \
    su postgres -c "psql -v ON_ERROR_STOP=1 -c \"CREATE ROLE test LOGIN SUPERUSER PASSWORD 'test'\""; \
    su postgres -c "createdb --owner=test test_db"; \
    su postgres -c "psql -d test_db -v ON_ERROR_STOP=1 -c 'CREATE EXTENSION IF NOT EXISTS vector' -c 'CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\"' -c 'CREATE EXTENSION IF NOT EXISTS pg_trgm'"; \
    pg_ctlcluster 17 main stop

COPY --chmod=755 <<'EOF' /usr/local/bin/lia-services
#!/bin/bash
# lia-services start|stop|status: the PostgreSQL and Redis bundled in this image, at the addresses the test
# settings name (postgresql://test:test@127.0.0.1:5432/test_db, Redis on 127.0.0.1:6379 without a password).
set -euo pipefail
case "${1:-}" in
  start)
    pg_ctlcluster 17 main start 2>/dev/null || true
    for _ in $(seq 1 60); do pg_isready -q -h 127.0.0.1 -p 5432 && break; sleep 1; done
    pg_isready -h 127.0.0.1 -p 5432
    if ! redis-cli -h 127.0.0.1 ping >/dev/null 2>&1; then
      redis-server --daemonize yes --bind "127.0.0.1 -::1" --port 6379 --save "" --appendonly no
    fi
    for _ in $(seq 1 30); do redis-cli -h 127.0.0.1 ping >/dev/null 2>&1 && break; sleep 1; done
    echo "redis: $(redis-cli -h 127.0.0.1 ping)"
    ;;
  stop)
    redis-cli -h 127.0.0.1 shutdown nosave 2>/dev/null || true
    pg_ctlcluster 17 main stop 2>/dev/null || true
    ;;
  status)
    pg_isready -h 127.0.0.1 -p 5432 || true
    echo "redis: $(redis-cli -h 127.0.0.1 ping 2>&1 || true)"
    ;;
  *)
    echo "usage: lia-services start|stop|status" >&2
    exit 2
    ;;
esac
EOF

COPY --chmod=755 <<'EOF' /usr/local/bin/lia-api
#!/bin/bash
# lia-api [uvicorn options]: serve the backend offline on http://127.0.0.1:8000 against the bundled services,
# with the stub settings of apps/api/.env.test (no provider credentials), after applying the migrations.
set -euo pipefail
lia-services start
cd /src/apps/api
set -a
. ./.env.test
set +a
alembic upgrade head
exec uvicorn src.main:app --host 127.0.0.1 --port 8000 "$@"
EOF

COPY --chmod=755 <<'EOF' /usr/local/bin/lia-web
#!/bin/bash
# lia-web: serve the production build of the frontend on http://127.0.0.1:3000, proxying /api/v1 to the backend
# `lia-api` serves. Next.js warns that `next start` does not match `output: standalone`: the standalone copy was
# removed from this image, and `next start` serves the same build.
set -euo pipefail
cd /src/apps/web
export API_URL_SERVER=http://127.0.0.1:8000 NEXT_PUBLIC_API_URL=
exec pnpm exec next start -H 127.0.0.1 -p 3000 "$@"
EOF

# A cheap check that the image works: every backend unit-test module imports with the installed dependencies.
RUN cd /src/apps/api \
    && if pytest tests/unit --collect-only -q --no-cov -p no:cacheprovider > /tmp/collect.log 2>&1; then \
         awk -F': ' '$2 ~ /^[0-9]+$/ {total += $2} END {print total " backend unit tests collected"}' /tmp/collect.log; \
       else \
         tail -n 40 /tmp/collect.log; echo "WARNING: collecting the backend unit tests failed"; \
       fi; \
    rm -f /tmp/collect.log
