# Threat model: AI Agent (QGIS plugin)

SECURITY.md in the repository is the authoritative statement of what we claim; read it first. Summary below.

## What this project does and where untrusted input enters
A QGIS plugin (about 9,000 downloads on plugins.qgis.org since its September 2026 release) that lets an AI model choose and run GIS tools inside the user's QGIS, with the user's permissions. Treat as untrusted:
- Everything the model sends back (tool names and arguments), including what a prompt injection could make it send.
- Text fetched from the web or read from layers, attribute tables, file names and project files, which reaches the model as data.
- HTTP responses from the TerraLab backend and the websocket wire protocol (src/core/protocol.py, src/core/ws_protocol.py).
- URLs passed to fetch tools and their redirects.

## Components that matter most / least
- Most: the danger levels and guards that must fail closed (src/tools/danger.py, src/tools/guards.py, src/core/executor*.py), path checks (src/core/security.py validate_path and validate_read, src/core/code_tripwire.py, src/core/code_guard*.py), the URL guard (src/core/net_opener.py check_url), key scrubbing (src/core/log_scrub.py), untrusted-text labelling (src/core/serialization.py), file links in answers (src/ui/file_links.py SAFE_TO_OPEN), undo (src/core/snapshot.py, src/core/checkpoints.py).
- Less: layout, styling and translations.
- Out of scope: the TerraLab backend (not in this repository), QGIS core, and the model giving a wrong answer.

## How to exercise it
QGIS Python bindings are installed in the image (`python3 -c "import qgis.core"`, run headless with QT_QPA_PLATFORM=offscreen). The plugin source is at /src and is linked into the QGIS profile as QGIS_AI-Agent. Tool specs live in src/tools/.

## How you rate severity
- Critical: a write, read, network fetch or code execution that a guard should have stopped, reachable from model output or fetched content without the user approving it; the activation key reaching the model, a result or a log.
- High: a write outside the allowed paths, a URL guard bypass (SSRF to local addresses, redirect escapes), a file link that launches a program, an undo that silently fails.
- Medium: denial of service of QGIS from crafted input.
- Low: anything that needs attacker code already running in QGIS.

## Anything to leave alone
- execute_code runs arbitrary Python once the user approves it: that is by design, not a finding. A way to run it without that approval is a finding.
- Hardening against a hostile sibling plugin: QGIS plugins share one Python process by design.
