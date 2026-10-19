# OSS Scanner build for rippled / xrpld (https://github.com/XRPLF/rippled): the reference C++ server for
# the XRP Ledger.
#
# This file lives next to project.yaml in the oss-scanner repository. The scanner copies it to
# .oss-scanner/Dockerfile in a fresh clone of rippled's develop branch and builds it with that clone as the
# build context, so `COPY . /src` puts the checkout inside the image. The network is available only while
# this file builds; the scan itself runs offline, so every dependency must be fetched here.
#
# Why Nix rather than apt: rippled is C++23 and needs a bleeding-edge toolchain (CI uses GCC 15 / Clang 23 and
# Conan 2.33, see nix/packages.nix). Those compilers are not packaged by any Debian/Ubuntu release, and
# rippled's own docs say "Linux always builds in a Nix image" (.github/workflows/reusable-build-test-config.yml).
# So instead of approximating the toolchain we build inside one of rippled's own Nix development shells.
#
# The default shell is used, not rippled's faster `.#gcc-plain`: rippled's prebuilt Conan packages (protoc, the
# gRPC plugin) are linked against the custom glibc the default shell builds, and fail to start without it.
# Realising that shell is the slow part of this build (about 19 minutes on one machine).
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

# Realise the dev shell once while the network is up. This pulls or builds the pinned toolchain (GCC 15 / Clang 23,
# CMake, Ninja, Conan 2.33, the Rust toolchain from rust-toolchain.toml) into the Nix store, so every later `nix develop` works offline. Its shellHook also runs
# rippled's Conan setup (./conan/init.sh) on entry.
RUN nix develop --command true

# Build, mirroring rippled CI (.github/actions/build-deps + reusable-build-test-config.yml):
#   - ./conan/init.sh installs the `ci` Conan profile, global.conf and the xrplf remote
#     (https://conan.xrplf.org/repository/conan/, which carries rippled's patched recipes). Idempotent.
#   - conan install uses --profile:all ci and the host options CI sets: xrpld=True (builds the server). We
#     leave tests off (tests=False): the unit suite is a large share of the compile steps, and the scanner build
#     has 45 minutes. We add assertions=True so the server's XRPL_ASSERT
#     invariant checks are compiled in -- an assert that a peer or transaction can trip is a finding worth
#     catching. build_type=Release keeps the build tractable.
#   - cmake configures with the Conan toolchain and adds -g1 -fno-omit-frame-pointer for debuggable stacks.
#   - cmake --build builds the default targets (the `xrpld` server).
# A step that fails fails the build, and primary_contact gets the log.
# XRPLD_NO_PATCH_NIX_BINARY=1 skips cmake/PatchNixBinary.cmake. That step retargets binaries to the system
# loader by running a `#!/bin/bash` script, and this image has no /bin/bash, so configure fails. The binaries
# keep their Nix-store loader and run inside `nix develop`, as described above.
RUN XRPLD_NO_PATCH_NIX_BINARY=1 nix develop --command bash -eux -o pipefail -c '\
      ./conan/init.sh && \
      mkdir -p .build && cd .build && \
      conan install .. \
        --output-folder . \
        --profile:all ci \
        --build missing \
        --options:host="&:tests=False" \
        --options:host="&:xrpld=True" \
        --options:host="&:assertions=True" \
        --settings:all build_type=Release \
        --conf:all tools.build:jobs="$(nproc)" && \
      cmake \
        -G Ninja \
        -DCMAKE_TOOLCHAIN_FILE:FILEPATH=build/generators/conan_toolchain.cmake \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_CXX_FLAGS="-g1 -fno-omit-frame-pointer" \
        -Dxrpld=ON -Dtests=OFF -Dassertions=ON \
        .. && \
      cmake --build . --parallel "$(nproc)"'

# Smoke test, so the log shows the image works: the server binary starts and reports its version.
RUN nix develop --command bash -c 'cd /src/.build && ./xrpld --version'

# The scanner adds its own layer on top of this image (Claude Code, analysis tools). The Nix base image has no
# apt/apk and lacks sed, awk and python3, which that layer's installer needs (it already has git, curl, grep and
# bash). It also has no system ELF loader, so the prebuilt `claude` binary cannot start; link the loader from
# Nix's glibc to the standard path. The layer also links claude into /usr/local/bin, which the base image lacks
# and does not have on PATH. Do all of this now while the network is up.
RUN set -e; \
    nix profile install nixpkgs#gnused nixpkgs#gawk nixpkgs#python3; \
    case "$(uname -m)" in \
      x86_64) loader=/lib64/ld-linux-x86-64.so.2 ;; \
      aarch64) loader=/lib/ld-linux-aarch64.so.1 ;; \
      *) echo "unsupported arch: $(uname -m)" >&2; exit 1 ;; \
    esac; \
    glibc="$(nix build --no-link --print-out-paths nixpkgs#glibc.out)"; \
    mkdir -p /usr/local/bin "$(dirname "$loader")"; \
    ln -sf "$glibc/lib/$(basename "$loader")" "$loader"
ENV PATH="/usr/local/bin:${PATH}"
