# syntax=docker/dockerfile:1
#
# ReqogniLoom — https://github.com/Popoboxxo/ReqogniLoom
# AI-native requirements & test management: Django 6.1 REST API (backend/),
# React 19 SPA (frontend/), PostgreSQL 16 + pgvector, Celery/Redis (not needed
# for the suite: reqogniloom.settings_test pins eager Celery + a locmem cache).
#
# Scanner contract: the checkout is COPY'd to /src. Everything the offline audit
# phase needs — Python and Node dependencies, the embedding model, a local
# PostgreSQL — is installed here while the network is still available.
#
# Test entrypoints (documented in threat_model.md):
#   run-backend-tests     pytest in /src/backend (starts Postgres first)
#   run-frontend-tests    `npm test` (vitest run) in /src/frontend
#   run-all-tests         both of the above, backend first

FROM python:3.12-bookworm

# Test settings from backend/reqogniloom/settings_test.py (imported by
# backend/pyproject.toml): Django 6.1 needs Python >= 3.12; the LLM provider is
# pinned to the deterministic mock, cache is LocMem, Celery runs eager. Only
# PostgreSQL is required as a service; the secret env vars the module does not
# pin itself are generated per process there. LLM_PROVIDER=mock is set as an
# ENV too (same value as testing/docker-compose.test.yml) because
# llm_adapter.providers.get_provider reads os.environ, not the Django setting.
ENV DEBIAN_FRONTEND=noninteractive \
    DJANGO_SETTINGS_MODULE=reqogniloom.settings_test \
    LLM_PROVIDER=mock \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTEST_ADDOPTS="-p no:cacheprovider" \
    HF_HOME=/opt/hf-cache \
    DB_HOST=127.0.0.1 \
    DB_PORT=5432

