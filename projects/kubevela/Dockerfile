# KubeVela build for the OSS Scanner.
# The scanner supplies the kubevela checkout as the build context. Network is available during this build
# and removed afterwards, so the Go toolchain, module cache, envtest binaries and test caches are all
# fetched here and kept in the image.
FROM golang:1.23-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    GOTOOLCHAIN=local \
    CGO_ENABLED=0

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git make jq procps \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download

# Build the three binaries the project ships: the controller, the CLI and the kubectl plugin.
RUN go build -o /usr/local/bin/manager ./cmd/core \
    && go build -o /usr/local/bin/vela ./references/cmd/cli \
    && go build -o /usr/local/bin/kubectl-vela ./cmd/plugin

# Unit tests run a real kube-apiserver and etcd through controller-runtime envtest. Fetch those binaries now
# so the suite works offline. The release branch matches the controller-runtime version in go.mod (v0.19.x);
# the Kubernetes version matches ENVTEST_K8S_VERSION in makefiles/dependency.mk.
RUN GOBIN=/usr/local/bin go install sigs.k8s.io/controller-runtime/tools/setup-envtest@release-0.19 \
    && setup-envtest use 1.31.0 --bin-dir /opt/envtest -p path \
    && ln -s "$(setup-envtest use 1.31.0 --bin-dir /opt/envtest -p path)" /opt/kubebuilder-assets
ENV KUBEBUILDER_ASSETS=/opt/kubebuilder-assets

# Run the unit tests the same way `make unit-test-core` does, so the build and test caches are warm offline.
# A failing baseline test must not abort the image build; the scanner still needs the environment.
RUN go test -timeout 30m $(go list ./pkg/... ./cmd/... ./apis/... | grep -v apiserver | grep -v applicationconfiguration) \
    || echo "WARNING: core unit tests reported failures"
RUN go test -timeout 20m $(go list ./references/... | grep -v apiserver) \
    || echo "WARNING: CLI unit tests reported failures"

# New reproducers and patches must also build with the network disabled.
ENV GOPROXY=off \
    GOSUMDB=off

CMD ["bash"]
