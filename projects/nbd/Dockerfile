# Build image for Anthropic's OSS Scanner (https://github.com/anthropics/oss-scanner).
# Variant stored in anthropics/oss-scanner as projects/nbd/Dockerfile (next to project.yaml),
# so that the NBD repository itself stays untouched.
#
# Contract: this file is built ONCE WITH NETWORK ACCESS, with the repository as build
# context. The audit then runs with NO network access, so everything the build, the tests
# and the analysis tooling need must be installed or fetched here. The checkout lives in the
# image at /src.
#
# Steps 1-4 are mandatory (system deps, pinned Rust toolchain, dependencies, builds for ALL
# cargo features): a failure there fails the build on purpose.
# Steps 5-8 are best effort (extra analysis tooling, smoke tests): a failure there only
# prints a WARNING, so it can never prevent the scan from starting.

FROM debian:bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_HOME=/opt/cargo \
    RUSTUP_HOME=/opt/rustup \
    PATH=/opt/cargo/bin:$PATH \
    CARGO_TERM_COLOR=never \
    RUST_BACKTRACE=1

# 1. System dependencies --------------------------------------------------------------------
# - build: C/C++ toolchain + cmake (rdkafka "cmake-build" compiles the bundled librdkafka)
# - features: libssl (kafka-tls), libsasl2 + krb5 (kafka-auth-gssapi), libcurl (kafka-auth-oauth)
# - analysis: debuggers, tracers, packet tools and iperf to write and run reproducers
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates curl git build-essential cmake pkg-config perl python3 clang \
        libssl-dev zlib1g-dev libzstd-dev liblz4-dev \
        libsasl2-dev libsasl2-modules-gssapi-mit libkrb5-dev libcurl4-openssl-dev \
        gdb valgrind strace ltrace tcpdump iproute2 iputils-ping netcat-openbsd socat \
        iperf iperf3 jq procps \
    && rm -rf /var/lib/apt/lists/*

# 2. Rust toolchain (rustup only, the exact version comes from rust-toolchain.toml) -----------
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
    | sh -s -- -y --profile minimal --default-toolchain none

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# `tr -d '\r'`: a checkout made with git's autocrlf has CRLF line endings, and a trailing CR
# would otherwise end up in the toolchain name.
RUN set -eux; \
    CHAN="$(sed -n 's/^channel *= *"\([^"]*\)".*/\1/p' rust-toolchain.toml | tr -d '\r')"; \
    test -n "$CHAN"; \
    rustup toolchain install "$CHAN" --profile minimal \
        -c rustfmt -c clippy -t x86_64-unknown-linux-gnu; \
    rustup default "$CHAN"; \
    rustc -Vv; cargo -V

# 3. Dependencies, fetched while online (Cargo.lock is authoritative: --locked) -------------
RUN cargo fetch --locked

# 4. Builds: ALL features (metrics-exporter, kafka-tls, kafka-auth-gssapi, kafka-auth-oauth),
#    so that feature-gated code is compiled and its dependencies are in the offline cache.
RUN cargo build --locked --all-features
RUN cargo test --locked --all-features --no-run
# The deployed artifact is a release build; keep it available to confirm reproducers on it.
RUN cargo build --locked --release --all-features \
    || echo "WARNING: release build failed"

# 5. Smoke tests (the multi-minute stress/spike/iperf benchmarks are only compiled, above).
RUN (cargo test --locked --all-features --lib \
     && cargo test --locked --all-features --test config \
     && cargo test --locked --all-features --test simple_latency) \
    || echo "WARNING: smoke tests reported failures"

