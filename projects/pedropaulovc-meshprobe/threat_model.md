# Threat model

## Scope and inputs

MeshProbe is a small Python CLI and stdio MCP service for read-only inspection of GLB, glTF, OBJ and STL models through a persistent, factory-clean Blender worker. This enrollment makes no adoption claim. Treat model containers, geometry, materials, external texture/buffer references, environment images, scene names, CLI/MCP commands, and saved workspace/checkpoint/cache files as potentially adversarial. Source files should remain unchanged; generated renders and inspection artifacts are writable output, not trusted input.

Review `src/meshprobe/sources.py` for source-bundle identity and sidecar path handling; `controller.py` and `blender/` for subprocess/protocol/import/render behavior; `protocol.py`, `service.py`, `cli.py` and `mcp_server.py` for command validation; and `workspace.py`, `artifacts.py`, `daemon.py` and `client.py` for publication, recovery, path selection and authentication. The daemon binds loopback and stores a random token in workspace metadata with mode 0600. Loopback is not itself an authentication boundary. Local users able to read that token or modify the user's environment are not assumed isolated by the daemon.

## Security boundaries and priorities

Prioritize unintended source modification, reading or writing outside intended source/workspace boundaries, symlink/path traversal, command or Python execution through data, token disclosure/bypass, corruption of content-addressed artifacts, and unbounded memory/CPU/disk consumption from malformed scenes or command streams. Distinguish a supported user-selected source/workspace/Blender executable from an attacker-controlled reference inside a model or saved artifact. A factory-clean Blender process is not an OS sandbox for hostile importers. Report a reachable third-party importer defect with the exact component/version and distinguish it from MeshProbe's own logic.

The evaluation harness's public model, private truth, agent-visible artifacts, and credential isolation are separate boundaries worth reviewing. Do not mount real credentials, run hosted agents, download private data, deploy services, or contact account APIs. Use temporary local synthetic inputs only.

## Available environment and offline exercise

The image contains Python 3.12, the full `uv.lock` development environment, compiled Python bytecode, a built wheel under `/src/dist`, and the official checksum-verified Blender 5.2.0 LTS Linux x64 distribution. Mesa llvmpipe/EGL libraries support headless software-rendering paths. The editable source and tests remain at `/src`; dependencies and the build cache are retained, and uv defaults to offline mode. No tests run during image construction.

Run with Docker networking disabled. Ordinary source smoke does not need a public or private qualification corpus:

```sh
meshprobe --help
meshprobe schema
blender --background --factory-startup --python-expr "import bpy; assert bpy.app.version[:3] == (5, 2, 0); print(bpy.app.version_string)"
python -m pytest tests/integration/test_blender_smoke.py
```

For broader source coverage, run `python -m pytest`; examine skipped tests rather than interpreting them as passed coverage. Tests use generated/local fixtures where possible. The pinned `evals/manifests/public/smoke.json` tier and `tools/clean_install_smoke.py` are qualification workflows, not this ordinary Blender/source startup smoke.

## Explicit limitations

This is Linux amd64 only. No native Windows AppContainer/Job Object execution or Windows Blender binary is provided. There is no GPU/CUDA passthrough; hardware-required rendering cannot be claimed from software rendering. Bubblewrap is installed, but Docker's seccomp/user-namespace policy and host AppArmor/cgroup configuration may prevent nested evaluation sandboxes. Installing a package inside the container does not grant or prove those host isolation capabilities; do not disable them merely to make tests pass.

In a non-privileged Docker run, sandbox tests gated only on Bubblewrap's presence may fail with namespace-permission errors instead of skipping. Report those failures as environment limitations; do not deselect sandbox tests or weaken isolation to claim passing scanner coverage.

No `.corpora` datasets are generated or downloaded. Full qualification needs the exact pinned procedural/curated corpora, public source assets, private manifests/data where applicable, a declared agent adapter, and working host isolation; hosted adapters additionally need authorized credentials/network access outside this enrollment. Recorded qualification reports in the checkout do not demonstrate qualification of this image. Neither full release thresholds nor Windows, hardware, cloud, or real-account behavior is claimed.

## Severity guidance

Code execution or cross-boundary disclosure/write from attacker-supplied model or protocol data is high severity, potentially critical if broadly exploitable. Authentication bypass and source destruction are high when an untrusted actor can reach them. Resource exhaustion is normally medium and should show practical limits and reachability. Cosmetic rendering differences or unsupported hardware by themselves are not security defects.
