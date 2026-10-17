FROM node:24-bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    COREPACK_ENABLE_DOWNLOAD_PROMPT=0

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 build-essential \
    && rm -rf /var/lib/apt/lists/*

# Match package.json's packageManager; every dependency is installed while online.
RUN npm install --global pnpm@10.33.1

# Scanner contract: source, built output, test fixtures and dependencies stay in /src.
WORKDIR /src
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . /src
RUN pnpm build && pnpm typecheck && pnpm validate-corpus

# Do not hide a failed smoke test. The same suite can be rerun with --network none.
RUN pnpm test

CMD ["bash"]
