# Build container for the oss-scanner audit of local-ai-mcp-servers.
# Build phase has internet (pip); the audit itself runs WITHOUT network, so
# everything needed is installed here. Both servers share the same two pinned
# dependencies (mcp, httpx); everything else is standard library.
FROM python:3.12-slim

WORKDIR /src
COPY . /src

RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir -r mcp-modelmanager/requirements.txt

# Byte-compile every module of both servers (catches syntax/import-level breakage).
RUN /opt/venv/bin/python -m compileall -q mcp-modelmanager mcp-ollama-vllm

# Offline self-test of the model manager: allowlist battery, catalog/template
# substitution, control-channel protocol, VM-side wrapper against a real
# SSH_ORIGINAL_COMMAND. Live groups require --live and are deliberately not run;
# parts needing operator setup scripts skip cleanly. Runs as a non-root user:
# one test asserts that the wrapper refuses a foreign-owned lib/, and that
# ownership distinction does not exist for root.
RUN useradd -m scan && cp -r /src /home/scan/src && chown -R scan:scan /home/scan/src
RUN cd /home/scan/src/mcp-modelmanager && su scan -c "/opt/venv/bin/python selftest.py"

# mcp-ollama-vllm's selftest.py needs a running Ollama/vLLM backend and is
# therefore not run at build time; its code is covered by compileall above and
# by the audit itself.
