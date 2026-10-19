# ViperCapture for the OSS Scanner.
#
# The scanner builds this file with the ViperCapture checkout as the context, then
# runs with no network. Everything a later test needs is fetched here:
#   - Python dependencies from requirements.txt (there is no pyproject or extra
#     requirements file; tests import pytest, which is not pinned upstream, so
#     pytest is installed too)
#   - Playwright's Chromium, Firefox, and WebKit, plus their system libraries
#   - ffmpeg, including the libvpx, libvpx-vp9, and libx264 encoders video output uses
#   - the frontend, installed and built the way .github/workflows/validate.yml does
#
# /src is the checkout. Browsers live in PLAYWRIGHT_BROWSERS_PATH.
FROM python:3.12-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PLAYWRIGHT_BROWSERS_PATH=/ms-playwright \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Node 22 is the version validate.yml installs with actions/setup-node.
COPY --from=node:22-bookworm-slim /usr/local/bin/node /usr/local/bin/node
COPY --from=node:22-bookworm-slim /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN ln -sf ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -sf ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
    && node --version && npm --version

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git ffmpeg \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# --with-deps installs the shared libraries the browsers need. The project's
# production image passes --no-shell, which skips Playwright's Chromium headless
# shell. tests/test_diagnostic_har.py launches the default Chromium (that shell)
# against a local page, and the README verification command installs it, so this
# image does too. playwright re-runs apt-get, so this step needs the network.
RUN pip install --no-cache-dir -r requirements.txt pytest \
    && python -m playwright install --with-deps chromium firefox webkit \
    && ffmpeg -hide_banner -encoders | grep -E 'libvpx|libx264'

# Frontend: npm ci, then `npm run build` (tsc + vite). Output is written to static/app.
RUN npm ci --prefix frontend --no-audit --no-fund \
    && npm run build --prefix frontend \
    && test -s static/app/index.html \
    && npm cache clean --force

# The Python checks validate.yml runs on Linux. A failing test is reported and does
# not abort the image build.
RUN python -m compileall -q vipercapture launch.py scripts/smoke.py \
    && (python -m unittest tests.test_launch_installer tests.test_capture_cli \
        || echo "WARNING: unittest reported failures")
