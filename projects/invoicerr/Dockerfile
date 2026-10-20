# syntax=docker/dockerfile:1
FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 postgresql redis-server \
    && rm -rf /var/lib/apt/lists/*

COPY . /src

WORKDIR /src/backend
RUN npm ci \
    && DATABASE_URL=not_required ./node_modules/.bin/prisma generate \
    && npm run build \
    && ./node_modules/.bin/playwright-core install --with-deps chromium

WORKDIR /src/frontend
RUN npm ci && npm run build

RUN pg_ctlcluster 15 main start \
    && su postgres -c "psql -c \"CREATE USER invoicerr WITH PASSWORD 'invoicerr' SUPERUSER\"" \
    && su postgres -c "createdb -O invoicerr invoicerr_db" \
    && pg_ctlcluster 15 main stop

COPY <<'SCRIPT' /usr/local/bin/invoicerr-start
#!/bin/sh
# Starts PostgreSQL, Redis and the backend on http://127.0.0.1:4000, then creates two users, each
# owning a separate company. Safe to run again. Users and cookies: /opt/invoicerr/users.json.
set -e
pg_ctlcluster 15 main start 2>/dev/null || true
redis-server --daemonize yes >/dev/null 2>&1 || true
until pg_isready -q; do sleep 1; done
until redis-cli ping >/dev/null 2>&1; do sleep 1; done

cd /src/backend
export DATABASE_URL=postgresql://invoicerr:invoicerr@127.0.0.1:5432/invoicerr_db
export REDIS_URL=redis://127.0.0.1:6379
if ! curl -sf http://127.0.0.1:4000/api/health >/dev/null; then
  ./node_modules/.bin/prisma migrate deploy >/dev/null
  PATH="$PWD/node_modules/.bin:$PATH" ./node_modules/.bin/prisma db seed >/dev/null
  ./node_modules/.bin/dotenv -e .env.test -- node --enable-source-maps dist/src/main \
    > /var/log/invoicerr-backend.log 2>&1 &
  for _ in $(seq 90); do curl -sf http://127.0.0.1:4000/api/health >/dev/null && break; sleep 2; done
fi
curl -sf http://127.0.0.1:4000/api/health >/dev/null || { tail -n 50 /var/log/invoicerr-backend.log; exit 1; }

mkdir -p /opt/invoicerr
[ -s /opt/invoicerr/users.json ] || node /src/e2e/load/seed.js http://127.0.0.1:4000 2 /opt/invoicerr/users.json
echo "backend: http://127.0.0.1:4000  users: /opt/invoicerr/users.json  log: /var/log/invoicerr-backend.log"
SCRIPT
RUN chmod +x /usr/local/bin/invoicerr-start

WORKDIR /src/backend
ENV DATABASE_URL=postgresql://invoicerr:invoicerr@127.0.0.1:5432/invoicerr_db \
    REDIS_URL=redis://127.0.0.1:6379
RUN pg_ctlcluster 15 main start && redis-server --daemonize yes \
    && ./node_modules/.bin/prisma migrate deploy \
    && PATH="$PWD/node_modules/.bin:$PATH" ./node_modules/.bin/prisma db seed \
    && (./node_modules/.bin/vitest run --maxWorkers=2 || echo "WARNING: backend tests reported failures") \
    && redis-cli shutdown nosave && pg_ctlcluster 15 main stop

WORKDIR /src