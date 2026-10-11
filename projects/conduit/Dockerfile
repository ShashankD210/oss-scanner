FROM oven/bun:1.3.5 AS bun
FROM node:24-bookworm-slim

COPY --from=bun /usr/local/bin/bun /usr/local/bin/bun
RUN ln -s /usr/local/bin/bun /usr/local/bin/bunx

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 build-essential \
    && rm -rf /var/lib/apt/lists/*

# The scanner expects the complete checkout, dependencies, and build at /src.
COPY . /src
WORKDIR /src
ENV HUSKY=0

# Match the repository's Bun version and preserve the committed lockfile.
RUN bun install --frozen-lockfile

# Fetch browser binaries and system dependencies before the offline audit.
RUN bunx --no-install playwright install --with-deps chromium webkit
RUN bun run build:preview
