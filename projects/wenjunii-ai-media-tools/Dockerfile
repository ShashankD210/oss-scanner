FROM node:24-bookworm-slim AS node
FROM python:3.12-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates git libstdc++6 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY . /src
WORKDIR /src
RUN python -m pip install --no-cache-dir .
RUN python -m unittest discover -s tests -v \
    && node --test tests/library.test.cjs \
    && python -m media_scout verify-public-archive
