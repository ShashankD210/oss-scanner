# Threat model



## Scope and trust boundary

build is the reference-style Python build frontend for PEP 517 projects. `ProjectBuilder` reads `pyproject.toml`, asks
the selected backend for build requirements and metadata, creates an isolated environment, installs requirements, and
invokes backend hooks to produce a source distribution or wheel. The CLI exposes the same flow through `python -m build`
and `pyproject-build`.

A build backend named by a project is arbitrary code and is trusted to execute with the user's privileges. Building an
untrusted source tree is therefore not a sandboxed operation. The frontend is responsible for invoking the backend and
installer it selected, maintaining argument and path boundaries, and rejecting malformed backend results before it
copies artifacts to the requested output directory.

Hostile inputs include:

- `pyproject.toml` build-system requirements, backend name and backend path;
- backend hook return values: dependency strings, metadata directories, artifact filenames and exceptions;
- source-tree paths, output-directory paths, symlinks and pre-existing files;
- wheel and sdist filenames and archive bytes returned by the backend;
- subprocess output, locale-dependent text and environment variables forwarded to isolated builds;
- custom runners supplied through the Python API. The runner itself is trusted, while build must pass it the exact
  command, working directory and extra environment required by the documented API.



## Components to examine

Prioritize `src/build/_builder.py`: validation of build-system configuration, backend-path containment, hook return
values, dependency checks, metadata preparation and final artifact selection. Examine `env.py` for isolated-environment
creation, interpreter choice and requirement installation, and `__main__.py` for CLI path handling and cleanup.

Check that a backend cannot use a crafted relative or absolute artifact name to make build accept a file outside the
backend output directory, overwrite an unrelated destination, or confuse an artifact left by an earlier hook. Review
temporary-directory cleanup and symlink behavior. Requirement parsing must preserve markers and URLs without turning
data into extra installer arguments.

The PEP 517 backend and installer are separate trust domains. A defect wholly inside pip, uv, virtualenv,
`pyproject-hooks` or a selected backend is out of scope unless build forms unsafe arguments or trusts a result it should
validate.



## Exercising the image

The checkout is at `/src`; build and its test dependencies are installed in `/opt/venv`, and a wheel is under
`/opt/wheels`. Run `python -m pytest tests`. The fixtures in `tests/packages` and helpers in
`tests/integration_sources.py` cover in-tree backends, malformed metadata and artifact production. Reproductions should
use a small source tree with a minimal backend that returns a crafted value from one hook.

The scan is offline. Tests that download build requirements need a local wheel or mocked installer; this is useful for
separating frontend behavior from an index or installer.



## Severity guidance

- High: a backend return value causes build, rather than the already-trusted backend, to overwrite or disclose a file
  outside the source, temporary and requested output directories; or build loads a backend outside the permitted
  `backend-path` without that code already being importable by design.
- Medium: dependency or command argument injection makes the frontend install a different requirement than the backend
  returned, an artifact from the wrong location is reported as the result, or cleanup follows a crafted symlink.
- Low: denial of service, misleading diagnostics, accepting an invalid artifact that has no demonstrated security
  impact, or leaving files inside a temporary build directory.

Include the frontend API or CLI command, source tree, backend hook results, selected runner and resulting filesystem
changes in a report.



## Out of scope

- Arbitrary behavior by the backend named in `pyproject.toml`, including network and filesystem access.
- Code execution by build requirements or the package installer during installation.
- Malicious contents inside a wheel or sdist that build returns without opening; consumers must validate archives when
  they extract or install them.
- A custom runner that changes the argv or environment passed to it.
- Package-index compromise, dependency confusion from index configuration, and unsupported backend behavior.
