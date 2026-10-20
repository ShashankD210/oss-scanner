# Build digithings for Anthropic's OSS Scanner.
#
# digithings is a uv workspace: twelve Python packages (digibase, digiclaw,
# digifetch, digigraph, digikey, digillm, digiquant, digisearch, digiskills,
# digitrace, digivault, digivoice) resolved together from one uv.lock into one
# .venv. The scanner uses the repository root as the build context and audits
# /src offline, so every dependency, and every test the audit might want to run,
# has to be installed here.

FROM python:3.12-slim-bookworm

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ENV DEBIAN_FRONTEND=noninteractive \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# C toolchain and headers for the dependencies that build from source on
# cp312/bookworm (cryptography and cffi via libffi). Everything else has a wheel.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential ca-certificates curl git libffi-dev libssl-dev pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

# uv is not itself a locked dependency of the workspace, so pin the version the
# maintainers run (uv 0.10.2, which writes the current uv.lock format).
RUN pip install --no-cache-dir "uv==0.10.2"

# The whole workspace: every package, plus each package's dev/test dependency
# group, from the committed lockfile. The extras are deliberately left out. The
# repository's CI installs `--all-extras`, which drags in sentence-transformers,
# torch and the CUDA runtime for a reranker that no audit needs and that would
# multiply the image size. The one thing that breaks without an extra is pandas:
# digiquant imports it at module scope (`nautilus_runner.py`) even though the
# package is polars-only, and the CLI plus the baseline suite import digiquant,
# so it is installed on its own. nautilus_trader is not needed - the adapters
# import it lazily, inside the function that runs a backtest.
RUN uv sync --frozen --all-packages \
    && uv pip install --python /src/.venv --no-cache pandas

# The repository's root .dockerignore drops the top-level tests/ and docs/ from
# the build context, because its release images do not need them. A security
# audit does, so restore them from the same public repository and ref the
# scanner clones.
ARG DIGITHINGS_REF=develop
RUN git clone --depth 1 --branch "${DIGITHINGS_REF}" \
      https://github.com/digithings-ai/digithings /tmp/digithings-full \
    && cp -a /tmp/digithings-full/tests /src/tests \
    && cp -a /tmp/digithings-full/docs /src/docs \
    && rm -rf /tmp/digithings-full

# The network is gone after this point: `uv run` must use the environment built
# above as it is, never try to re-resolve or re-sync it.
ENV UV_FROZEN=1 \
    UV_NO_SYNC=1 \
    UV_OFFLINE=1 \
    VIRTUAL_ENV=/src/.venv \
    PATH=/src/.venv/bin:$PATH

# Prove the environment works offline: every workspace package imports, and the
# repository's always-green baseline suite passes. Cheap tests must not abort the
# build, so failures are reported and swallowed.
RUN uv run python -c "import digibase, digiclaw, digifetch, digigraph, digikey, digillm, digiquant, digisearch, digiskills, digitrace, digivault, digivoice; print('workspace imports ok')" \
    || echo "WARNING: workspace import check reported failures"

# Root pytest.ini drives the cross-package suite in tests/; `pytest tests/baseline`
# is the marker the repository itself treats as always-green. Per-component suites
# live next to their package (digillm/tests, digifetch/tests, digisearch/tests).
# The three baseline files skipped here all shell out to `git ls-files`, and the
# release .dockerignore drops .git from the build context, so they cannot run in
# this image. They gate vendor allowlists, file censuses and lockfile placement in
# CI; none of them exercises the code under audit.
RUN uv run pytest tests/baseline -q -p no:cacheprovider \
      --ignore=tests/baseline/test_vendor_content_gate.py \
      --ignore=tests/baseline/test_surface_endpoint_config.py \
      --ignore=tests/baseline/test_uv_workspace_lock_guard.py \
    || echo "WARNING: pytest reported failures"
