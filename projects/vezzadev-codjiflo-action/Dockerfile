FROM node:22-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    npm_config_nodedir=/usr/local \
    npm_config_build_from_source=true \
    CFLAGS="-g -fno-omit-frame-pointer" \
    CXXFLAGS="-g -fno-omit-frame-pointer"

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

# These are upstream build dependencies, not a replacement for the action under /src.
# Preserve the pinned monorepo lock: the standalone lock links the engine but does
# not contain the engine's resolved diff-match-patch/type dependency records.
ARG CODJIFLO_COMMIT=0ef8bebd1f336accb171e9c7b79af3bea46227e9
RUN mkdir -p /opt/codjiflo-dependencies /packages /scripts \
    && curl --fail --location --retry 3 \
       "https://codeload.github.com/vezzadev/codjiflo/tar.gz/${CODJIFLO_COMMIT}" \
       | tar -xz --strip-components=1 -C /opt/codjiflo-dependencies
WORKDIR /opt/codjiflo-dependencies
# Only this TypeScript-source workspace is needed. Its runtime dependency and
# types are pure JS/type declarations; no monorepo app lifecycle is applicable.
# The action's own native/esbuild installation below DOES run lifecycle scripts.
RUN npm ci --workspace=@codjiflo/diff-engine --include-workspace-root=false --ignore-scripts \
    && ln -s /opt/codjiflo-dependencies/packages/diff-engine /packages/diff-engine \
    && cp /opt/codjiflo-dependencies/scripts/generate-schema.cjs /scripts/generate-schema.cjs

COPY . /src
WORKDIR /src
# The unmodified upstream schema generator expects ../action/ from /scripts.
RUN ln -s /src /action \
    && npm ci \
    && npm run build

# Audit/test execution is offline. Dependencies and Node native headers have
# already been fetched; no GitHub token, API calls, deployment or tests run here.
CMD ["bash"]
