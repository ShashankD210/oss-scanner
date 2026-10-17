# Scanner image for https://github.com/cloudfoundry-community/ocf-scheduler (develop).
#
# The audit runs offline, so this build fetches every module, builds the scheduler, its tools and the mock UAA and
# CF API servers into /src/bin, compiles the tests, and leaves a local PostgreSQL that the tests and the scheduler
# use. threat_model.md, next to this file, says how to run each of them.

# go.mod asks for Go 1.26. The image has git and gcc, which the race detector needs.
FROM golang:1.26-bookworm
ENV DEBIAN_FRONTEND=noninteractive
# PostgreSQL for the store tests and a running scheduler; curl to drive its HTTP API. Packages are fetched over HTTPS.
RUN sed -i 's|http://deb.debian.org|https://deb.debian.org|g' /etc/apt/sources.list.d/debian.sources \
    && apt-get update && apt-get install -y --no-install-recommends postgresql curl \
    && rm -rf /var/lib/apt/lists/*

# A database named test, owned by postgres/postgres, as docker-compose.yml sets up for `make test`.
# start-test-db starts the cluster; nothing runs between image layers.
RUN printf '#!/bin/sh\nset -e\npg_ctlcluster 15 main start\n' > /usr/local/bin/start-test-db \
    && chmod +x /usr/local/bin/start-test-db \
    && start-test-db \
    && su postgres -c "psql -q -c \"ALTER USER postgres PASSWORD 'postgres'\"" \
    && su postgres -c "createdb test" \
    && pg_ctlcluster 15 main stop
ENV DATABASE_URL=postgres://postgres:postgres@localhost/test?sslmode=disable

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download

# Every command (the scheduler, tzlist, the dev CLI, and the mock UAA and CF API servers), then the tests with and
# without -race. A step that fails fails the build, and primary_contact gets the log.
RUN mkdir -p bin \
    && go build -o bin/ ./cmd/... \
    && go test -run '^$' ./... \
    && go test -race -run '^$' ./...

# Run the unit and integration tests as `make test` does, so the log shows the image works, without letting a failure
# abort the build.
RUN start-test-db \
    && (timeout 1200 ./scripts/blanket || echo "WARNING: scripts/blanket reported failures") \
    && pg_ctlcluster 15 main stop

# Everything is fetched. From here on the go command must not try to reach a proxy or download a toolchain.
ENV GOPROXY=off \
    GOTOOLCHAIN=local