# Lab files. This Dockerfile lives in anthropics/oss-scanner, not in the NBD repository, so
# they cannot be copied from the checkout: they are written here (decoded source of the
# sender: a small Python script that sends crafted UDP datagrams to a multicast group).
RUN mkdir -p /opt/nbd-lab \
    && printf '%s\n' \
        '[nbd]' \
        'socket_buffer_size = 1048576' \
        'verbosity = "info"' \
        '' \
        '[kafka]' \
        'broker = "127.0.0.1:9092"' \
        'connection_timeout = 2000' \
        'message_timeout = 2000' \
        'message_retries = 2' \
        '' \
        '[metrics]' \
        'interface = "127.0.0.1"' \
        'port = 9000' \
        '' \
        '[[provider]]' \
        'topic = "lab-feed"' \
        'group = "239.255.0.1"' \
        'port = 20001' \
        'message_size = 1500' \
        'interface = "127.0.0.1"' \
        > /opt/nbd-lab/config.toml \
    && echo "IyEvdXNyL2Jpbi9lbnYgcHl0aG9uMwoiIiJTZW5kIGNyYWZ0ZWQgVURQIGRhdGFncmFtcyB0byBhIG11bHRpY2FzdCBncm91cCAodG8gZXhlcmNpc2UgTkJEIG9mZmxpbmUpLgoKRXhhbXBsZXMgKGxvb3BiYWNrIGxhYiwgc2VlIC5vc3Mtc2Nhbm5lci90aHJlYXRfbW9kZWwubWQpOgogIHNlbmRfbXVsdGljYXN0LnB5IC0tY291bnQgMTAwMCAtLXNpemUgMTQwMAogIHNlbmRfbXVsdGljYXN0LnB5IC0tc2l6ZSAwICAgICAgICAgICAgICAgICAgICAjIGVtcHR5IGRhdGFncmFtCiAgc2VuZF9tdWx0aWNhc3QucHkgLS1zaXplIDY1NTA3IC0tcGF0dGVybiB6ZXJvcwogIHNlbmRfbXVsdGljYXN0LnB5IC0tZmlsZSBwYXlsb2FkLmJpbiAtLWNvdW50IDUwCiIiIgppbXBvcnQgYXJncGFyc2UKaW1wb3J0IG9zCmltcG9ydCBzb2NrZXQKaW1wb3J0IHN5cwppbXBvcnQgdGltZQoKTUFYX1VEUF9QQVlMT0FEID0gNjU1MDcKCgpkZWYgYnVpbGRfcGF5bG9hZChhcmdzOiBhcmdwYXJzZS5OYW1lc3BhY2UsIHNlcTogaW50KSAtPiBieXRlczoKICAgIGlmIGFyZ3MuZmlsZToKICAgICAgICB3aXRoIG9wZW4oYXJncy5maWxlLCAicmIiKSBhcyBmaDoKICAgICAgICAgICAgcmV0dXJuIGZoLnJlYWQoKQogICAgaWYgYXJncy5wYXR0ZXJuID09ICJ6ZXJvcyI6CiAgICAgICAgcmV0dXJuIGJ5dGVzKGFyZ3Muc2l6ZSkKICAgIGlmIGFyZ3MucGF0dGVybiA9PSAiaW5jciI6CiAgICAgICAgcmV0dXJuIGJ5dGVzKChzZXEgKyBpKSAmIDB4RkYgZm9yIGkgaW4gcmFuZ2UoYXJncy5zaXplKSkKICAgIHJldHVybiBvcy51cmFuZG9tKGFyZ3Muc2l6ZSkKCgpkZWYgbWFpbigpIC0+IGludDoKICAgIHAgPSBhcmdwYXJzZS5Bcmd1bWVudFBhcnNlcihkZXNjcmlwdGlvbj1fX2RvY19fLCBmb3JtYXR0ZXJfY2xhc3M9YXJncGFyc2UuUmF3RGVzY3JpcHRpb25IZWxwRm9ybWF0dGVyKQogICAgcC5hZGRfYXJndW1lbnQoIi0tZ3JvdXAiLCBkZWZhdWx0PSIyMzkuMjU1LjAuMSIsIGhlbHA9Im11bHRpY2FzdCBncm91cCAoZGVmYXVsdDogJShkZWZhdWx0KXMpIikKICAgIHAuYWRkX2FyZ3VtZW50KCItLXBvcnQiLCB0eXBlPWludCwgZGVmYXVsdD0yMDAwMSwgaGVscD0iVURQIHBvcnQgKGRlZmF1bHQ6ICUoZGVmYXVsdClzKSIpCiAgICBwLmFkZF9hcmd1bWVudCgiLS1pZmFjZSIsIGRlZmF1bHQ9IjEyNy4wLjAuMSIsIGhlbHA9ImxvY2FsIElQdjQgYWRkcmVzcyBvZiB0aGUgc2VuZGluZyBpbnRlcmZhY2UgKGRlZmF1bHQ6ICUoZGVmYXVsdClzKSIpCiAgICBwLmFkZF9hcmd1bWVudCgiLS1zaXplIiwgdHlwZT1pbnQsIGRlZmF1bHQ9NjQsIGhlbHA9InBheWxvYWQgc2l6ZSBpbiBieXRlcywgMC4uJWQgKGRlZmF1bHQ6ICUlKGRlZmF1bHQpcykiICUgTUFYX1VEUF9QQVlMT0FEKQogICAgcC5hZGRfYXJndW1lbnQoIi0tY291bnQiLCB0eXBlPWludCwgZGVmYXVsdD0xLCBoZWxwPSJudW1iZXIgb2YgZGF0YWdyYW1zIChkZWZhdWx0OiAlKGRlZmF1bHQpcykiKQogICAgcC5hZGRfYXJndW1lbnQoIi0taW50ZXJ2YWwiLCB0eXBlPWZsb2F0LCBkZWZhdWx0PTAuMCwgaGVscD0ic2Vjb25kcyBiZXR3ZWVuIGRhdGFncmFtcyAoZGVmYXVsdDogJShkZWZhdWx0KXMpIikKICAgIHAuYWRkX2FyZ3VtZW50KCItLXBhdHRlcm4iLCBjaG9pY2VzPVsicmFuZG9tIiwgInplcm9zIiwgImluY3IiXSwgZGVmYXVsdD0icmFuZG9tIikKICAgIHAuYWRkX2FyZ3VtZW50KCItLWZpbGUiLCBoZWxwPSJzZW5kIHRoZSBjb250ZW50IG9mIEZJTEUgYXMgdGhlIGRhdGFncmFtIHBheWxvYWQgKG92ZXJyaWRlcyAtLXNpemUvLS1wYXR0ZXJuKSIpCiAgICBhcmdzID0gcC5wYXJzZV9hcmdzKCkKCiAgICBpZiBub3QgMCA8PSBhcmdzLnNpemUgPD0gTUFYX1VEUF9QQVlMT0FEOgogICAgICAgIHAuZXJyb3IoIi0tc2l6ZSBtdXN0IGJlIGJldHdlZW4gMCBhbmQgJWQiICUgTUFYX1VEUF9QQVlMT0FEKQoKICAgIHNvY2sgPSBzb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULCBzb2NrZXQuU09DS19ER1JBTSwgc29ja2V0LklQUFJPVE9fVURQKQogICAgc29jay5zZXRzb2Nrb3B0KHNvY2tldC5JUFBST1RPX0lQLCBzb2NrZXQuSVBfTVVMVElDQVNUX1RUTCwgMSkKICAgIHNvY2suc2V0c29ja29wdChzb2NrZXQuSVBQUk9UT19JUCwgc29ja2V0LklQX01VTFRJQ0FTVF9MT09QLCAxKQogICAgc29jay5zZXRzb2Nrb3B0KHNvY2tldC5JUFBST1RPX0lQLCBzb2NrZXQuSVBfTVVMVElDQVNUX0lGLCBzb2NrZXQuaW5ldF9hdG9uKGFyZ3MuaWZhY2UpKQoKICAgIHNlbnQgPSAwCiAgICBmb3Igc2VxIGluIHJhbmdlKGFyZ3MuY291bnQpOgogICAgICAgIHNvY2suc2VuZHRvKGJ1aWxkX3BheWxvYWQoYXJncywgc2VxKSwgKGFyZ3MuZ3JvdXAsIGFyZ3MucG9ydCkpCiAgICAgICAgc2VudCArPSAxCiAgICAgICAgaWYgYXJncy5pbnRlcnZhbCA+IDA6CiAgICAgICAgICAgIHRpbWUuc2xlZXAoYXJncy5pbnRlcnZhbCkKICAgIHByaW50KCJzZW50ICVkIGRhdGFncmFtKHMpIHRvICVzOiVkIHZpYSAlcyIgJSAoc2VudCwgYXJncy5ncm91cCwgYXJncy5wb3J0LCBhcmdzLmlmYWNlKSwgZmlsZT1zeXMuc3RkZXJyKQogICAgcmV0dXJuIDAKCgppZiBfX25hbWVfXyA9PSAiX19tYWluX18iOgogICAgc3lzLmV4aXQobWFpbigpKQo=" | base64 -d > /opt/nbd-lab/send_multicast.py \
    && chmod +x /opt/nbd-lab/send_multicast.py

