# Threat model



## Scope and trust boundary

pyproject-api reads a project's PEP 517 build-system configuration and communicates with its build backend, either in a
persistent process or a fresh subprocess. It exposes hook results for requirements, metadata, source distributions,
wheels and editable wheels. When a backend lacks a metadata hook, it builds a wheel and extracts its `.dist-info`
directory.

The build backend selected by `pyproject.toml` is arbitrary project code and is trusted to execute. The library is not a
sandbox. Its security responsibilities are to constrain in-tree backend paths, preserve the JSON protocol between the
frontend and backend, validate hook result types and paths, and avoid filesystem effects beyond caller-supplied output
directories.

Hostile inputs include:

- `pyproject.toml`, particularly `build-backend`, `backend-path` and requirement strings;
- hook return values, exception fields and stdout/stderr produced by a backend;
- wheel filenames and archive entries returned by `build_wheel` or `build_editable`;
- metadata and output directories supplied by an API caller, including symlinks and pre-existing contents;
- configuration settings serialized into the backend protocol;
- concurrent backend output mixed with protocol messages, truncated response files and invalid JSON.



## Components to examine

Prioritize `_frontend.py`. `_in_tree_path` must reject absolute paths, traversal, symlink escapes, Windows drive/UNC
forms, NULs and races while resolving a backend path. Every hook result joined to a wheel, sdist or metadata directory
must remain inside that directory and refer to the artifact type the hook promised.

`metadata_from_built` opens a backend-produced wheel and extracts `.dist-info` entries. Test absolute names, `..`,
backslashes, repeated separators, symlink-like entries, duplicate names, huge compression ratios and a first path
component that merely ends in `.dist-info`. Extraction must not write outside the selected metadata directory or accept
metadata from a second distribution.

Review `ensure_empty_dir` and `_check_metadata_dir` for recursive deletion through symlinks and for aliases of the
project root. The frontend/backend response-file protocol uses a temporary JSON path: malformed output must not make the
frontend read, delete or wait on another path. Persistent backend reuse must not confuse responses between hooks or
retain caller data that should be per request.



## Exercising the image

The project is at `/src`, installed in editable mode in `/opt/venv`; test dependencies and a wheel are available
offline. Run `python -m pytest tests`. `tests/demo_pkg_inline` and the backend fixtures cover in-tree backends. A useful
security test implements a minimal backend that returns crafted filenames or creates a wheel containing hostile archive
entries, then calls only the public `Frontend` API.



## Severity guidance

- High: a backend-produced artifact or archive entry makes pyproject-api write or delete outside a caller-supplied
  output directory, or a rejected `backend-path` imports code outside the project that would not otherwise be selected.
- Medium: hook result confusion returns another build's artifact, malformed protocol data causes an operation on an
  unrelated temporary file, or a small archive causes substantial resource exhaustion.
- Low: a controlled exception, hang, invalid metadata result or leftover file confined to a temporary/output directory.

Include the Python version, frontend implementation, project tree, backend code, hook sequence and filesystem result in
reports.



## Out of scope

- Arbitrary code execution or filesystem access by the backend selected in `pyproject.toml`.
- Requirement installation and dependency resolution; callers such as tox perform those operations.
- A caller deliberately passing the project root as an output directory after the library rejects it.
- Vulnerabilities in a build backend or `packaging` that reproduce without pyproject-api.
- Backend stdout and stderr containing secrets the backend itself printed.
