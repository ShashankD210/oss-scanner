FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
RUN pip install --no-cache-dir -r requirements.txt pytest
RUN python -m pytest -q || echo "WARNING: pytest reported failures"