# System packages:
#   - postgresql-16 (PGDG): matches the project's pgvector/pgvector:pg16 stack
#   - nodejs 22 (NodeSource): frontend engines (Vite 8 / Vitest 5) need >= 20.19
#   - git, build-essential, libpq-dev: the backend requirements include a
#     django-celery-beat VCS pin (needs git at install time) and psycopg2
#     builds against libpq.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl gnupg git build-essential pkg-config libpq-dev \
  && install -d /etc/apt/keyrings /usr/share/keyrings \
  && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
       -o /etc/apt/keyrings/apt.postgresql.org.asc \
  && echo "deb [signed-by=/etc/apt/keyrings/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt bookworm-pgdg main" \
       > /etc/apt/sources.list.d/pgdg.list \
  && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
       | gpg --dearmor -o /usr/share/keyrings/nodesource.gpg \
  && echo "deb [signed-by=/usr/share/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" \
       > /etc/apt/sources.list.d/nodesource.list \
  && apt-get update && apt-get install -y --no-install-recommends \
       postgresql-16 postgresql-server-dev-16 nodejs \
  && rm -rf /var/lib/apt/lists/*

# pgvector from source: migration 0024 runs `CREATE EXTENSION IF NOT EXISTS
# vector` and migrations 0024/0025 add HNSW indexes over vector(384) columns.
RUN git clone --depth 1 --branch v0.8.0 https://github.com/pgvector/pgvector.git /tmp/pgvector \
  && make -C /tmp/pgvector PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config \
  && make -C /tmp/pgvector PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config install \
  && rm -rf /tmp/pgvector

# scanner contract: the checkout lives at /src
COPY . /src
WORKDIR /src

# CPU-only torch from the PyTorch index: the default PyPI wheel bundles the
# NVIDIA CUDA runtime (several GB) this self-hosted deployment never uses —
# same reasoning as backend/Dockerfile in the project.
RUN pip install --no-cache-dir \
      --extra-index-url https://download.pytorch.org/whl/cpu \
      -r /src/backend/requirements.txt

# Bake the sentence-transformers embedding model (all-MiniLM-L6-v2, 384 dims —
# the width migrations 0024/0025/0069 size the columns to) into the image so
# the offline phase never reaches HuggingFace.
RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')" \
  && chmod -R a+rX "${HF_HOME}"

# Pin offline mode only AFTER the model is cached (same ordering as the
# project's backend/Dockerfile): with it set during the download above,
# huggingface_hub would refuse to fetch anything.
ENV HF_HUB_OFFLINE=1

# Frontend dependencies. The repo carries no package-lock.json (npm is used
# directly in the project's own Dockerfiles and CI), so `npm install`.
RUN cd /src/frontend && npm install --no-audit --no-fund

# Test entrypoints. PostgreSQL refuses to run as root, so the cluster is
# managed through pg_ctlcluster, which drops privileges to the postgres user
# (Debian's postinst creates the default "main" cluster on install). The app
# role is a SUPERUSER because the suite's RLS tests run
# `ALTER ROLE "reqogniloom_app" ...` and migration 0024 needs CREATE EXTENSION
# rights — pgvector is not a trusted extension on PG16. This matches the test
# credentials the project's own testing/docker-compose.test.yml pins
# (DB_USER: reqogniloom, superuser, not the least-privilege app role).
RUN <<'EOF'
set -eu
cat >/usr/local/bin/reqlo-start-db <<'SH'
#!/bin/sh
# Start the local PostgreSQL 16 cluster and provision the test database.
# Idempotent: safe to run before every test invocation.
set -eu
if ! pg_lsclusters -h | grep -Eq '^16[[:space:]]+main[[:space:]]'; then
  pg_createcluster 16 main
fi
pg_ctlcluster 16 main start 2>/dev/null || true
attempt=0
until pg_isready -q -h "${DB_HOST:-127.0.0.1}" -p "${DB_PORT:-5432}"; do
  attempt=$((attempt + 1))
  [ "$attempt" -ge 10 ] && exit 1
  sleep 1
done
su postgres -c "psql -v ON_ERROR_STOP=0 -q -c \"CREATE ROLE ${DB_USER:-reqogniloom} LOGIN SUPERUSER PASSWORD '${DB_PASSWORD:-reqogniloom}'\""
su postgres -c "psql -v ON_ERROR_STOP=0 -q -c \"CREATE DATABASE ${DB_NAME:-reqogniloom} OWNER ${DB_USER:-reqogniloom}\""
SH
cat >/usr/local/bin/run-backend-tests <<'SH'
#!/bin/sh
# Backend suite: pytest from /src/backend. pytest-django creates the
# test_reqogniloom database itself (the role is a superuser, so it may).
# Usage: run-backend-tests [extra pytest args, e.g. persistence/tests]
set -eu
reqlo-start-db
# Test-only credentials: reqogniloom/settings.py fails fast when DB_PASSWORD is
# unset, and migration 0048 (app-role) demands DB_APP_PASSWORD with no default
# (the project's CI injects a fixed dummy the same way). SECRET_KEY and
# AUTH_JWT_SECRET are pinned by settings_test itself.
export DB_PASSWORD="${DB_PASSWORD:-reqogniloom}"
export DB_APP_PASSWORD="${DB_APP_PASSWORD:-reqogniloom}"
cd /src/backend
exec pytest "$@"
SH
cat >/usr/local/bin/run-frontend-tests <<'SH'
#!/bin/sh
# Frontend suite: vitest run from /src/frontend.
# theme-contrast.test.ts reads backend/admin_ops/fixtures/themePalettes.light.json
# via resolve(__dirname, "../../.."), which resolves to /src in this layout.
# Usage: run-frontend-tests [extra vitest args, e.g. src/test/theme-contrast.test.ts]
set -eu
cd /src/frontend
exec npm test -- "$@"
SH
cat >/usr/local/bin/run-all-tests <<'SH'
#!/bin/sh
# Both suites, backend first. Exits non-zero if either fails.
set -eu
reqlo-start-db
export DB_PASSWORD="${DB_PASSWORD:-reqogniloom}"
export DB_APP_PASSWORD="${DB_APP_PASSWORD:-reqogniloom}"
cd /src/backend && pytest "$@"
cd /src/frontend && npm test -- "$@"
SH
chmod 0755 /usr/local/bin/reqlo-start-db /usr/local/bin/run-backend-tests \
  /usr/local/bin/run-frontend-tests /usr/local/bin/run-all-tests
EOF

# The scanner expects a long-running container; a tame default that also works
# interactively for local `tools/check` runs.
CMD ["run-all-tests"]
