# Source review image. The plugin is a single .plg installer (inline PHP and shell) plus a Community Applications profile; nothing to compile.
FROM debian:bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 php-cli shellcheck jq xmlstarlet \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src
