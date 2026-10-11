# go.mod requires Go >= 1.26.3; pin the patched 1.26 toolchain.
FROM golang:1.26.9-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    GOTOOLCHAIN=local \
    CGO_ENABLED=0
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git nodejs file \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src/host
# Fetch all locked module dependencies online and retain the module cache.
RUN go mod download all
# Match the installer's Windows/amd64 target, but retain debug information.
# Do not run install.sh: it registers a host and changes Windows trust stores.
RUN mkdir -p /src/dist \
    && GOOS=windows GOARCH=amd64 go build -mod=readonly -trimpath \
         -ldflags="-H windowsgui" -o /src/dist/passkey-bridge.exe .

WORKDIR /src
# The extension is plain JavaScript/JSON with no package dependencies or bundling.
RUN node --check extension/background.js \
    && node -e 'JSON.parse(require("fs").readFileSync("extension/manifest.json", "utf8"))'

# Disable implicit module/checksum downloads for subsequent offline Go work.
ENV GOPROXY=off GOSUMDB=off
CMD ["/bin/bash"]
