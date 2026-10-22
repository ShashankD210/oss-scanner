# syntax=docker/dockerfile:1
FROM node:24.19.0-bookworm
RUN apt-get update && apt-get install -y --no-install-recommends git ca-certificates python3 \
    && rm -rf /var/lib/apt/lists/* \
    && npm install --global corepack@0.34.1 && corepack enable
COPY . /src
WORKDIR /src
# The lockfile and local vendor directories are authoritative; no private tokens are needed.
RUN yarn install --immutable
RUN yarn build:extension
COPY --chmod=0755 <<'SH' /usr/local/bin/scanner-core-test
#!/bin/sh
set -eu
cd /src
exec yarn vitest run --maxWorkers=2 \
 tests/unit/iroha-tabs-authorization.spec.ts tests/unit/wallet-security-provenance.spec.ts \
 tests/unit/keyring-json.spec.ts tests/unit/legacy-keyring-upgrade.spec.ts \
 tests/unit/bitcoin-address.spec.ts tests/unit/ton-wallet-security.spec.ts
SH
RUN scanner-core-test
ENV COREPACK_ENABLE_NETWORK=0 \
    YARN_ENABLE_NETWORK=0
