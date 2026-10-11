FROM python:3.11-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/src/src \
    PIP_DISABLE_PIP_VERSION_CHECK=1
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates git \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
RUN python -m pip install --no-cache-dir .
RUN python -m unittest discover -v \
    && python tools/verify_repository.py \
    && python -m compileall -q src tools touchdesigner/extensions
