# BRiSKapi: a free-threaded Python package (3.14t, as CI runs it), two Rust programs, and Node decoder hosts. The scanner
# builds this with the repository as the build context and the checkout at /src.
FROM debian:trixie-slim
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential pkg-config git \
    && rm -rf /var/lib/apt/lists/*

# Node 22 (the decoder hosts and their tests), the Rust toolchain, and uv (to install the free-threaded Python, which
# the official Python images do not ship), copied from their official images.
COPY --from=node:22-trixie-slim /usr/local/bin/node /usr/local/bin/node
COPY --from=rust:1-slim-trixie /usr/local/cargo /usr/local/cargo
COPY --from=rust:1-slim-trixie /usr/local/rustup /usr/local/rustup
COPY --from=ghcr.io/astral-sh/uv:0.13 /uv /usr/local/bin/uv
ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    UV_PYTHON_INSTALL_DIR=/opt/python \
    UV_LINK_MODE=copy \
    VIRTUAL_ENV=/opt/venv \
    PATH=/opt/venv/bin:/usr/local/cargo/bin:$PATH \
    RUSTFLAGS="-C force-frame-pointers=yes" \
    CARGO_PROFILE_RELEASE_DEBUG=true \
    CARGO_PROFILE_DEV_DEBUG=true \
    BRISK_MOCK_CACHE=/src/cache/brisk-mock

COPY . /src
WORKDIR /src

# The Rust programs, built as CI builds them (the tests run brisk-parquet from rust/brisk_parquet/target/release),
# and their test builds, so that the crates are fetched now and the audit needs no network.
RUN cargo build --release --locked --manifest-path rust/brisk_parquet/Cargo.toml \
    && cargo build --release --locked --manifest-path rust/brisk_quote_ingest/Cargo.toml \
    && cargo test --no-run --locked --manifest-path rust/brisk_parquet/Cargo.toml \
    && cargo test --no-run --locked --manifest-path rust/brisk_quote_ingest/Cargo.toml

# The package with its tests' dependencies (boto3 is the AWS SDK the cloud services run on; the tests use fakes).
RUN uv venv --python 3.14t /opt/venv \
    && uv pip install -r requirements-dev.txt -e . \
    && python -c "import sys, numpy, pyarrow, pandas, briskapi; assert not sys._is_gil_enabled(), 'the GIL is on'"

# The public demo's files that the decoder replays, pinned by SHA-256 in briskapi/decoder/assets.json and fetched
# from the demo site as CI does. They are not in the repository.
RUN python tools/brisk_mock/download_mock.py --cache "$BRISK_MOCK_CACHE"

# Run the tests so the image is known to work, but do not let a failing test abort the build.
RUN (timeout 1500 python -m pytest tests -q || echo "WARNING: pytest reported failures") \
    && (timeout 600 node --test --test-concurrency=1 tools/brisk_mock/*.test.cjs || echo "WARNING: node tests reported failures")
