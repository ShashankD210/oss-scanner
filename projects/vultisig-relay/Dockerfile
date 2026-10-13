FROM golang:1.25.1-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends redis-server \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

RUN go mod download && go build -gcflags=all="-N -l" ./... \
    && go build -o /usr/local/bin/vultisig-relay ./cmd/...
RUN (redis-server --daemonize yes && go test ./... -timeout 300s) || echo "WARNING: go test reported failures"
