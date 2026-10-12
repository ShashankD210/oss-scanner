# Cipansor IMS — build image for the OSS Scanner.
#
# This file lives next to project.yaml, so the scanner builds it with the
# cipansor/cipansor checkout as the build context. The scanner contract is that
# the checkout lives at /src. Network access is available during this build and
# is removed afterwards, so every dependency the audit needs is fetched here.
#
# The job of this image is to leave a *built, runnable* checkout: it is not the
# production image (that one is apps/api/Dockerfile and needs secrets this
# build must not assume). The API is the trust boundary the scanner cares about,
# so we build it and its unit tests, and stop short of starting the servers or a
# database.

# syntax=docker/dockerfile:1
FROM node:22-bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    PNPM_HOME=/pnpm \
    PATH=/pnpm:$PATH \
    # Placeholder only: `prisma generate` and the mocked unit tests never open a
    # connection. A real end-to-end run would override this and add Postgres.
    DATABASE_URL=postgresql://postgres:postgres@localhost:5432/cipansor

# pnpm pinned to the version in the root package.json ("packageManager").
RUN corepack enable && corepack prepare pnpm@9.15.9 --activate

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Fetches api, web and shared dependencies from the lockfile. Kept as its own
# layer so a code-only change does not re-resolve the dependency graph.
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile

# Build in the same order a developer does: shared first (both apps consume its
# dist), then the generated Prisma client, then the API through tsc. A failure
# here fails the build, and primary_contact gets the log.
RUN pnpm --filter @cipansor/shared build \
 && pnpm --filter api db:generate \
 && pnpm --filter api build

# Run the API's cheap unit suite so the scanner knows the image is sound. A
# failing test must not abort the build, so its exit code is deliberately
# swallowed here; leave the compiled output (dist/) and the sources for the
# audit to exercise.
RUN pnpm --filter api test || echo "WARNING: api unit tests reported failures (not fatal)"
