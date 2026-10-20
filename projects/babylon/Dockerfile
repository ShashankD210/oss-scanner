# Retain the source, Go toolchain, module cache and debug symbols for analysis.
FROM golang:1.25.8-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CGO_ENABLED=1 \
    GOPROXY=https://proxy.golang.org,direct
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential clang cmake ninja-build pkg-config git python3 \
      ca-certificates curl jq \
    && rm -rf /var/lib/apt/lists/*

# The scanner supplies the Babylon checkout as the build context.
COPY . /src
WORKDIR /src
RUN go mod download
RUN LEDGER_ENABLED=false BABYLON_BUILD_OPTIONS=nostrip make build

# Compile the security-focused tests while network access is available.
# Go's module cache includes the platform-specific dynamic wasmvm library.
RUN mkdir -p /opt/babylon-tests && \
    for package in $(go list ./btcstaking/... ./crypto/... \
      ./x/btcstaking/... ./x/finality/... ./x/btclightclient/... \
      ./x/btccheckpoint/...); do \
      name=$(printf '%s' "$package" | tr '/' '_'); \
      go test -c -o "/opt/babylon-tests/${name}.test" "$package" || exit 1; \
    done
# A smoke test is useful, but failure must not prevent scanner analysis.
RUN go test -count=1 -timeout=5m ./btcstaking/... ./crypto/... \
    || echo 'WARNING: Babylon staking/crypto smoke tests reported failures'
