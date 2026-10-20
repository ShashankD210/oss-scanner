# Scanner image for https://github.com/dojofoo/dojofoo (main): a pnpm and Turborepo monorepo on Node 24.
#
# The audit runs offline, so this build installs every workspace dependency from the lockfile (pnpm applies the
# repository's patches during the install), builds the packages the `dojofoo` CLI ships, and type-checks them.
# threat_model.md, next to this file, says what each package does and how to run its tests.

# Node 24 is what the CLI, the agent runtime and authoring require ("engines": ">=24").
FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    CI=1 \
    TURBO_TELEMETRY_DISABLED=1
# git for course installs from repositories in tests; ripgrep and procps for the tests that inspect processes.
RUN sed -i 's|http://deb.debian.org|https://deb.debian.org|g' /etc/apt/sources.list.d/debian.sources \
    && apt-get update && apt-get install -y --no-install-recommends git ca-certificates ripgrep procps \
    && rm -rf /var/lib/apt/lists/*

# The pnpm version from package.json's packageManager field.
RUN corepack enable && corepack prepare pnpm@10.28.0 --activate

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Every dependency, exactly as locked, with the repository's patches applied.
RUN pnpm install --frozen-lockfile

# The packages under packages/ (CLI, config, agent runtime, authoring, local web UI). The agent build fetches the
# pinned Eve release from the npm registry and patches it, so it must run here, online. A failing step fails the build.
RUN pnpm turbo run build --filter='./packages/*'

# Type-check and run the offline unit tests so the log shows the image works, without letting a failure abort the
# build. Live tests (*.live.test.ts) call paid models and are excluded; tests that need native agent CLIs skip
# themselves when those are absent.
RUN for p in cli authoring ui evals; do (cd packages/$p && pnpm run types:check) || echo "WARNING: $p types:check failed"; done \
    && (cd packages/cli && timeout 900 pnpm exec vitest run || echo "WARNING: cli tests reported failures") \
    && (cd packages/ui && timeout 1200 pnpm exec vitest run --exclude '**/*.live.test.ts' --exclude 'e2e/**' \
          || echo "WARNING: ui tests reported failures")

# Everything is fetched. From here on pnpm and npm must not try to reach the registry.
ENV npm_config_offline=true
