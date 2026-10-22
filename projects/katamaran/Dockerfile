FROM golang:1.26.9-bookworm AS go
FROM node:26.10.0-bookworm-slim AS node
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash build-essential ca-certificates curl git make python3 python3-venv shellcheck \
    && rm -rf /var/lib/apt/lists/*
COPY --from=go /usr/local/go /opt/go
COPY --from=node /usr/local /usr/local
ENV PATH="/opt/go/bin:/opt/python/bin:/opt/biome/node_modules/.bin:${PATH}" \
    CGO_ENABLED=1 GOTOOLCHAIN=local GOMAXPROCS=2 GOFLAGS=-p=2
COPY . /src
WORKDIR /src
RUN set -eux; \
    test "$(node --version)" = "$(cat .node-version)"; \
    version="$(python3 -c 'import json; print(json.load(open("biome.json"))["$schema"].split("/")[-2])')"; \
    npm install --prefix /opt/biome --no-audit --no-fund "@biomejs/biome@$version"; \
    python3 -m venv /opt/python; \
    pip install --no-cache-dir yamllint==1.38.0; \
    go mod download
RUN --network=none make check
