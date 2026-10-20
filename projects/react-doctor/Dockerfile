FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    TURBO_TELEMETRY_DISABLED=1 \
    DO_NOT_TRACK=1
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 build-essential \
    && rm -rf /var/lib/apt/lists/*

# The pnpm version pinned by `packageManager` in package.json.
RUN npm install -g pnpm@10.29.1

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Same steps as CI (.github/workflows/ci.yml): install, then build every package under packages/.
RUN pnpm install --frozen-lockfile
RUN pnpm build

# Unit tests (vitest through turbo) for the CLI, core engine, API, both lint plugins, the fuzz harness and the evals.
# Some evals and public-repo tests reach the network and are skipped or fail offline, so failures do not stop the build.
RUN pnpm test --concurrency=1 || echo "WARNING: tests reported failures"
