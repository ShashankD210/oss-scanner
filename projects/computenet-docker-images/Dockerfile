# Analysis image for the Lium pod image sources. The real pod images are multi-GB CUDA builds, so we do not build
# them here. Instead we lay out the files they ship (/start.sh, the nginx proxy, the SSH server, Jupyter) on Ubuntu
# 24.04, the base most pod images use, so an agent can run the entrypoint and the proxy offline. shellcheck and
# hadolint are installed for static checks, and the repo's own script tests run at the end.
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git bash procps iproute2 netcat-openbsd jq file \
      python3 python3-pip python3-venv python3-pytest python3-aiohttp python3-boto3 \
      openssh-server openssh-client nginx shellcheck \
    && rm -rf /var/lib/apt/lists/*

# hadolint is not packaged for Ubuntu; take the release binary.
ARG HADOLINT_VERSION=v2.12.0
RUN arch="$(uname -m)"; case "$arch" in aarch64) arch=arm64;; esac; \
    curl -fsSL -o /usr/local/bin/hadolint \
      "https://github.com/hadolint/hadolint/releases/download/${HADOLINT_VERSION}/hadolint-Linux-${arch}" \
    && chmod +x /usr/local/bin/hadolint && hadolint --version

# Jupyter Lab, which start.sh starts when JUPYTER_PASSWORD is set.
RUN pip install --no-cache-dir --break-system-packages jupyterlab && jupyter lab --version

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Pod layout, as templates/ubuntu and the web-UI templates (vscode, comfyui, ...) build it.
RUN mkdir -p /workspace \
    && install -m 755 /src/scripts/start.sh /start.sh \
    && cp /src/scripts/proxy/nginx.conf /etc/nginx/nginx.conf \
    && cp /src/scripts/proxy/readme.html /usr/share/nginx/html/readme.html \
    && rm -f /etc/ssh/ssh_host_*

# Cheap checks. A failure is logged, not fatal.
RUN nginx -t || echo "WARNING: nginx -t failed"
RUN bash -n /start.sh || echo "WARNING: bash -n start.sh failed"
RUN find scripts templates -name '*.sh' -print0 | xargs -0 shellcheck -S warning \
    || echo "WARNING: shellcheck reported issues"
RUN find templates -name 'Dockerfile*' -print0 | xargs -0 hadolint --failure-threshold error \
    || echo "WARNING: hadolint reported issues"
RUN templates/pytorch/tests/test-entrypoint.sh || echo "WARNING: pytorch test-entrypoint failed"
RUN for t in templates/dolphin/tests/test_sidecar.py templates/dolphin/tests/test_watchdog.py \
             templates/pearl-miner/tests/test_sidecar.py templates/engy/tests/test_*.py; do \
      python3 "$t" || echo "WARNING: $t failed"; \
    done
RUN cd templates/lium-cluster && python3 -m pytest -q test_lium_fabric_env.py test_lium_rdma_runc.py \
    || echo "WARNING: lium-cluster pytest failed"
