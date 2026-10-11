FROM node:24-bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive \
    CI=true
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates git python3 \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
RUN npm ci --no-audit --no-fund \
    && npm run check:content \
    && npm test \
    && npm run check:images \
    && npm run build
