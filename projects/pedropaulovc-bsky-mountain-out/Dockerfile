# syntax=docker/dockerfile:1
FROM node:22-bookworm-slim

ENV CI=true \
    WRANGLER_SEND_METRICS=false

# The scanner supplies the public repository checkout as the build context.
# Never add local .dev.vars, .env files, or cloud credentials to that context.
COPY . /src
WORKDIR /src

# Install the lockfile's complete development/runtime dependencies, including
# Wrangler's esbuild/workerd packages and Photon's WebAssembly image runtime.
# --dry-run bundles the actual Worker without publishing or using cloud APIs.
RUN npm ci --include=dev
RUN --network=none npm run check \
    && ./node_modules/.bin/wrangler deploy --dry-run --outdir /src/.oss-scanner-build

# The existing Vitest suite is run by the scanner after network removal:
# npm test
CMD ["npm", "test"]
