# syntax=docker/dockerfile:1
# Scanner image for https://github.com/hoprnet/hoprd (main)
#
# hoprd is the node daemon and REST API for the HOPR privacy-preserving mixnet.
# The audit runs offline, so this build fetches every crate in Cargo.lock, builds the workspace binaries and
# libraries in the dev profile (debug assertions and symbols on), builds the release feature set (Curvy PIX pool)
# beside them, precompiles every test binary, and runs the unit tests as a smoke check. It also ships a local chain
# (anvil, the HOPR contracts and bloklid), so hoprd nodes, hoprd-localcluster and the PIX cluster tests run without
# network access. threat_model.md, next to this file, says how to run all of it.

# The chain image that upstream's localcluster/docker-compose.yml pins: anvil, the HOPR and Curvy contract deployer,
# and bloklid, built with nix (x86-64 only).
FROM europe-west3-docker.pkg.dev/hoprassociation/docker-images/bloklid-anvil@sha256:e07eb3ac522d69e02fe599a352b26df5dd6189d37a28406b245ca435041ff185 AS chain

FROM rust:1-trixie
ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_TERM_COLOR=never \
    RUST_BACKTRACE=1

# Native build dependencies: a C toolchain for the -sys crates (aws-lc-sys, tikv-jemalloc-sys, zstd-sys), pkg-config, git
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential pkg-config \
    && rm -rf /var/lib/apt/lists/*

# cargo-nextest, recent enough for the commands documented in upstream's localcluster tests (`-j 1 --no-capture`)
ARG NEXTEST_VERSION=0.9.148
RUN case "$(uname -m)" in x86_64) platform=linux ;; aarch64) platform=linux-arm ;; *) exit 1 ;; esac \
    && curl -fsSL "https://get.nexte.st/${NEXTEST_VERSION}/${platform}" | tar -xzf - -C "${CARGO_HOME:-/usr/local/cargo}/bin" \
    && cargo nextest --version

# The local chain: the chain image's nix store, its entry points on PATH, and `hopr-chain-up` to start it
COPY --from=chain /nix /nix
COPY --from=chain /bin/anvil /bin/bloklid /bin/blokli-contract-deployer /bin/blokli-anvil-entrypoint /usr/local/bin/
COPY --chmod=755 <<'EOF' /usr/local/bin/hopr-chain-up
#!/usr/bin/env bash
# Start a fresh local HOPR chain (anvil, HOPR and Curvy contracts, bloklid) in the background, replacing any chain
# this script started before, and wait until Blokli answers at $HOPRD_CHAIN_URL. The log is /tmp/hopr-chain/chain.log.
set -euo pipefail
dir=/tmp/hopr-chain
# The chain runs in its own process group: the entrypoint execs into bloklid, and anvil (nix names it
# .anvil-wrapped) is its child. Stopping the whole group frees both ports before the next chain binds them.
if [ -f "${dir}/pgid" ]; then
  pgid="$(cat "${dir}/pgid")"
  kill -TERM -- "-${pgid}" 2>/dev/null || true
  for _ in $(seq 1 50); do
    kill -0 -- "-${pgid}" 2>/dev/null || break
    sleep 0.2
  done
  kill -KILL -- "-${pgid}" 2>/dev/null || true
fi
rm -rf "${dir}" && mkdir -p "${dir}"
# setsid makes the chain a new session and process group leader, which records its group id and keeps running after
# the shell that started it exits
BLOKLI_DATA_DIRECTORY="${dir}/data" BLOKLI_CONFIG_PATH="${dir}/config.toml" \
  setsid nohup bash -c 'echo "$$" >"$1" && exec blokli-anvil-entrypoint' bash "${dir}/pgid" \
  >"${dir}/chain.log" 2>&1 </dev/null &
for _ in $(seq 1 300); do
  if curl -sf "${HOPRD_CHAIN_URL}/readyz" >/dev/null; then
    echo "chain ready at ${HOPRD_CHAIN_URL}"
    exit 0
  fi
  sleep 1
done
echo "the chain did not become ready in 300 s, see ${dir}/chain.log" >&2
exit 1
EOF
# Read by hoprd-localcluster and its tests: use the chain started by hopr-chain-up instead of a container
ENV HOPRD_CHAIN_URL=http://127.0.0.1:8080

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Every build below happens once, so incremental compilation caches would only add ~12 GB to the image
ENV CARGO_INCREMENTAL=0

# Fetch every crate in Cargo.lock, for every feature, so later builds with other features stay offline
RUN cargo fetch --locked

# Everything is fetched. From here on cargo must not try to reach a registry.
ENV CARGO_NET_OFFLINE=true

# The release feature set (`-p hoprd -p hoprd-api -F strategy-pix-curvy`, as flake.nix builds the Docker image),
# kept as target/debug/hoprd-pix-curvy. It is built first because the workspace build below replaces
# target/debug/hoprd with the localcluster feature set.
RUN cargo build --locked -p hoprd -p hoprd-api -F strategy-pix-curvy \
    && cp target/debug/hoprd target/debug/hoprd-pix-curvy

# The workspace binaries and libraries. localcluster enables hoprd's strategy-pix-test, and the workspace build unifies
# features, so target/debug/hoprd carries the plain secp256k1 test pool that the localcluster PIX tests expect.
RUN cargo build --locked --workspace --bins --lib

# The hoprd binary the localcluster tests start
ENV HOPRD_BIN=/src/target/debug/hoprd

# Precompile every test binary so the tests run offline
RUN cargo nextest run --locked --workspace --no-run

# Run the unit tests as a smoke check, without letting a failure abort the build
RUN (timeout 900 cargo nextest run --locked --workspace --lib || echo "WARNING: unit tests reported failures")
