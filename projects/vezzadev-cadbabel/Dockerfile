FROM node:24-bookworm

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 build-essential sqlite3 \
    && rm -rf /var/lib/apt/lists/*

ENV CI=true \
    WRANGLER_SEND_METRICS=false

COPY . /src
WORKDIR /src

# Node >=24 is the manifest/CI contract. Retain development dependencies and
# both Wrangler's and the Vitest pool's local Worker runtimes for offline tests.
# The WOFF2 font is already tracked in public/; do not run its download script.
# There is no build script: use CI's dry-run bundling command, never deploy.
# No .dev.vars, Stripe keys, Cloudflare token, or remote bindings are required.
RUN npm ci --include=dev --no-audit --no-fund \
    && ./node_modules/.bin/wrangler deploy --dry-run \
       --config wrangler.ppe.jsonc --outdir /src/scanner-build

CMD ["/bin/bash"]
