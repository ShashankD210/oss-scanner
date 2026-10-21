# syntax=docker/dockerfile:1
# Public CPU builder used by Milvus master in .env and docker-compose.yml.
FROM milvusdb/milvus-env:ubuntu22.04-20261009-bcd6ade

USER root
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ARG BUILD_JOBS=8
ENV DEBIAN_FRONTEND=noninteractive \
    CONAN_HOME=/root/.conan2 \
    CONAN_CPU_COUNT=${BUILD_JOBS} \
    CMAKE_BUILD_PARALLEL_LEVEL=${BUILD_JOBS} \
    CARGO_BUILD_JOBS=${BUILD_JOBS} \
    jobs=${BUILD_JOBS}

# Keep source, compilers, native dependencies and package caches in the image.
COPY . /src
WORKDIR /src

# Match the Conan version required by master, without modifying system Python.
RUN uv tool install --python python3.12 conan==2.25.1 --force \
    && uv venv -p 3.12 /src/.venv \
    && source /src/.venv/bin/activate \
    && uv pip install pip setuptools==80.9.0 wheel==0.45.1
ENV PATH=/src/.venv/bin:/root/.cargo/bin:/usr/local/bin:/go/bin:/usr/local/go/bin:$PATH

# Download modules for the server, shared packages and Go client while online.
RUN go mod download \
    && (cd pkg && go mod download) \
    && (cd client && go mod download)

# The normal build fetches protobuf sources, Conan packages and Rust crates.
# Use the standard Release dependency packages, keeping debug information in
# Milvus itself rather than rebuilding every dependency in a different profile.
ENV CFLAGS="-g1 -fno-omit-frame-pointer" \
    CXXFLAGS="-g1 -fno-omit-frame-pointer"
# Conan 2 reads tools.build:jobs; retain full logs and print bounded progress.
RUN set -e; \
    mkdir -p "$CONAN_HOME"; \
    printf 'tools.build:jobs=%s\n' "$BUILD_JOBS" >> "$CONAN_HOME/global.conf"; \
    (make build-cpp-with-unittest mode=Release \
        && make build-go build-pyudf-runtime-wheel PYTHON=/src/.venv/bin/python) \
        > /tmp/milvus-build.log 2>&1 & \
    build_pid=$!; \
    while kill -0 "$build_pid" 2>/dev/null; do \
        tail -n 5 /tmp/milvus-build.log 2>/dev/null || true; \
        sleep 30; \
    done; \
    if wait "$build_pid"; then \
        tail -n 20 /tmp/milvus-build.log; \
    else \
        tail -n 200 /tmp/milvus-build.log; \
        exit 1; \
    fi

ENV LD_LIBRARY_PATH=/src/internal/core/output/lib:/src/internal/core/output/lib64:/src/cmake_build/lib \
    PKG_CONFIG_PATH=/src/internal/core/output/lib/pkgconfig:/src/internal/core/output/lib64/pkgconfig

# Compile focused tests while downloads are still allowed. Fail on build errors.
RUN source scripts/setenv.sh && set -e \
    && go test -tags dynamic,test -gcflags="all=-N -l" -run '^$' ./internal/parser/planparserv2 \
    && (cd pkg && go test -tags dynamic,test -gcflags="all=-N -l" -run '^$' ./util/crypto)

# Tests run offline during the audit. Do not invoke make's download targets there.
ENV GOPROXY=off GOSUMDB=off CARGO_NET_OFFLINE=true UV_OFFLINE=1
RUN --network=none source scripts/setenv.sh && set -e \
    && go test -tags dynamic,test -gcflags="all=-N -l" -count=1 ./internal/parser/planparserv2 \
    && (cd pkg && go test -tags dynamic,test -gcflags="all=-N -l" -count=1 ./util/crypto) \
    && internal/core/output/unittest/all_tests --gtest_filter='StringExpr.Term:StringExpr.Compare'

ENTRYPOINT []
CMD ["bash"]
