# Unpkg is pure Go (no cgo); go.mod requires go 1.26.
FROM golang:1.26.9-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CGO_ENABLED=0 \
    GOTOOLCHAIN=local

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential python3 \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download \
    && go build ./... \
    && go build -o /usr/local/bin/ljdump ./cmd/ljdump

# Run cheap tests and warm the build cache, without failing the build.
RUN go test -short -timeout 20m ./... || echo "WARNING: go test reported failures"

# Audit runs offline: prevent implicit module/toolchain downloads.
ENV GOPROXY=off \
    GOSUMDB=off

CMD ["bash"]
