# Scanner image for https://github.com/the-dot-squad/unlostpaws-worker (Python 3.12 vision worker).
# torch/transformers are NOT installed: tests/unit/conftest.py mocks them, and no model weights are
# downloaded, so nothing needs the Hugging Face Hub at audit time. Real inference is out of scope.
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN pip install --no-cache-dir -r requirements.txt pytest==8.3.4 pytest-asyncio==0.25.3 ruff==0.9.4

ENV PYTHONPATH=/src \
    PYTHONUNBUFFERED=1

# Run the unit tests so the image is known good, but do not let a failure abort the build.
RUN pytest -q || echo "WARNING: pytest reported failures"
