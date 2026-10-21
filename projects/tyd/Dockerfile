# Scanner image for https://github.com/fdaio/tyd (main).
#
# The audit runs without Internet access, so this build fetches every Go module the repository needs, builds the
# three binaries and compiles every test package. threat_model.md, next to this file, says what to test and how to
# rate a finding.
#
# golang:1.26-bookworm is the Go version in go.mod, and it is Debian, so the scanner's own layer installs its tools
# on top. git, make and python3 come with the image: `make test` and the acceptance script need them.

FROM golang:1.26-bookworm
ENV DEBIAN_FRONTEND=noninteractive

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Every module the daemon, the CLI, the Control Panel and the relay need, the test-only ones included.
RUN go mod download

# The binaries and every test package compile, with and without the race detector. A step that fails fails the
# build, and primary_contact gets the log.
RUN mkdir -p bin \
    && go build -o bin/tyd ./cmd/tyd \
    && go build -o bin/controlpanel ./cmd/controlpanel \
    && go build -o bin/relay ./cmd/relay \
    && go vet ./... \
    && go test -run '^$' ./... \
    && go test -race -run '^$' ./...

# Run the tests and the acceptance script so the log shows the image works, but do not let a failure abort the build.
RUN (go test ./... || echo "WARNING: go test reported failures") \
    && (make acceptance || echo "WARNING: make acceptance reported failures")

# Everything is fetched. From here on the go command must not try to reach a proxy or download a toolchain.
ENV GOPROXY=off \
    GOTOOLCHAIN=local
