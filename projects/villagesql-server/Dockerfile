# OSS Scanner build for VillageSQL Server (a MySQL 8.4 LTS fork with the VillageSQL Extension Framework).
#
#   /src     the checkout
#   /build   Debug build with AddressSanitizer + UndefinedBehaviorSanitizer, the same configuration as
#            .github/workflows/sanitizer.yml. Binaries in /build/bin, bundled .veb files in /build/lib/veb.
#            Rebuild after a patch: cmake --build /build --target mysqld
#            Unit tests are not built (WITH_UNIT_TESTS=OFF) so the build fits the scanner's time limit.
#   /data    an initialized data directory (root@localhost, no password)
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive

# setup_linux_build_env.sh is the project's definitive dependency list; CI installs from it too.
COPY villagesql/bld_tools/setup_linux_build_env.sh /tmp/
RUN /tmp/setup_linux_build_env.sh \
    && apt-get install -y --no-install-recommends ninja-build \
    && rm -rf /var/lib/apt/lists/* /tmp/setup_linux_build_env.sh

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# The server refuses in-source builds, so the build tree is /build.
RUN cmake -S /src -B /build -G Ninja \
      -DCMAKE_BUILD_TYPE=Debug -DWITH_DEBUG=1 \
      -DWITH_ASAN=1 -DWITH_UBSAN=1 \
      -DWITH_SSL=system -DWITH_ROUTER=OFF -DWITH_NUMA=OFF -DWITH_UNIT_TESTS=OFF \
      -DWITH_NDB=OFF -DWITH_NDBCLUSTER_STORAGE_ENGINE=OFF \
    && cmake --build /build -j"$(nproc)"

# Leak reports at shutdown are noise for this purpose; MTR sets its own *SAN_OPTIONS for the servers it starts.
ENV ASAN_OPTIONS=detect_leaks=0:abort_on_error=1:symbolize=1 \
    UBSAN_OPTIONS=print_stacktrace=1

RUN /build/bin/mysqld --no-defaults --user=root --initialize-insecure --basedir=/build --datadir=/data

# Run the cheap tests so you know the image works, but do not let a failing test abort the build.
RUN cd /build/mysql-test \
    && (timeout 300 ./mysql-test-run.pl --parallel=auto --force --suite=villagesql/select,villagesql/insert \
        || echo "WARNING: MTR smoke test reported failures") \
    && rm -rf /build/mysql-test/var
