# Scanner image for https://github.com/catdevman/gitgrove (main).
# Tests shell out to git and create symlinks, matching CI: go test -race ./...
FROM golang:1.26.2-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    GOTOOLCHAIN=local

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git gcc libc6-dev \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download

RUN go build -o /usr/local/bin/gitgrove .

# Race detector needs cgo, which gcc above provides. Bounded so a stuck git
# subprocess cannot consume the build.
# The scan runs as root, and root ignores the mode bits these three tests chmod
# onto a directory or a file. They pass for an ordinary user and fail here.
RUN timeout 900 go test -race -count=1 -timeout 10m \
      -skip '^(TestSaveReportsAnUnwritableDirectory|TestEnsureReportsAnUnreadableDestination|TestCopyDirReportsAnUnreadableFile)$' \
      ./...

ENV GOPROXY=off \
    GOSUMDB=off
