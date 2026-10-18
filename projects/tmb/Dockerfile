# Builds the TMB plugin the way its CI does (Node 22 + bun), then runs the
# unit layer once with the network up so the embedding model is cached in the
# image before the scanner cuts network access.
FROM node:22-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    TMB_DISABLE_REMOTE_SYNC=1 \
    TMB_SKIP_AUTO_PRESCAN=1
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git jq sqlite3 shellcheck perl python3 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=oven/bun:1 /usr/local/bin/bun /usr/local/bin/bun

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Build as the maintainers build: workspace install, tsc + esbuild bundle of
# the MCP server into mcp/trajectory-server/dist/.
RUN bun install --frozen-lockfile && bun run build

# Unit tests of the MCP server (also warms ~/.cache/huggingface with the
# bge-small embedding model the server loads at runtime). Failures are logged,
# not fatal. The full suite is `bash tests/run-all.sh` (see threat_model.md).
RUN cd mcp/trajectory-server \
    && (node --experimental-sqlite --test dist/test/*.test.js || echo "WARNING: unit tests reported failures")
