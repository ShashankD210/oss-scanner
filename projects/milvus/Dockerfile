# syntax=docker/dockerfile:1
# Public CPU builder used by Milvus master in .env and docker-compose.yml.
FROM milvusdb/milvus-env:ubuntu22.04-20261009-bcd6ade AS setup

USER root
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ARG BUILD_JOBS=8
ENV DEBIAN_FRONTEND=noninteractive \
    CONAN_HOME=/root/.conan2 \
    CONAN_CPU_COUNT=${BUILD_JOBS} \
    CMAKE_BUILD_PARALLEL_LEVEL=${BUILD_JOBS} \
    CARGO_BUILD_JOBS=${BUILD_JOBS} \
    jobs=${BUILD_JOBS}

# Match the Conan version required by master, without modifying system Python.
RUN uv tool install --python python3.12 conan==2.25.1 --force \
    && uv venv -p 3.12 /src/.venv \
    && source /src/.venv/bin/activate \
    && uv pip install pip setuptools==80.9.0 wheel==0.45.1
ENV PATH=/src/.venv/bin:/root/.cargo/bin:/usr/local/bin:/go/bin:/usr/local/go/bin:$PATH

# Conan 2 reads tools.build:jobs rather than CONAN_CPU_COUNT.
RUN mkdir -p "$CONAN_HOME" \
    && printf 'tools.build:jobs=%s\n' "$BUILD_JOBS" >> "$CONAN_HOME/global.conf"

# Keep full logs while printing bounded progress below BuildKit's output limit.
COPY --chmod=0755 <<'EOF' /usr/local/bin/milvus-build-step
#!/bin/bash
set -uo pipefail
logfile=$1
shift
printf 'Build step started; full log: %s\n' "$logfile"
"$@" >"$logfile" 2>&1 &
build_pid=$!
while kill -0 "$build_pid" 2>/dev/null; do
    tail -n 5 "$logfile" 2>/dev/null || true
    sleep 30
done
wait "$build_pid"
result=$?
if [ "$result" -eq 0 ]; then
    tail -n 20 "$logfile"
else
    tail -n 200 "$logfile"
fi
exit "$result"
EOF

# Keep the checkout and build caches. The injected scanner recipe is metadata;
# excluding it lets recipe-only fixes reuse completed source build stages.
COPY --exclude=.oss-scanner . /src
WORKDIR /src

# Download modules for the server, shared packages and Go client while online.
RUN go mod download \
    && (cd pkg && go mod download) \
    && (cd client && go mod download)

FROM setup AS dependencies

# Use standard Release dependency packages and retain Milvus debug information.
ENV CFLAGS="-g1 -fno-omit-frame-pointer" \
    CXXFLAGS="-g1 -fno-omit-frame-pointer"
RUN milvus-build-step /tmp/milvus-dependencies.log \
    make generated-proto plan-parser-lib mode=Release

FROM dependencies AS native

# Keep compiler selection stable when the scanner installs additional tooling.
ENV CC=/usr/bin/gcc-12 CXX=/usr/bin/g++-12

# The prerequisites were built in the previous stage; avoid fetching them again.
RUN milvus-build-step /tmp/milvus-native-build.log \
    make -o generated-proto -o plan-parser-lib build-cpp-with-unittest mode=Release

ENV LD_LIBRARY_PATH=/src/internal/core/output/lib:/src/internal/core/output/lib64:/src/cmake_build/lib \
    PKG_CONFIG_PATH=/src/internal/core/output/lib/pkgconfig:/src/internal/core/output/lib64/pkgconfig

FROM native AS server

RUN milvus-build-step /tmp/milvus-server-build.log \
    make build-go build-pyudf-runtime-wheel PYTHON=/src/.venv/bin/python
RUN source /src/.venv/bin/activate \
    && uv pip install cmake_build/runtime/pyudf/wheels/*.whl

FROM server AS final

# Compile focused tests while downloads are still allowed. Fail on build errors.
RUN source scripts/setenv.sh && set -e \
    && go test -tags dynamic,test -gcflags="all=-N -l" -run '^$' ./internal/parser/planparserv2 \
    && (cd pkg && go test -tags dynamic,test -gcflags="all=-N -l" -run '^$' ./util/crypto)

# Invoke the verified toolchain directly; auto-selection needs sumdb offline.
RUN ln -s "$(go env GOROOT)" /opt/milvus-go
ENV GOROOT=/opt/milvus-go PATH=/opt/milvus-go/bin:$PATH GOTOOLCHAIN=local

# Tests run offline during the audit. Do not invoke make's download targets there.
ENV GOPROXY=off GOSUMDB=off CARGO_NET_OFFLINE=true UV_OFFLINE=1
RUN --network=none source scripts/setenv.sh && source /src/.venv/bin/activate && set -e \
    && ./bin/milvus dry-run \
    && PYUDF_TEST_INSTALLED_RUNTIME=1 python \
        internal/util/function/pyudf/python/tests/test_installed_runtime.py -v \
    && go test -tags dynamic,test -gcflags="all=-N -l" -count=1 ./internal/parser/planparserv2 \
    && (cd pkg && go test -tags dynamic,test -gcflags="all=-N -l" -count=1 ./util/crypto) \
    && internal/core/output/unittest/all_tests --gtest_filter='StringExpr.Term:StringExpr.Compare'

ENTRYPOINT []
CMD ["bash"]