# The lab configuration must stay valid.
RUN ./target/debug/nbd --check-config-file /opt/nbd-lab/config.toml \
    || echo "WARNING: lab configuration rejected by nbd --check-config-file"

# 6. Dependency and supply-chain tooling (needs a recent toolchain: use "stable") ------------
RUN rustup toolchain install stable --profile minimal \
    || echo "WARNING: stable toolchain not installed"
RUN cargo +stable install --locked cargo-audit \
    || echo "WARNING: cargo-audit not installed"
RUN cargo +stable install --locked cargo-deny \
    || echo "WARNING: cargo-deny not installed"
# Advisory databases are downloaded now so that both tools work offline afterwards.
# cargo-audit has no separate "fetch" subcommand: it downloads its database on a normal run
# (its exit status is non-zero when advisories are found, hence the `; true`).
RUN (cargo audit; true) && test -d /opt/cargo/advisory-db/crates \
    || echo "WARNING: cargo-audit advisory database not fetched"
RUN cargo deny fetch all || (cargo deny check advisories; true) \
    || echo "WARNING: cargo-deny advisory database not fetched"
RUN (cargo audit --no-fetch; cargo deny --offline check advisories) \
    || echo "WARNING: offline dependency checks did not run"

# 7. Fuzzing tooling: cargo-fuzz + nightly, and a warm-up that caches libfuzzer-sys and
#    arbitrary in the cargo registry so `cargo fuzz init/run` works without network.
RUN cargo +stable install --locked cargo-fuzz \
    || echo "WARNING: cargo-fuzz not installed"
