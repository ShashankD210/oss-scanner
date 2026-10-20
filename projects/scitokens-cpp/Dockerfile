# OSS Scanner build for scitokens-cpp (https://github.com/scitokens/scitokens-cpp).
#
# This file lives next to project.yaml in the oss-scanner repository. The scanner places it in a fresh clone of
# scitokens-cpp's master branch and builds it with that clone as the context.
#
# Dependencies mirror the upstream GitHub Actions workflow (.github/workflows/ccpp.yml). jwt-cpp is vendored in the
# repository; GoogleTest comes from the distribution (SCITOKENS_EXTERNAL_GTEST=ON), so no submodules are needed.
#
# Two builds are produced from the same checkout in /src:
#   /src/build       RelWithDebInfo, -g -fno-omit-frame-pointer. Run the test suite with `ctest` in this directory.
#                    Tools: /src/build/{scitokens-verify,scitokens-create,scitokens-list-access,...}
#   /src/build-asan  Debug build with AddressSanitizer (SCITOKENS_WITH_ASAN=ON). Use it to confirm memory-safety
#                    findings. Same tools and tests as above.
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential cmake ninja-build pkg-config gdb \
      libssl-dev openssl sqlite3 libsqlite3-dev libcurl4-openssl-dev uuid-dev libgtest-dev \
      python3 \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# 1) Reference build with debug info and frame pointers.
RUN cmake -S /src -B /src/build -G Ninja \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_C_FLAGS="-g -fno-omit-frame-pointer" \
      -DCMAKE_CXX_FLAGS="-g -fno-omit-frame-pointer" \
      -DSCITOKENS_BUILD_UNITTESTS=ON \
      -DSCITOKENS_EXTERNAL_GTEST=ON \
    && cmake --build /src/build -j"$(nproc)"

# 2) AddressSanitizer build.
RUN cmake -S /src -B /src/build-asan -G Ninja \
      -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_C_FLAGS="-g -fno-omit-frame-pointer" \
      -DCMAKE_CXX_FLAGS="-g -fno-omit-frame-pointer" \
      -DSCITOKENS_BUILD_UNITTESTS=ON \
      -DSCITOKENS_EXTERNAL_GTEST=ON \
      -DSCITOKENS_WITH_ASAN=ON \
    && cmake --build /src/build-asan -j"$(nproc)"

ENV ASAN_OPTIONS=abort_on_error=1:symbolize=1

# Run the tests so you know the image works, but do not let a failing test abort the build. The integration tests
# start a local JWKS server (test/jwks_server.py) and need no Internet access.
RUN cd /src/build && (ctest --output-on-failure --timeout 300 || echo "WARNING: ctest reported failures")
