FROM golang:1.26-bookworm

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 ripgrep \
    && rm -rf /var/lib/apt/lists/*

ENV GOTOOLCHAIN=local

COPY . /src
WORKDIR /src

# Keep the module and build caches in the image for the offline audit.
RUN go mod download \
    && go build -o /src/bin/kontext ./cmd/kontext \
    && go test -run '^$' ./...

CMD ["bash"]
