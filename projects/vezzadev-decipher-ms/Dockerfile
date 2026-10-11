FROM node:24-bookworm

# Keep native module tooling and local HTTP/SQL inspection available offline.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 build-essential sqlite3 \
    && rm -rf /var/lib/apt/lists/*

ENV CI=true \
    ASTRO_TELEMETRY_DISABLED=1 \
    WRANGLER_SEND_METRICS=false

# The scanner supplies the public repository checkout as the build context.
COPY . /src
WORKDIR /src

# Keep dev dependencies, Wrangler/Miniflare/workerd, and generated font/static
# assets. Astro's configured Google font provider needs network at build time.
# No .dev.vars, Cloudflare login, remote D1, or runtime secrets are needed.
RUN npm ci --include=dev --no-audit --no-fund \
    && npm run build

CMD ["/bin/bash"]
