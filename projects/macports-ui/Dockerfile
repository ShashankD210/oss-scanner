# Source review image. This is a macOS Swift app with a privileged helper (Xcode project), which cannot be compiled on Linux, so the image holds the sources for review; no build step.
FROM debian:bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 php-cli shellcheck jq xmlstarlet \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src
