# Tracekit: a Python package (tracekit-ai) with a TypeScript verifier and SDK in sdk/typescript.
FROM python:3.12-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONFAULTHANDLER=1
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential git pkg-config libre2-dev nodejs npm \
    && rm -rf /var/lib/apt/lists/*

# The git witness tests make commits, so git needs an identity.
RUN git config --system user.name "Tracekit tests" \
    && git config --system user.email tests@tracekit.invalid

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Install the package with every optional extra and the test dependencies, as CI does, plus the contrib packages,
# so the whole suite can run offline.
RUN python -m pip install -e ".[dev,yaml,langchain,grpc,postgres,aws,why-anthropic]" \
    && python -m pip install ./contrib/*/

# The TypeScript verifier and SDK: fetch its dependencies now, while the network is up.
RUN cd sdk/typescript && (npm install --no-audit --no-fund && npm run build --if-present \
    || echo "WARNING: sdk/typescript did not build")

# Run the tests so you know the image works; a failing test does not abort the build.
RUN timeout 1500 python -m pytest -q --no-header -p no:cacheprovider -o faulthandler_timeout=240 \
      || echo "WARNING: pytest reported failures"
