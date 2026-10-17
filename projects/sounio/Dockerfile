# Build environment for Sounio. The scanner builds this with the repository as the build context.
# Sounio ships a committed static Linux x86-64 compiler (bin/madaros-linux-x86_64, launched through bin/souc),
# so no toolchain build is needed to exercise the compiler. Network is available only during this build.
FROM debian:bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential clang gdb git make python3 binutils file \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

ENV SOUNIO_STDLIB_PATH=/src/stdlib \
    SOUC_BIN=/src/bin/souc

# Make sure the committed launchers and compiler ELFs are executable.
RUN chmod +x bin/souc bin/madaros bin/madaros-linux-x86_64 2>/dev/null || true \
    && (test -f bin/madaros-linux-x86_64 || gunzip -k bin/madaros-linux-x86_64.gz) \
    && chmod +x bin/madaros-linux-x86_64

# Smoke test: the compiler must identify itself, type-check and run the hello-world example.
RUN bin/souc --version && bin/souc info \
    && bin/souc check examples/hello.sio \
    && bin/souc run examples/hello.sio

# Run the cheap regression gates, but do not let a failing gate abort the build.
RUN (bash scripts/ci/real_language_runner_gate.sh || echo "WARNING: real_language_runner_gate reported failures") \
    && (bash scripts/gates/g6_madaros_identity.sh || echo "WARNING: g6_madaros_identity reported failures")
