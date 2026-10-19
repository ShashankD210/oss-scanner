FROM golang:1.26-trixie

ENV DEBIAN_FRONTEND=noninteractive \
    CGO_ENABLED=0 \
    GOTOOLCHAIN=local

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git jq procps \
    && rm -rf /var/lib/apt/lists/*

# The scanner supplies the relay checkout as the build context. It is a go.work
# workspace of five modules: the server (.), sdk, jobq, auth and secret.
COPY . /src
WORKDIR /src

# The embedded admin UI is excluded by .dockerignore and fetched by the release
# build; an empty dist keeps go:embed satisfied without it.
RUN mkdir -p cmd/relay/web/dist && touch cmd/relay/web/dist/.gitkeep

RUN go mod download \
    && for m in sdk jobq auth secret; do (cd "$m" && go mod download); done

RUN go build ./... ./sdk/... ./jobq/... ./auth/... ./secret/... \
    && go build -o /usr/local/bin/relay ./cmd/relay

# Unit tests only: integration tests (build tag "integration") need Postgres and
# are not compiled here. A failing baseline test must not fail the image.
RUN go test -short -count=1 -p 4 -timeout 20m ./... ./sdk/... ./jobq/... ./auth/... ./secret/... \
    || echo 'WARNING: baseline unit tests reported failures'

# Reproducers and patches must build with the network disabled.
ENV GOPROXY=off \
    GOSUMDB=off

CMD ["bash"]
