# FlexGPU TouchDesigner

This project provides Python source, configuration validation, frame transport,
workers, and TouchDesigner adapters for real-time generated point worlds. Public
code is hardware-neutral; live projects, licensed components, private model
weights, and local workstation profiles are not part of this enrollment.

## Inputs and boundaries

Treat network/frame metadata, transported arrays and image files, discovery
messages, shared-directory contents, and imported configuration as untrusted
where they can cross a process or machine boundary. Prioritize unchecked sizes,
malformed/non-finite arrays, traversal and symlink races, unsafe deserialization,
command construction, process identity mistakes, and control messages enabling
workers or changing outputs without authorization. Review `src/`, `tools/`,
`integrations/`, worker scripts, and public TouchDesigner adapters.

Operator-owned configuration can intentionally contain commands; distinguish
that capability from untrusted runtime data altering the command or path. Keep
default-off live integrations and role/phase gates intact.

## Offline checks and runtime limits

The checkout is `/src`. Run:

```sh
python -m unittest discover -s tests -v
python -m compileall -q src tools tests touchdesigner
```

The image installs the same pinned source-test dependencies as CI. It provides
portable/source tests, not the complete Windows PowerShell release gate or live
TouchDesigner acceptance. GPU/model runtimes, proprietary adapters, VR/headsets,
cameras, and licensed components are unavailable. Use tiny synthetic frames,
temporary directories and loopback transports. Do not start show processes,
connect to installation hardware, or inspect private component internals.

## Severity and reporting

Prioritize execution, arbitrary reads/writes, secret leakage, and remotely
triggered persistent disruption. State the exposed transport and trust settings,
and distinguish a live integration hypothesis from an exercised source-level
reproducer. Include a minimal offline test and patch without rewriting local
profiles or changing deployment defaults. Keep reports private.
