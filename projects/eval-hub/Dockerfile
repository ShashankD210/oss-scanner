# Keep the checkout, build tools, and dependency caches for offline analysis.
# Match the minimum Go version in EvalHub's go.mod.
FROM golang:1.26.7-bookworm

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git make python3 python3-venv \
    && rm -rf /var/lib/apt/lists/*

# Scanner contract: the complete repository is available at /src.
COPY . /src
WORKDIR /src

# Fetch Go dependencies and the Python SDK used by the local test adapter
# while the build still has network access. Keep the SDK outside the checkout.
RUN go mod download \
    && python3 -m venv /opt/evalhub-venv \
    && /opt/evalhub-venv/bin/pip install --no-cache-dir 'eval-hub-sdk[adapter]>=1.0.4'
ENV PATH="/opt/evalhub-venv/bin:${PATH}"

# Build the API, runtime init, sidecar, and MCP binaries as in the Makefile.
# Compile all Go tests without executing suite setup that needs external services.
RUN make build \
    && go test -race -exec /bin/true ./...

# No module or toolchain downloads should be needed during the audit.
ENV GOPROXY=off \
    GOSUMDB=off \
    GOTOOLCHAIN=local

# These integration tests provision MLflow/PostgreSQL even with -short.
# BuildKit injects an OTLP endpoint; keep it out of the application's TLS tests.
# Record other baseline failures without preventing the scanner from studying them.
RUN env -u OTEL_EXPORTER_OTLP_ENDPOINT -u OTEL_EXPORTER_OTLP_TRACES_ENDPOINT \
      -u OTEL_EXPORTER_OTLP_INSECURE -u OTEL_EXPORTER_OTLP_TRACES_INSECURE \
      timeout 1200 go test -race -short -timeout 10m \
      -skip '^(TestMLFlowIntegration|TestGetEvaluationJobs_Postgres)$' \
      ./internal/... ./cmd/... ./pkg/... \
    || echo 'WARNING: baseline Go tests reported failures; inspect the build log'

CMD ["bash"]
