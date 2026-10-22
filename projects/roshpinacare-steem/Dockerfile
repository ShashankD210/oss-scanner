# roshpinacare-sys — sovereign fleet project (Next.js + Bun)
# The scanner builds this with the repository checkout as the build context.
FROM oven/bun:1

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends       ca-certificates git python3 openssl     && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# install dependencies (network is available during this build only)
RUN bun install

# generate the prisma client (the schema ships with the repo)
RUN bunx prisma generate

# build the way the maintainers build it (standalone Next.js output)
RUN bun run build

# cheap smoke so the image is known-good before scans start
RUN bun run lint
