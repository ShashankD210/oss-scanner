FROM node:22-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CI=1

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates git build-essential python3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
COPY . /src

RUN npm ci --no-audit --no-fund
RUN npm install --global --no-audit --no-fund tsx@4.23.15
RUN tsx scripts/build-api.mjs
