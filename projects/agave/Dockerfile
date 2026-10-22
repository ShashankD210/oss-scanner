FROM node:26.10.0-bookworm-slim AS node
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash build-essential ca-certificates curl git openjdk-21-jre-headless \
      python3 python3-venv shellcheck unzip xz-utils \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
# The scanner removes this context filter; the packaging gate checks its source.
RUN git show HEAD:.dockerignore > .dockerignore
RUN set -eux; \
    version="$(tr -d '[:space:]' < .zigversion)"; \
    digest="$(sed -n 's/^ARG ZIG_SHA256_X86_64=//p' Dockerfile)"; \
    test -n "$version" && test -n "$digest"; \
    curl --fail --location --retry 3 "https://ziglang.org/download/$version/zig-x86_64-linux-$version.tar.xz" -o /opt/zig.tar.xz; \
    echo "$digest  /opt/zig.tar.xz" | sha256sum --check; \
    mkdir /opt/zig; \
    tar -xJf /opt/zig.tar.xz -C /opt/zig --strip-components=1; \
    rm /opt/zig.tar.xz
ENV PATH="/opt/zig:/opt/bun:/opt/python/bin:${PATH}"
RUN set -eux; \
    version="$(python3 -c 'import json; print(json.load(open("package.json"))["packageManager"].split("@")[1])')"; \
    curl --fail --location --retry 3 "https://github.com/oven-sh/bun/releases/download/bun-v$version/bun-linux-x64.zip" -o /opt/bun.zip; \
    unzip -q /opt/bun.zip -d /opt; \
    mv /opt/bun-linux-x64 /opt/bun; \
    rm /opt/bun.zip; \
    bun install --frozen-lockfile
RUN python3 -m venv /opt/python \
    && pip install --no-cache-dir uv==0.12.13 ruff==0.16.4 \
    && uv lock --check --project tests \
    && uv lock --check --project research/kernels
COPY --from=node /usr/local /usr/local
RUN curl --fail --location --retry 3 \
      https://github.com/koalaman/shellcheck/releases/download/v0.11.0/shellcheck-v0.11.0.linux.x86_64.tar.xz \
      -o /opt/shellcheck.tar.xz \
    && echo "8c3be12b05d5c177a04c29e3c78ce89ac86f1595681cab149b65b97c4e227198  /opt/shellcheck.tar.xz" | sha256sum --check \
    && tar -xJf /opt/shellcheck.tar.xz -C /opt \
    && cp /opt/shellcheck-v0.11.0/shellcheck /usr/local/bin/shellcheck \
    && rm /opt/shellcheck.tar.xz
RUN --network=none zig build -j2
RUN --network=none zig build ci -j2
