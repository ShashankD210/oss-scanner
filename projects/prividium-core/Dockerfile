# Prividium Core: a pnpm/turbo workspace (Fastify permissions API, public SDK, support packages). Mirrors
# .github/workflows/ci.yml: frozen-lockfile install without scripts, SDK build, OpenAPI export, turbo build, then
# vitest against PostgreSQL on localhost:5500 (packages/test-deps hardcodes that URL and clones a template database
# per worker).

# Node 22.16.0 (.nvmrc); pnpm 10.28.1 (package.json packageManager).
FROM node:22.16.0-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN npm install -g pnpm@10.28.1 && npm cache clean --force

# PostgreSQL 18 from PGDG, the version CI runs. The suite needs port 5500, password `notsecurepassword`, many
# connections and no durability; mmap shared memory keeps it independent of the container's /dev/shm size.
RUN install -d /usr/share/postgresql-common/pgdg \
    && curl -fsSL --retry 5 --retry-all-errors https://www.postgresql.org/media/keys/ACCC4CF8.asc \
       -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
    && echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt bookworm-pgdg main" \
       > /etc/apt/sources.list.d/pgdg.list \
    && apt-get update && apt-get install -y --no-install-recommends postgresql-18 \
    && rm -rf /var/lib/apt/lists/* \
    && sed -i "s/^port = .*/port = 5500/" /etc/postgresql/18/main/postgresql.conf \
    && printf '%s\n' "max_connections = 1000" "fsync = off" "synchronous_commit = off" \
         "full_page_writes = off" "dynamic_shared_memory_type = mmap" \
       >> /etc/postgresql/18/main/postgresql.conf \
    && pg_ctlcluster 18 main start \
    && su postgres -c "psql -p 5500 -c \"ALTER USER postgres PASSWORD 'notsecurepassword'\"" \
    && pg_ctlcluster 18 main stop

# The tests expect the database up; run this first in any shell, then `pnpm test`.
RUN printf '%s\n' '#!/bin/sh' \
      'pg_ctlcluster 18 main start 2>/dev/null || true' \
      'until pg_isready -h localhost -p 5500 -U postgres >/dev/null 2>&1; do sleep 0.2; done' \
      'echo "postgres ready on localhost:5500"' \
      > /usr/local/bin/start-test-db \
    && chmod +x /usr/local/bin/start-test-db

COPY . /src
WORKDIR /src

ENV TURBO_TELEMETRY_DISABLED=1 DO_NOT_TRACK=1

# As CI: install, build the SDK first (the permissions API imports its dist/), export the OpenAPI spec
# (@repo/api-types builds from it and it is not committed), then build everything.
RUN pnpm install --frozen-lockfile --ignore-scripts
RUN pnpm --filter @repo/prividium-sdk build
RUN pnpm --filter @repo/permissions-api run export-openapi-spec
RUN pnpm build

# Nothing below this line, or in the scan, may reach the network.

# Smoke-test the suite as CI runs it. Test failures do not fail the build.
RUN start-test-db \
    && (timeout 2400 pnpm test || echo "WARNING: pnpm test reported failures or timed out") \
    && pg_ctlcluster 18 main stop

# Start the scan from a tree identical to HEAD, so a patch carries only its own changes.
RUN git checkout -- .
