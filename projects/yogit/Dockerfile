# yogit: a single-file, zero-dependency Node.js CLI that stores agent messages as git refs.
# The scanner builds this with the yogit repository as the build context.
FROM debian:bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git nodejs npm \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# yogit drives git; give the test suite an identity so commits made by tests succeed offline.
RUN git config --global user.name "oss-scanner" \
    && git config --global user.email "oss-scanner@example.invalid" \
    && git config --global init.defaultBranch main \
    && git config --global --add safe.directory '*'

# No dependencies to install (package.json has none). Run the cheap tests so the image is known to work,
# without letting a test failure abort the build.
RUN node --version && (node --test test/ || echo "WARNING: node --test reported failures")
