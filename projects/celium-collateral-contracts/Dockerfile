# Builds Datura-ai/celium-collateral-contracts: the Solidity contract with Foundry, and the Python
# tooling (celium_collateral_contracts/) from pdm.lock. forge-std and the solc binary are fetched here,
# because the audit runs offline.
FROM debian:bookworm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git jq python3 python3-venv \
    && rm -rf /var/lib/apt/lists/*

# Foundry (forge, cast, anvil, chisel), pinned.
ARG FOUNDRY_VERSION=v1.8.3
RUN arch="$(dpkg --print-architecture)" \
    && curl -fsSL "https://github.com/foundry-rs/foundry/releases/download/${FOUNDRY_VERSION}/foundry_${FOUNDRY_VERSION}_linux_${arch}.tar.gz" \
       | tar -xz -C /usr/local/bin \
    && forge --version

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Pin solc so forge does not try to download a newer one while offline.
ENV HOME=/root \
    FOUNDRY_SOLC_VERSION=0.8.30 \
    FOUNDRY_OFFLINE=true \
    PATH=/opt/venv/bin:$PATH

# forge-std is listed in .gitmodules but is not a committed submodule, so clone the version foundry.toml names.
RUN rm -rf libs/forge-std \
    && git clone --depth 1 --branch v1.9.6 https://github.com/foundry-rs/forge-std libs/forge-std

RUN FOUNDRY_OFFLINE=false forge build && forge build --force

# Python tooling, with the versions from pdm.lock.
RUN python3 -m venv /opt/venv \
    && pip install --no-cache-dir pdm==2.29.2 \
    && PDM_PYTHON=/opt/venv/bin/python pdm export --prod --no-hashes -o /tmp/requirements.lock \
    && pip install --no-cache-dir -r /tmp/requirements.lock pytest==9.1.1 \
    && pip install --no-cache-dir --no-deps -e /src

# Cheap tests. unittests/test_collateral_flow.py needs a live subtensor node, so it is not run here.
RUN forge test -vv || echo "WARNING: forge test reported failures"
RUN pytest -q unittests/test_address_conversion.py || echo "WARNING: pytest reported failures"
