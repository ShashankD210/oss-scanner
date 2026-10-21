# FloorVote: a TypeScript monorepo of Cloudflare Workers (api/, central/, root/) and a React web app (web/).
# Mirrors the project's CI (.github/workflows/ci.yml): Node 26, `npm ci` at the root and in central/.
FROM node:26-bookworm
ENV CI=1

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Install every dependency while the network is available, including the workerd binary the Worker tests run on.
RUN npm ci && (cd central && npm ci)

# Build the web app as CI does. Typecheck the Workers but do not fail the image on type errors.
RUN cd web && npm run build
RUN (cd api && npx tsc --noEmit -p tsconfig.build.json) \
    && (cd central && npx tsc --noEmit -p tsconfig.build.json) \
    || echo "WARNING: typecheck reported errors"

# Run the cheap tests so we know the image works offline, but do not let a failing test abort the build.
RUN (cd api && npm test) || echo "WARNING: api tests reported failures"
RUN (cd central && npm test) || echo "WARNING: central tests reported failures"
