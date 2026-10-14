# OSS Scanner build for rippled / xrpld (https://github.com/XRPLF/rippled): the reference C++ server for
# the XRP Ledger.
#
# This file lives next to project.yaml in the oss-scanner repository. The scanner copies it to
# .oss-scanner/Dockerfile in a fresh clone of rippled's develop branch and builds it with that clone as the
# build context, so `COPY . /src` puts the checkout inside the image. The network is available only while
# this file builds; the scan itself runs offline, so every dependency must be fetched here.
#
# Why Nix rather than apt: rippled is C++23 and pins a bleeding-edge toolchain (CI uses GCC 15 / Clang 23 and
# Conan 2.33, see nix/packages.nix). Those compilers are not packaged by any Debian/Ubuntu release, and
# rippled's own docs say "Linux always builds in a Nix image" (.github/workflows/reusable-build-test-config.yml).
# So instead of approximating the toolchain we build inside rippled's own pinned Nix development shell -- the
# exact environment its CI uses. This makes the compiler, CMake, Ninja, Conan and Rust toolchain match upstream
# and avoids a whole class of "builds in CI, not here" failures.
#
# Consequence for the scan: binaries built in the dev shell link against glibc/libstdc++ in the Nix store, so
# they must be run from inside the shell. Prefix every run command with `nix develop --command`, run from /src
# (threat_model.md spells this out). The dev shell is realised once below while the network is up, so later
# `nix develop` invocations are offline cache hits.
FROM nixos/nix:latest

# rippled's flake needs flakes and the unified nix CLI, both still gated behind experimental-features.
RUN mkdir -p /etc/nix \
    && printf 'experimental-features = nix-command flakes\nmax-jobs = auto\n' >> /etc/nix/nix.conf

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Realise the dev shell once while the network is up. This pulls the pinned toolchain (GCC 15 / Clang 23,
# CMake, Ninja, Conan 2.33, the Rust toolchain from rust-toolchain.toml) into the Nix store -- mostly binary
# cache hits from cache.nixos.org -- so every later `nix develop` works offline. Its shellHook also runs
# rippled's Conan setup (./conan/init.sh) on entry.
RUN nix develop --command true

# Build, mirroring rippled CI (.github/actions/build-deps + reusable-build-test-config.yml):
#   - ./conan/init.sh installs the `ci` Conan profile, global.conf and the xrplf remote
#     (https://conan.xrplf.org/repository/conan/, which carries rippled's patched recipes). Idempotent.
#   - conan install uses --profile:all ci and the host options CI sets: tests=True (pulls in gtest and builds
#     the unit suite) and xrpld=True (builds the server). We add assertions=True so the server's XRPL_ASSERT
#     invariant checks are compiled in -- an assert that a peer or transaction can trip is a finding worth
#     catching. build_type=Release keeps the build and the unit tests tractable.
#   - cmake configures with the Conan toolchain and adds -g -fno-omit-frame-pointer for debuggable stacks.
#   - cmake --build builds the default targets (the `xrpld` server and the unit-test binaries).
# A step that fails fails the build, and primary_contact gets the log.
RUN nix develop --command bash -eux -o pipefail -c '\
      ./conan/init.sh && \
      mkdir -p .build && cd .build && \
      conan install .. \
        --output-folder . \
        --profile:all ci \
        --build missing \
        --options:host="&:tests=True" \
        --options:host="&:xrpld=True" \
        --options:host="&:assertions=True" \
        --settings:all build_type=Release \
        --conf:all tools.build:jobs="$(nproc)" && \
      cmake \
        -G Ninja \
        -DCMAKE_TOOLCHAIN_FILE:FILEPATH=build/generators/conan_toolchain.cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_CXX_FLAGS="-g -fno-omit-frame-pointer" \
        -Dxrpld=ON -Dtests=ON -Dassertions=ON \
        .. && \
      cmake --build . --parallel "$(nproc)"'

# Smoke test, so the log shows the image works. rippled registers its unit suites by name and
# `--unittest=<prefix>` filters them; here we run the `ripple.app` group (transactor, ledger and app-layer
# suites) under a time cap. Failures are printed but never abort the build -- the scanner wants the built tree
# even if a suite is slow or flaky. The full suite (`./xrpld --unittest`) is left for the scan; threat_model.md
# says how to drive it and the rest of the binary.
RUN nix develop --command bash -c '\
      cd /src/.build && \
      (timeout 20m ./xrpld --unittest=ripple.app --unittest-jobs "$(nproc)" \
        || echo "WARNING: xrpld --unittest reported failures or ran out of time")'
