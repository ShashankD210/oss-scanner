FROM python:3.12-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates git \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
RUN python -m pip install --no-cache-dir -r requirements-test.txt
RUN python -m unittest discover -s tests -v \
    && python -m compileall -q src tools tests touchdesigner
