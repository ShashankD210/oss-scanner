# Source review image for an Unraid plugin (PHP / shell). There is nothing to compile: the plugin is
# PHP pages, PHP/shell helpers and a .plg installer. PHP and shell tooling is installed so the agent can
# run the plugin's own test suite and lint the code.
FROM php:8.3-cli-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git shellcheck jq xmlstarlet python3 \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Cheap sanity checks; failures are reported but do not abort the build.
RUN find /src -name '*.php' -not -path '*/node_modules/*' -print0 | xargs -0 -n1 php -l >/dev/null || echo "WARNING: php -l reported errors"
RUN if [ -f /src/tests/run.sh ]; then (cd /src/tests && sh run.sh) || echo "WARNING: tests reported failures"; fi
