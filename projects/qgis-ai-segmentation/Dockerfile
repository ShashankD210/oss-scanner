# QGIS plugin: needs QGIS's Python bindings plus the packages the plugin installs into its own venv.
FROM qgis/qgis:ltr
ENV DEBIAN_FRONTEND=noninteractive \
    QT_QPA_PLATFORM=offscreen \
    PIP_BREAK_SYSTEM_PACKAGES=1
RUN apt-get update && apt-get install -y --no-install-recommends python3-pip git \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Runtime packages the plugin installs at first run (src/core/venv_manager.py). CPU torch keeps the image small.
RUN pip3 install --no-cache-dir --index-url https://download.pytorch.org/whl/cpu "torch>=2.5.1,<3" "torchvision>=0.20.1,<1" \
    && pip3 install --no-cache-dir "numpy>=1.26,<2" "rasterio>=1.3,<2" "sam2>=1.0,<2" defusedxml pytest

# Make the plugin importable the way QGIS loads it.
RUN mkdir -p /root/.local/share/QGIS/QGIS3/profiles/default/python/plugins \
    && ln -s /src /root/.local/share/QGIS/QGIS3/profiles/default/python/plugins/AI_Segmentation \
    && (python3 -c "import qgis.core, sys; sys.path.insert(0, '/src'); import src.core" || echo "WARNING: import check failed")
