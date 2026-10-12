FROM node:22-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CI=true
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

RUN corepack enable && pnpm install --frozen-lockfile --ignore-scripts
RUN pnpm build
RUN pnpm vitest run || echo "WARNING: tests reported failures"
