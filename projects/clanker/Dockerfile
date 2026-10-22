FROM ghcr.io/astral-sh/uv:0.12.13 AS uv
FROM node:26.10.0-bookworm-slim AS node
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash build-essential ca-certificates curl git patch pkg-config python3 \
      python3-venv rsync shellcheck sqlite3 unzip xz-utils \
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
      https://github.com/oven-sh/bun/releases/download/bun-v1.4.2/bun-linux-x64.zip -o /opt/bun.zip \
    && unzip -q /opt/bun.zip -d /opt \
    && mv /opt/bun-linux-x64 /opt/bun \
    && ln -s bun /opt/bun/bunx \
    && rm /opt/bun.zip \
    && uv venv --python /usr/bin/python3 /opt/lint \
    && uv pip install --python /opt/lint/bin/python ruff==0.16.4 yamllint==1.38.0
ENV PATH="/opt/zig:/opt/bun:/opt/lint/bin:${PATH}" \
    NO_COLOR=1 LC_ALL=C TZ=UTC SOURCE_DATE_EPOCH=1704067200
COPY . /src
WORKDIR /src
RUN bun install --frozen-lockfile \
    && (cd tools/ts && bun install --frozen-lockfile) \
    && zig build --fetch=all \
    && scripts/apply-patches.sh
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
RUN --network=none zig build -j2 \
    && zig build tools -j2 \
    && zig build test -j2 \
    && zig build fmt -j2
RUN --network=none bun test tools/oxlint \
    && bun run lint \
    && bun run typecheck \
    && bun scripts/brand.ts --check \
    && tools/ts/verify.sh \
    && ui/app/verify-css.sh \
    && python3 -B -m unittest scripts.test_sbom scripts.test_backup_state scripts.test_verify_backup scripts.test_install_state_backup scripts.test_release_checksum scripts.test_check_threat_model \
    && python3 scripts/check-threat-model.py \
    && git ls-files -z '*.py' | xargs -0 ruff check
RUN --network=none zig-out/bin/clanker gate \
    && zig build e2e -j2
