# syntax=docker/dockerfile:1
# Keep the source, compilers, dependency caches and test drivers for offline audit.
FROM node:22-bookworm-slim AS node
FROM golang:1.25-bookworm
COPY --from=node /usr/local/ /usr/local/
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential clang cmake pkg-config libpcre2-dev python3 git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src/gateway
RUN go mod download
WORKDIR /src/frontend
RUN npm ci

WORKDIR /src
ENV CGO_ENABLED=1 \
    CGO_LDFLAGS="-L/opt/silencer-native -Wl,-rpath,/opt/silencer-native" \
    LD_LIBRARY_PATH=/opt/silencer-native \
    GOPROXY=off \
    GOTOOLCHAIN=local
RUN mkdir -p /opt/silencer-native /opt/silencer-bin \
    && g++ -std=c++17 -g -O1 -fno-omit-frame-pointer -Wall -Wextra -Werror -fPIC -shared \
       engine/cpp/engine.cpp -Iengine/cpp -lpcre2-8 -o /opt/silencer-native/libsilencer_engine.so \
    && g++ -std=c++17 -g -O1 -fno-omit-frame-pointer -pthread -Wall -Wextra -Werror \
       engine/cpp/engine_test.cpp -Iengine/cpp -L/opt/silencer-native -lsilencer_engine \
       -Wl,-rpath,/opt/silencer-native -o /opt/silencer-native/engine_test
RUN cd gateway \
    && go build -tags silencer_engine -o /opt/silencer-bin/silencer-gateway ./cmd/gateway \
    && go build -o /opt/silencer-bin/silencer-migrate ./cmd/migrate \
    && go build -o /opt/silencer-bin/silencer-host-agent ./cmd/host-agent
RUN cd frontend && npm run build

COPY <<'EOF' /usr/local/bin/silencer-test
#!/usr/bin/env bash
set -euo pipefail
cd /src
/opt/silencer-native/engine_test
cd gateway
go test -count=1 ./...
go test -count=1 -tags silencer_engine ./...
cd /src/frontend
npm run build
npm run lint
npm run test:logs
npm run test:capture
EOF
RUN chmod +x /usr/local/bin/silencer-test \
    && (silencer-test > /opt/silencer-test-build.log 2>&1 \
        || { cat /opt/silencer-test-build.log; echo 'WARNING: tests reported failures; inspect /opt/silencer-test-build.log'; })
ENV SILENCER_WEB_DIST=/src/frontend/dist \
    SILENCER_EXTENSION_DISABLED=1 \
    SILENCER_ENGINE=cpp
WORKDIR /src
CMD ["bash"]
