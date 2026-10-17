# Scanner image for https://github.com/cloudfoundry-community/ocf-scheduler-cf-plugin (develop).
#
# The audit runs offline, so this build fetches every module, generates the cron help text, builds the plugin and
# compiles the tests. threat_model.md, next to this file, says how to run them.

# go.mod asks for Go 1.26. The image has git and gcc, which the race detector needs.
FROM golang:1.26-bookworm
ENV DEBIAN_FRONTEND=noninteractive

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download

# `make build` without the release metadata: go generate renders the cron help (cron_expression_styles.go), then
# the plugin binary lands at /src/ocf-scheduler-cf-plugin. A step that fails fails the build, and primary_contact
# gets the log.
RUN go generate ./... \
    && go build -o ocf-scheduler-cf-plugin . \
    && go test -run '^$' ./... \
    && go test -race -run '^$' ./...

# Run the unit tests so the log shows the image works, without letting a failure abort the build. acceptance/ needs
# a live Cloud Foundry and is left out.
RUN (timeout 900 go test -race $(go list ./... | grep -v /acceptance) || echo "WARNING: go test reported failures")

# Everything is fetched. From here on the go command must not try to reach a proxy or download a toolchain.
ENV GOPROXY=off \
    GOTOOLCHAIN=local
