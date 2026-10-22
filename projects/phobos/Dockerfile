FROM ghcr.io/astral-sh/uv:0.12.13 AS uv
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash build-essential ca-certificates curl git openssl python3 \
      python3-venv ripgrep shellcheck socat xz-utils \
    && rm -rf /var/lib/apt/lists/*
COPY --from=uv /uv /usr/local/bin/uv
COPY . /src
WORKDIR /src
RUN GITHUB_PATH=/opt/zig-path scripts/setup-ci-zig.sh
ENV PATH="/src/.scratch/toolchains/zig-x86_64-linux-0.17.0:${PATH}" \
    UV_PYTHON_DOWNLOADS=never PYTHONPATH=/src/tools
RUN uv sync --locked
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
RUN --network=none zig build -j2 \
    && scripts/test.sh Debug \
    && scripts/test.sh ReleaseSafe
RUN --network=none uv run --locked python -m pytest \
      tools/api_scanner/tests tools/client_api_scanner/tests \
      tools/perf_harness/tests tools/release/tests --basetemp=.scratch/scanner-pytest \
    && uv run --locked ruff check tools/perf_harness/plot_perf.py tools/perf_harness/report.py tools/perf_harness/tests tools/release \
    && uv run --locked black --check tools/perf_harness/plot_perf.py tools/perf_harness/report.py tools/perf_harness/tests tools/release \
    && uv run --locked mypy --explicit-package-bases --ignore-missing-imports tools/perf_harness/plot_perf.py tools/perf_harness/report.py tools/perf_harness/tests/test_plot_perf.py tools/release \
    && zig fmt --check build.zig build.zig.zon src tests/fixtures/float_abi_probe.zig \
    && zig test -target native-native-musl build.zig \
    && shellcheck scripts/build-release.sh scripts/setup-ci-zig.sh scripts/test.sh tools/perf_harness/*.sh tools/perf_harness/tests/*.sh \
    && (cd vendor/sqlite && sha256sum --check SHA256SUMS)
RUN --network=none zig-out/bin/phobos -d /src/.scratch/scanner-data --dump-api > .scratch/scanner-api.json \
    && uv run --locked python -m api_scanner coverage --baseline api/api_surface.json --phobos .scratch/scanner-api.json \
    && uv run --locked python -m api_scanner table --phobos .scratch/scanner-api.json --check
