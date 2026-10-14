# Scanner image for https://github.com/argoproj-labs/gitops-promoter.
#
# The audit runs offline. This build fetches both Go modules, the envtest Kubernetes
# assets the controller tests need, and compiles the manager and its tests.
# threat_model.md, next to this file, says how to exercise the result.

# Same image the repository's Dockerfile pins. It is Debian-based and includes git and gcc.
FROM golang:1.27.1@sha256:512690a5660563b57d37ecc31129e7f136e831db2aed24a1dbeb8ad7380dc0fa

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates git make \
    && rm -rf /var/lib/apt/lists/* \
    && git config --global user.email "scanner@example.com" \
    && git config --global user.name "OSS Scanner" \
    && git config --global init.defaultBranch main

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Root module, plus hack/celcost which is its own module.
RUN go mod download \
    && (cd hack/celcost && go mod download)

# The manager binary, built the same way the repository's Dockerfile builds it.
RUN CGO_ENABLED=0 go build -o /usr/local/bin/gitops-promoter ./cmd/main.go

# Kubebuilder assets for controller and apiserver envtest. Version matches ENVTEST_K8S_VERSION
# and ENVTEST_VERSION in the Makefile.
RUN go install sigs.k8s.io/controller-runtime/tools/setup-envtest@release-0.25 \
    && mkdir -p /usr/local/kubebuilder \
    && assets="$(/go/bin/setup-envtest use 1.37.0 --bin-dir /usr/local/kubebuilder -p path)" \
    && ln -sfn "$assets" /usr/local/kubebuilder/assets
ENV KUBEBUILDER_ASSETS=/usr/local/kubebuilder/assets

# Compile every non-e2e test so the build cache is warm offline. -run '^$' compiles
# without executing suites. A failure here fails the image build.
RUN go test -run '^$' $(go list ./... | grep -v /test/e2e)

# Unit tests that do not start an envtest cluster. Controller envtest is available
# (KUBEBUILDER_ASSETS is set) but slow; run it from the shell. A failure here does
# not abort the image build.
RUN go test -count=1 -timeout 25m \
        ./internal/git/... \
        ./internal/webhookreceiver/... \
        ./internal/utils/... \
        ./internal/scms/... \
        ./internal/labels/... \
        ./internal/webrequest/... \
    || echo "WARNING: unit tests reported failures"

# Everything is fetched. From here on the go command must not reach a proxy or download a toolchain.
ENV GOPROXY=off \
    GOTOOLCHAIN=local
