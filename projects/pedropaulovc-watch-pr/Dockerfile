FROM node:24-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    WRANGLER_SEND_METRICS=false \
    npm_config_audit=false \
    npm_config_fund=false

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git curl python3 build-essential libatomic1 libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# The scanner supplies a clean repository checkout, not an operator worktree.
# Keep the whole source, tests, scripts, docs, and locked development dependencies.
COPY . /src
WORKDIR /src

# Install optional platform packages and run dependency install scripts while online:
# workerd, esbuild, and Rollup must already be usable when networking is removed.
RUN npm ci --include=dev --include=optional --no-audit --no-fund

# Match the PR workflow's esbuild ESM/browser/ES2022 bundle, retaining source maps.
# Also build the repository's separate telemetry gateway and OIDC issuer Workers.
# No Cloudflare deployment, OAuth exchange, or telemetry destination setup is run.
RUN mkdir -p build \
    && ./node_modules/.bin/esbuild src/index.ts --bundle --format=esm --platform=browser --target=es2022 --sourcemap=external --sources-content=true --outfile=build/worker.js \
    && ./node_modules/.bin/esbuild src/telemetry-gateway.ts --bundle --format=esm --platform=browser --target=es2022 --sourcemap=external --sources-content=true --outfile=build/telemetry-gateway.js \
    && ./node_modules/.bin/esbuild src/oidc-issuer.ts --bundle --format=esm --platform=browser --target=es2022 --sourcemap=external --sources-content=true --outfile=build/oidc-issuer.js

# Run the existing tests and the isolated request smoke documented in threat_model.md
# after the online build, under the scanner's network-disabled environment.
CMD ["/bin/bash"]
