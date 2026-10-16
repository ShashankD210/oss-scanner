# Threat model



## Scope and trust boundary

pytest-env is a pytest plugin that reads environment assignments from pytest configuration and optional dotenv files,
then mutates `os.environ` before test collection. Configuration supports conditional replacement, Python string
formatting against the existing environment and raw values.

A repository's tests, pytest plugins and configuration are normally trusted to execute code. pytest-env is not a sandbox
for an untrusted checkout. Its smaller security boundary is preserving the exact variable names and values the
configuration requests, avoiding unintended reads when loading dotenv files, and keeping diagnostics from disclosing
secret values.

Inputs to test as hostile data include:

- `env` entries in `pytest.ini`, `tox.ini`, `setup.cfg` and `pyproject.toml`, including malformed flags, duplicate keys,
  empty names, Unicode, braces and long values;
- values already present in the process environment and referenced through formatting;
- dotenv paths, file contents, quoting, interpolation and encoding;
- pytest root-directory and configuration-file discovery, especially with nested repositories and `--rootdir`;
- terminal and log output produced for invalid configuration.

The configured variable values are allowed to affect tests. A security finding should show an effect beyond assigning
the named environment variable, reading the explicitly selected dotenv file, or standard Python formatting behavior
documented by the plugin.



## Components to examine

`src/pytest_env/plugin.py` contains configuration declaration, parsing and environment mutation. Review ordering between
dotenv values, existing variables and inline `env` entries; handling of the `skip_if_set`, `transform` and `raw` flags;
and behavior when one entry fails after earlier entries changed the process environment.

Check format-string expansion for accidental access beyond environment keys, unexpected recursive expansion and secret
values included in exceptions. Dotenv path resolution must be relative to the documented pytest root and must not
silently select a similarly named file above the project. Configuration parsing should reject malformed entries rather
than treating flag text as part of a variable name.

The `python-dotenv` parser and pytest's own configuration discovery are dependencies. Findings belong here when the
plugin supplies an unsafe path, merges values incorrectly or exposes data returned by those dependencies.



## Exercising the image

The checkout is at `/src`, installed in `/opt/venv` with its test group. Run `python -m pytest tests`. Most tests invoke
isolated pytest runs through `pytester`; add a small temporary project with a configuration file and assert on the child
process environment. Tests should restore the parent environment and use placeholder secrets.



## Severity guidance

- High: configuration for one named variable causes pytest-env to execute code or read an unrelated credential file in a
  normal trusted-test workflow where the configuration itself is treated as data.
- Medium: the plugin exposes an existing secret in logs or exceptions, resolves a dotenv path outside the documented
  root, or overwrites a variable despite an active `skip_if_set` guarantee.
- Low: denial of service, an incorrect value confined to the test process, or diagnostics containing only values the
  repository already supplied.

Include the pytest invocation, configuration format, initial environment and observed environment in reports.



## Out of scope

- Arbitrary behavior from tests, conftest files or other pytest plugins in the repository.
- A repository deliberately assigning a dangerous variable such as `PATH`, `PYTHONPATH` or a tool-specific command.
- Python format fields that perform the documented environment-key lookup, unless they access something other than the
  mapping value.
- Vulnerabilities wholly inside pytest or python-dotenv that reproduce without pytest-env.
- Secrets already printed by the test suite or another plugin.
