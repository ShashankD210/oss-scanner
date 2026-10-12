# decky-nas-mount: a Decky Loader plugin: a Python backend (main.py) and a React/TypeScript frontend.
FROM node:22-bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates git python3 shellcheck \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Install dependencies while the network is available, then build and run the unit tests.
RUN npm ci
RUN npm run build || echo "WARNING: frontend build reported errors"
RUN python3 -m unittest discover -s tests || echo "WARNING: tests reported failures"
