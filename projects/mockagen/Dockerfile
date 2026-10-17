# Scanner image for https://github.com/catdevman/mockagen (main).
# Unit tests are `go test ./...` (mise task `test`). The integration tests start
# database servers through testcontainers, so they are compiled here and not run.
FROM golang:1.27.0-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    GOTOOLCHAIN=local

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git gcc libc6-dev \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download

RUN go build -o /usr/local/bin/mockagen ./cmd/mockagen

# Compile the integration-tagged tests into the build cache. Running them needs
# a Docker daemon, which the scan does not have.
RUN go test -tags=integration -c -o /tmp/mockagen.integration.test ./cmd/mockagen \
    && rm -f /tmp/mockagen.integration.test

RUN timeout 600 go test -count=1 -timeout 8m ./...

ENV GOPROXY=off \
    GOSUMDB=off
