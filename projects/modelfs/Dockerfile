FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash binutils build-essential ca-certificates curl fuse3 git libfuse3-dev \
      pkg-config python3 python3-venv shellcheck xz-utils zstd \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

RUN set -eux; \
    version="$(sed -n 's/^[[:space:]]*\.minimum_zig_version *= *"\([^"]*\)".*/\1/p' build.zig.zon)"; \
    digest="$(sed -n 's/^ZIG_TARBALL_SHA256="\([^"]*\)"/\1/p' scripts/lib.sh)"; \
    test -n "$version" && test -n "$digest"; \
    curl --fail --location --retry 3 "https://ziglang.org/download/$version/zig-x86_64-linux-$version.tar.xz" -o /opt/zig.tar.xz; \
    echo "$digest  /opt/zig.tar.xz" | sha256sum --check; \
    mkdir /opt/zig; \
    tar -xJf /opt/zig.tar.xz -C /opt/zig --strip-components=1; \
    rm /opt/zig.tar.xz

ENV PATH="/opt/zig:/opt/uv/bin:/src/.venv/bin:${PATH}"
RUN python3 -m venv /opt/uv \
    && /opt/uv/bin/pip install --no-cache-dir uv==0.12.13 \
    && uv sync --frozen --python /usr/bin/python3

# Permission-denial tests must run without root's DAC override.
RUN useradd --create-home --uid 10001 --shell /bin/bash modelfs-audit \
    && chown -R modelfs-audit:modelfs-audit /src
USER modelfs-audit
ENV HOME=/home/modelfs-audit
RUN zig build -Doptimize=Debug && ./scripts/check.sh

USER root
ENV HOME=/root
RUN git config --system --add safe.directory /src
