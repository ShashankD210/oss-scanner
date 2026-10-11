FROM oven/bun:1.4.2-debian
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential python3 \
    && rm -rf /var/lib/apt/lists/*

# Scanner contract: the target repository, not this registry, is the context.
COPY . /src
WORKDIR /src

# bun.lock pins the type packages and TypeScript, including its native binary.
RUN bun install --frozen-lockfile
# Bun embeds its SQLite runtime; keep source, tests and debug sourcemaps.
RUN mkdir -p /src/dist \
    && bun build --compile --sourcemap ./src/cli.ts --outfile /src/dist/llm-quota

# No credential discovery, provider requests, token refresh or tests during build.
CMD ["/bin/bash"]
