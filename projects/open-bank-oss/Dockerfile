# syntax=docker/dockerfile:1
# OpenBank: a Gradle (Kotlin/Quarkus, JDK 25) monorepo of ~50 services plus a Next.js admin UI.
FROM eclipse-temurin:25-jdk-noble
RUN apt-get update && apt-get install -y --no-install-recommends git curl ca-certificates xz-utils \
    && rm -rf /var/lib/apt/lists/*
# Node 22 for openbank-admin-ui.
RUN arch="$(dpkg --print-architecture)"; [ "$arch" = amd64 ] && arch=x64; \
    curl -fsSL "https://nodejs.org/dist/v22.20.0/node-v22.20.0-linux-${arch}.tar.xz" \
      | tar -xJ -C /usr/local --strip-components=1

COPY . /src
WORKDIR /src
# .git is excluded by the repo .dockerignore; the build only stamps this commit into build-info metadata.
ENV GRADLE_OPTS="-Dorg.gradle.daemon=false" \
    SOURCE_COMMIT=0000000000000000000000000000000000000000
# The fleet needs a larger heap than the repo default; cap workers so the Kotlin compilers fit too.
ARG GRADLE_FLAGS="-Dorg.gradle.jvmargs=-Xmx6g -Dorg.gradle.workers.max=4"

# Compile every module (main + test sources) so all dependencies land in ~/.gradle for --offline use.
# --continue: one module failing to compile must not leave the rest of the fleet unbuilt.
# Retried: a transient download failure on one shared library otherwise blocks every module.
RUN for attempt in 1 2 3; do \
      ./gradlew $GRADLE_FLAGS assemble testClasses --continue -x quarkusBuild && exit 0; \
      echo "gradle attempt $attempt failed, retrying"; \
    done; echo "WARNING: some modules failed to compile"

# Pull the runtime-only test dependencies by running one small unit test.
RUN (timeout 900 ./gradlew :openbank-libs-domain:test $GRADLE_FLAGS \
    || echo "WARNING: libs-domain tests reported failures")

# Admin UI dependencies.
RUN cd openbank-admin-ui && npm ci --no-audit --no-fund
