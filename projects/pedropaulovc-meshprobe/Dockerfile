FROM ghcr.io/astral-sh/uv:latest AS uv
FROM python:3.12-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    UV_PYTHON_DOWNLOADS=never \
    UV_LINK_MODE=copy \
    GALLIUM_DRIVER=llvmpipe \
    LIBGL_ALWAYS_SOFTWARE=1 \
    PATH=/src/.venv/bin:/opt/blender-5.2.0-linux-x64:$PATH

COPY --from=uv /uv /uvx /usr/local/bin/
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl xz-utils build-essential git bubblewrap \
      libegl1 libgl1 libgl1-mesa-dri libglu1-mesa libsm6 libice6 \
      libx11-6 libxext6 libxi6 libxrender1 libxfixes3 libxrandr2 libxxf86vm1 \
      libxkbcommon0 libwayland-client0 libwayland-cursor0 libwayland-egl1 \
      libdbus-1-3 libasound2 \
    && rm -rf /var/lib/apt/lists/*

# The repository pins 5.2.0 LTS. Do not substitute a distro or older Blender.
# SHA-256 is from the official blender-5.2.0.sha256 release manifest.
RUN test "$(dpkg --print-architecture)" = amd64 \
    && curl --fail --location \
      https://download.blender.org/release/Blender5.2/blender-5.2.0-linux-x64.tar.xz \
      --output /tmp/blender-5.2.0-linux-x64.tar.xz \
    && printf '%s  %s\n' \
      96f6c181a30f4950607839dc84d42a354b250d8a0231b098b59b7bc69c351c48 \
      /tmp/blender-5.2.0-linux-x64.tar.xz | sha256sum --check --strict \
    && tar -xJf /tmp/blender-5.2.0-linux-x64.tar.xz -C /opt \
    && rm /tmp/blender-5.2.0-linux-x64.tar.xz

COPY . /src
WORKDIR /src
# Keep the locked development environment and build cache for offline source work.
# No qualification corpora, hosted agents, or account credentials are fetched.
RUN uv sync --locked --all-groups --python /usr/local/bin/python3.12 --compile-bytecode \
    && uv build --wheel --out-dir /src/dist \
    && .venv/bin/python -m compileall -q src tools tests
ENV UV_OFFLINE=1
CMD ["bash"]
