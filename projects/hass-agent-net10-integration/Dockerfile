# HASS.Agent .NET10 integration: a Home Assistant custom integration (Python, custom_components/hass_agent).
# Its tests run on Home Assistant itself through pytest-homeassistant-custom-component, which pins the Home
# Assistant version and needs the Python that Home Assistant uses (3.14).
FROM python:3.14-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONDONTWRITEBYTECODE=1
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential git \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Home Assistant and its test helpers, and the linter CI uses. Everything is fetched here; the audit runs offline.
RUN python -m pip install -r requirements_test.txt ruff==0.16.1

# The integration is plain Python: nothing to compile. Run the suite (about a hundred tests, a few seconds) so a
# broken image shows; a failing test does not stop the build.
RUN (python -m pytest -q -p no:cacheprovider || echo "WARNING: pytest reported failures") \
    && (ruff check custom_components/hass_agent || echo "WARNING: ruff reported findings")
