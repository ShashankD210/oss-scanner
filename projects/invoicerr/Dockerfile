FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 postgresql redis-server \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

WORKDIR /src/backend
RUN npm ci \
    && DATABASE_URL=not_required npx prisma generate --schema=prisma/schema.prisma \
    && npm run build

WORKDIR /src/frontend
RUN npm ci && npm run build

WORKDIR /src/backend
RUN npm test || echo "WARNING: backend tests reported failures"

WORKDIR /src