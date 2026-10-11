FROM node:24-bookworm

# Keep native module tooling and local HTTP/SQL inspection available offline.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 build-essential sqlite3 \
    && rm -rf /var/lib/apt/lists/*

# The locked Cloudflare prerenderer binds and fetches "localhost". Docker can
# resolve the listener to ::1 while Node fetch connects to 127.0.0.1; keep
# both on IPv4 without changing the project's workerd prerendering behavior.
ENV CI=true \
    ASTRO_TELEMETRY_DISABLED=1 \
    WRANGLER_SEND_METRICS=false \
    NODE_OPTIONS=--dns-result-order=ipv4first

# The scanner supplies the public repository checkout as the build context.
COPY . /src
WORKDIR /src

# Keep dev dependencies, Wrangler/Miniflare/workerd, and generated font/static
# assets. Astro's configured Google font provider needs network at build time.
# No .dev.vars, Cloudflare login, remote D1, or runtime secrets are needed.
# Approve only the native-tool install scripts reported by the locked install.
# Use persistent npm config: project-scoped ci rejects --allow-scripts flags.
RUN npm config set allow-scripts=esbuild,sharp,workerd --location=user \
    && npm ci --include=dev --no-audit --no-fund \
    && npm run build

CMD ["/bin/bash"]
