# unraid-disklocation-next: a Node.js 22 (TypeScript) daemon plus a Svelte/Vite frontend.
FROM node:22-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates git python3 php-cli shellcheck \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Install every dependency here, while the network is available, then build.
RUN npm ci
RUN npm run build || echo "WARNING: tsc build reported errors"
RUN npm run build:frontend || echo "WARNING: frontend build reported errors"
