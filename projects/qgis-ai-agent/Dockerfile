# QGIS plugin: pure Python on top of QGIS's own bindings, no extra packages at runtime.
FROM qgis/qgis:ltr
ENV DEBIAN_FRONTEND=noninteractive \
    QT_QPA_PLATFORM=offscreen \
    PIP_BREAK_SYSTEM_PACKAGES=1
RUN apt-get update && apt-get install -y --no-install-recommends python3-pip git \
    && rm -rf /var/lib/apt/lists/* \
    && pip3 install --no-cache-dir pytest

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Make the plugin importable the way QGIS loads it.
RUN mkdir -p /root/.local/share/QGIS/QGIS3/profiles/default/python/plugins \
    && ln -s /src /root/.local/share/QGIS/QGIS3/profiles/default/python/plugins/QGIS_AI-Agent \
    && (cd /src && python3 -c "import qgis.core; import src.core.security, src.core.net_opener, src.core.executor" || echo "WARNING: import check failed")
