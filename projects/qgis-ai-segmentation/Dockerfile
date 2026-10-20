# QGIS plugin: needs QGIS's Python bindings plus the packages the plugin installs into its own venv.
FROM qgis/qgis:ltr
ENV DEBIAN_FRONTEND=noninteractive \
    QT_QPA_PLATFORM=offscreen
RUN apt-get update && apt-get install -y --no-install-recommends python3-pip python3-venv git \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# The plugin installs its runtime packages into an isolated venv at first run (src/core/venv_manager.py).
# Mirror that here: a venv that still sees QGIS's bindings, with CPU torch to keep the image small.
RUN python3 -m venv --system-site-packages /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir --index-url https://download.pytorch.org/whl/cpu "torch>=2.5.1,<3" "torchvision>=0.20.1,<1" \
    && /opt/venv/bin/pip install --no-cache-dir "numpy>=1.26,<2" "rasterio>=1.3,<2" "sam2>=1.0,<2" defusedxml pytest
ENV PATH=/opt/venv/bin:$PATH

# Make the plugin importable the way QGIS loads it.
RUN mkdir -p /root/.local/share/QGIS/QGIS3/profiles/default/python/plugins \
    && ln -s /src /root/.local/share/QGIS/QGIS3/profiles/default/python/plugins/AI_Segmentation \
    && (python3 -c "import qgis.core, sys; sys.path.insert(0, '/src'); import src.core" || echo "WARNING: import check failed")
