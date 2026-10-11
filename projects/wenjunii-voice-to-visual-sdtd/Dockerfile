FROM python:3.11-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    HF_HUB_OFFLINE=1 \
    TRANSFORMERS_OFFLINE=1
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates git \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
# Matches CI: hardware/model runtimes are mocked in this portable test profile.
RUN python -m pip install --no-cache-dir -r requirements-test.txt
RUN python -m unittest discover -s tests -v