RUN rustup toolchain install nightly --profile minimal \
    && mkdir -p /opt/fuzz-warmup && cd /opt/fuzz-warmup \
    && cargo +stable init --lib --name fuzz_warmup . \
    && cargo +stable add libfuzzer-sys \
    && cargo +stable add arbitrary --features derive \
    && cargo +nightly fetch \
    || echo "WARNING: fuzzing warm-up failed"

# 8. OPTIONAL, disabled by default: a local Apache Kafka broker (KRaft) to test the real sink.
#    Enable only after pinning a version and verifying its published SHA-512 checksum.
# ARG KAFKA_VERSION=<pinned version>
# ARG KAFKA_SHA512=<expected SHA-512 of the tarball, copied from the Apache release page>
# RUN apt-get update && apt-get install -y --no-install-recommends default-jre-headless \
#     && rm -rf /var/lib/apt/lists/*
# RUN curl -fsSLO "https://archive.apache.org/dist/kafka/${KAFKA_VERSION}/kafka_2.13-${KAFKA_VERSION}.tgz" \
#     && echo "${KAFKA_SHA512}  kafka_2.13-${KAFKA_VERSION}.tgz" | sha512sum -c - \
#     && tar -xzf "kafka_2.13-${KAFKA_VERSION}.tgz" -C /opt

# From here on cargo must never try to reach the network (the audit runs offline).
ENV CARGO_NET_OFFLINE=true