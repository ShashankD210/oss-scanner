FROM ghcr.io/astral-sh/uv:0.12.13 AS uv
FROM node:26.10.0-bookworm-slim AS node
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash build-essential ca-certificates curl git libvncserver-dev pkg-config \
      python3 python3-venv qemu-system-x86 qemu-utils shellcheck unzip xz-utils \
    && rm -rf /var/lib/apt/lists/*
COPY --from=uv /uv /usr/local/bin/uv
COPY --from=node /usr/local /usr/local
RUN curl --fail --location --retry 3 \
      https://ziglang.org/download/0.16.0/zig-x86_64-linux-0.16.0.tar.xz -o /opt/zig.tar.xz \
    && echo '70e49664a74374b48b51e6f3fdfbf437f6395d42509050588bd49abe52ba3d00  /opt/zig.tar.xz' | sha256sum --check \
    && mkdir /opt/zig \
    && tar -xJf /opt/zig.tar.xz -C /opt/zig --strip-components=1 \
    && rm /opt/zig.tar.xz
RUN curl --fail --location --retry 3 \
      https://github.com/oven-sh/bun/releases/download/bun-v1.4.0/bun-linux-x64.zip -o /opt/bun.zip \
    && unzip -q /opt/bun.zip -d /opt \
    && mv /opt/bun-linux-x64 /opt/bun \
    && ln -s bun /opt/bun/bunx \
    && rm /opt/bun.zip \
    && uv venv --python /usr/bin/python3 /opt/lint \
    && uv pip install --python /opt/lint/bin/python yamllint==1.38.0
ENV PATH="/opt/zig:/opt/bun:/opt/lint/bin:${PATH}"
COPY . /src
WORKDIR /src
RUN bun install --frozen-lockfile && zig build --fetch=all
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
RUN --network=none zig build check -j2 \
    && bun run lint \
    && bun run typecheck \
    && bun run test
