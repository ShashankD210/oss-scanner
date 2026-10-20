# olcRTC (https://github.com/ghostlane-project/olcrtc), built with its checkout as the context.
# The scan that follows has no network, so this fetches every module and warms the build cache
# for the flavours CI builds: the default one, the race detector's, olcrtc_lean (the mobile apps'
# build) and olcrtc_testhooks.
FROM golang:1.26.3

# iproute2: the tests that connect two pion peers need an interface besides lo, and a sandbox with
# no network has none; `ip` adds a dummy one (.oss-scanner/threat_model.md, "How to exercise it").
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends iproute2 \
    && rm -rf /var/lib/apt/lists/*

# The checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

# go.mod names the toolchain; GOTOOLCHAIN=auto (the default) fetches a newer one here if go.mod
# ever asks for it, while the network is still there.
RUN go mod download && go mod verify && go version

# The binaries a scan can run: the command line program (srv and cnc modes), its lean flavour,
# and the c-shared library desktop clients load.
RUN mkdir -p /src/build \
    && go build -o /src/build/olcrtc ./cmd/olcrtc \
    && go build -tags olcrtc_lean -o /src/build/olcrtc-lean ./cmd/olcrtc \
    && go build -buildmode=c-shared -o /src/build/libolcrtc.so ./cmd/olcrtc-cgo \
    && go build ./...

# Every test binary of the four flavours compiled; the suite run with the race detector as CI runs
# it. A failing test does not stop the build.
RUN go vet ./... || echo "WARNING: go vet reported problems"
RUN go test -count=1 -run '^$' ./... >/dev/null \
    && go test -count=1 -run '^$' -tags olcrtc_lean ./... >/dev/null \
    && go test -count=1 -run '^$' -tags olcrtc_testhooks ./... >/dev/null \
    || echo "WARNING: a test binary did not compile"
RUN go test -count=1 -race -timeout 20m ./... || echo "WARNING: go test -race reported failures"
