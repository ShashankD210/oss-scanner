FROM golang:1.25-bookworm
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates git make \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download

# server and CLI tools. The default build serves a stub web console, so no Node toolchain is needed.
RUN go build -o /usr/local/bin/ ./cmd/immudb ./cmd/immuadmin ./cmd/immuclient

# run the unit tests so the build and test caches are warm offline, without failing the image build.
# The skipped tests expect a permission denied error on a read-only path, which root (the scanner's user) never gets.
ENV LOG_LEVEL=error \
    GOFLAGS="-skip=^(TestOpenFail|TestInvalidOpening|TestImmudbStoreEdgeCases|TestHistoryFileCache_SetMissingFolder|TestDbCreationInInvalidDirectory|TestStreamUtilsFiles)$"
RUN go test -timeout 45m ./embedded/... ./pkg/... || echo "WARNING: tests reported failures"
