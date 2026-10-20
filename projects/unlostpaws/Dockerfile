# Scanner image for https://github.com/the-dot-squad/unlostpaws (Next.js, JavaScript, no TypeScript).
# `next build` is deliberately skipped: it prerenders pages that need a live MongoDB, which does not
# exist in the scanner VM. Audit the source and run code paths directly (node, next lint) instead.
FROM node:24-bookworm

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# postinstall copies a wasm file into public/ and applies patches/ via patch-package, so the whole
# checkout must be present before this step.
RUN npm ci --no-audit --no-fund

# Cheap sanity check; do not let lint findings abort the build.
RUN npm run lint || echo "WARNING: eslint reported problems"
