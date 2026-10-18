# Scanner image for https://github.com/oikonome/oikonome (main branch).
# Oikonome is a Python 3.14 / FastAPI server with a React (Vite) web app it serves, a Postgres
# database with row-level security per household, and an Expo mobile app whose pure logic is
# tested under plain Node. While the network is available this build installs every dependency,
# builds the web app, installs the server into the virtualenv its Makefile expects, and sets up a
# local Postgres on the port the test suite expects, so the audit can lint, test and run the server
# offline. threat_model.md, next to this file, describes how to exercise the result.

FROM python:3.14-slim
ENV DEBIAN_FRONTEND=noninteractive PIP_NO_CACHE_DIR=1 PYTHONUNBUFFERED=1

# Postgres (the suite needs a real server: row-level security is the tenant isolation under
# test), Node 22 for the web build and the mobile logic tests, git + a compiler for the odd wheel.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl gnupg git make build-essential procps \
      postgresql postgresql-contrib \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src
RUN git config --system --add safe.directory /src

# The web app, built as the product's own Dockerfile builds it; the server serves webapp/dist.
RUN cd webapp && npm ci --silent && npm run build

# The mobile app's dependencies, so its logic tests (plain Node, no device) resolve their imports.
RUN cd mobile && npm ci --silent --no-audit --no-fund

# The server, editable, into the virtualenv the Makefile's lint/test targets use
# (server/.venv); [dev] adds ruff, which `make lint` runs before the suite.
RUN python -m venv server/.venv && server/.venv/bin/pip install -e './server[dev]'

# The test database: the suite reaches postgres/devpass on 127.0.0.1:5433 (server/tests/util.py)
# and creates its own databases and the least-privilege app role from there.
RUN PGV="$(ls /usr/lib/postgresql | head -1)" \
    && sed -i "s/^port = .*/port = 5433/" "/etc/postgresql/$PGV/main/postgresql.conf" \
    && service postgresql start \
    && su postgres -c "psql -p 5433 -c \"ALTER USER postgres PASSWORD 'devpass'\"" \
    && service postgresql stop

# One command runs the whole check offline: start Postgres, lint, the server suite (~8,600 tests,
# 15–40 minutes), the mobile logic tests.
RUN printf '%s\n' '#!/bin/sh' 'set -e' 'service postgresql start >/dev/null' 'cd /src' \
      'make lint' 'make test' \
      'node --experimental-strip-types mobile/scripts/pure-logic-tests.mjs' \
      > /usr/local/bin/oikonome-tests && chmod +x /usr/local/bin/oikonome-tests

# Cheap proof the image works: a handful of suite modules against the local Postgres, and the
# mobile logic tests. A failure here warns rather than aborting the build.
RUN service postgresql start >/dev/null \
    && (cd server && .venv/bin/python -m unittest tests.test_wizard tests.test_onboarding \
          || echo "WARNING: server tests reported failures") \
    && (node --experimental-strip-types mobile/scripts/pure-logic-tests.mjs >/dev/null \
          || echo "WARNING: mobile logic tests reported failures") \
    && service postgresql stop

# Running the server offline (no bank aggregator, no mail): see threat_model.md "How to exercise it".
ENV OIKONOME_SPA_DIST=/src/webapp/dist
