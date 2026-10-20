# Threat model: AI Segmentation (QGIS plugin)

## What this project does and where untrusted input enters
A QGIS plugin (about 72,000 downloads on plugins.qgis.org) that segments objects in rasters with SAM models, locally or through the TerraLab cloud API. It runs inside the user's QGIS process with their file system access. Treat as untrusted:
- Raster and vector files the user opens (GeoTIFF, VRT, remote COG and XYZ/WMS tiles), including file names and layer metadata.
- Every HTTP response: TerraLab API replies (api.terra-lab.ai), model checkpoint downloads, the standalone Python and pip downloads done by the venv bootstrap.
- Archives the plugin extracts (src/core/archive_utils.py, src/core/streamed_download.py): path traversal, symlinks, size bombs.
- Model checkpoints loaded with torch (pickle) and anything that reaches subprocess or the isolated venv worker (src/core/venv_*.py, src/workers/).
- Requests reaching the local agent API (src/agent_bridge.py, src/mcp_api*.py) from other local processes or AI agents, including prompt-injected text from layer content.

## Components that matter most / least
- Most: downloads and extraction, venv bootstrap and subprocess calls, the agent bridge and MCP API, auth token handling and storage (account and activation code).
- Less: UI widgets, styling, translations in i18n/.
- Out of scope: the TerraLab backend itself (not in this repository), QGIS core, and third-party packages (torch, sam2, rasterio).

## How to exercise it
QGIS Python bindings are installed in the image (`python3 -c "import qgis.core"`, run headless with QT_QPA_PLATFORM=offscreen). The plugin source is at /src and is linked into the QGIS profile as AI_Segmentation.

## How you rate severity
- Critical: code execution or file write outside the plugin folder from a crafted raster, archive, download or API reply; theft of the user's auth token.
- High: path traversal or arbitrary file read, bypass of the agent API guard (src/mcp_api_guard.py), quota bypass that a remote party can trigger.
- Medium: denial of service of QGIS from a crafted input.
- Low: anything that needs the attacker to already control the user's machine.

## Anything to leave alone
Do not report that the plugin calls the network or installs packages at first run: that is by design and shown to the user.
