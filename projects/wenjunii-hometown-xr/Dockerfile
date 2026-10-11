FROM python:3.11-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    HF_HUB_OFFLINE=1 \
    TRANSFORMERS_OFFLINE=1 \
    AWS_EC2_METADATA_DISABLED=true
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential ca-certificates git libgomp1 \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
# Use the declared PyTorch version with CPU wheels for the scanner's Linux VM.
RUN python -m pip install --no-cache-dir 'torch==2.1.0' \
    --index-url https://download.pytorch.org/whl/cpu
RUN python -m pip install --no-cache-dir -r requirements-test.txt -e .
# Keep dependency-risk review separate from building the code to audit. The
# current repository's review deadline is overdue; do not renew it in this image.
RUN python -m pytest \
    && python main.py --help
