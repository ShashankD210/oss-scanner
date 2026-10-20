FROM oven/bun:1.3.5 AS bun
FROM node:22-bookworm

COPY --from=bun /usr/local/bin/bun /usr/local/bin/bun
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      ca-certificates git python3 build-essential redis-server postgresql ripgrep \
    && rm -rf /var/lib/apt/lists/*

# Keep the entire checkout and development dependencies for offline investigation.
COPY . /src
WORKDIR /src
ENV PLAYWRIGHT_BROWSERS_PATH=/opt/playwright
RUN bun install --frozen-lockfile
RUN cd apps/web && bun x playwright install --with-deps chromium

# Build the API, dashboard, and desktop main process without pruning source/tests.
# The marketing/docs site and signed native release installers are not required
# to exercise the API, authentication, persistence, or MCP security boundaries.
RUN bun run --cwd apps/api build \
    && bun run --cwd apps/web build \
    && bun run --cwd apps/desktop build:main

CMD ["bash"]
