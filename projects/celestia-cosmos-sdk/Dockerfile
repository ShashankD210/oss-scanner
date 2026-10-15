FROM golang:1.26.9-trixie

ENV DEBIAN_FRONTEND=noninteractive \
    GOTOOLCHAIN=local \
    GOMAXPROCS=2 \
    GOFLAGS="-p=2 -mod=readonly"

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git make \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# Each nested module has its own dependency graph and test packages.
RUN set -eu; find . -name go.mod -not -path './.git/*' | sort | \
    while read -r modfile; do \
      (cd "$(dirname "$modfile")" && (go mod download || go mod download)); \
    done

RUN cd simapp && go build -tags 'netgo ledger' -o /src/build/simd ./simd

# Populate the test cache while downloads are available, including tagged E2E tests.
RUN set -eu; find . -name go.mod -not -path './.git/*' | sort | \
    while read -r modfile; do \
      (cd "$(dirname "$modfile")" && go test -run '^$' -timeout 10m -tags 'ledger test_ledger_mock' ./...); \
    done \
    && cd /src/tests && go test -run '^$' -timeout 10m -tags 'e2e ledger test_ledger_mock' ./e2e/... \
    && cd /src/simapp && go test -run '^$' -timeout 10m -tags 'app_v1 ledger test_ledger_mock' ./...

RUN go test -short -timeout 10m -tags 'ledger test_ledger_mock' ./crypto/... ./types/... \
    && cd /src/math && go test -short -timeout 5m ./... \
    && cd /src && git diff --exit-code -- ':(glob)**/go.mod' ':(glob)**/go.sum' go.mod go.sum

ENV GOPROXY=off
