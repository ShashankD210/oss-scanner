FROM debian:bookworm
RUN apt-get update && apt-get install -y --no-install-recommends shellcheck python3 git \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
RUN find scripts -name '*.sh' -print0 | xargs -0 -r -n1 bash -n
RUN shellcheck -S warning $(find scripts -name '*.sh') || echo "WARNING: shellcheck reported issues"
RUN python3 -m py_compile scripts/audit-modules.py
