# Threat model: itis-dakota

## What this project is

A Python wheel built by the IT'IS Foundation around upstream Dakota
(snl-dakota/dakota), a numerical optimization and uncertainty-quantification
engine. The repository adds packaging: `build_backend/`, build patches in
`src_patches_v6*/`, and wheel/SBOM scripts in `scripts/`. The Dakota C++
sources live in `dakota/` and are built unmodified apart from those patches.
Users call it through `import dakota` or the `dakota` executable.

## Trust model

- Dakota input files, analysis drivers (fork/system interfaces) and Python
  interfaces are authored by the user running Dakota. Executing the commands
  or scripts they name is intended behaviour, not a vulnerability.
- The user is trusted to choose which files Dakota reads. Dakota is not a
  sandbox and is not meant to process hostile input from remote parties.

## Untrusted input worth testing

Reports are in scope when a crafted file, with no malicious driver involved,
corrupts memory or runs code in:

- the input file parser (NIDR and the Dakota grammar in `dakota/src` and
  `dakota/packages/pecos`/`dakota_parser`);
- binary restart files (Boost serialization);
- tabular and imported data files, and `params`/`results` files returned by
  drivers;
- HDF5 input and output handling.

## Severity guidance

- High or critical: memory corruption (out-of-bounds read/write,
  use-after-free) reachable from a crafted input, restart or data file,
  especially with a working reproducer.
- Medium: crash, hang or unbounded memory use from a crafted file with no
  memory corruption. Unsafe temporary-file handling in the driver
  interface.
- Low: crashes from malformed input where the user already controls
  everything else, and undefined behaviour with no demonstrated impact.
- Not a vulnerability: anything that needs a malicious analysis driver,
  Python interface script or input file the user wrote themselves, and
  numerical-accuracy bugs.

## Out of scope

- Third-party code vendored under `dakota/packages/` and
  `dakota/dakota-examples/`, unless reachable through Dakota's own entry
  points. For upstream Dakota bugs, please say so in the report so they can
  be forwarded upstream.
- Windows/MinGW and macOS patches and CI workflows. Only the Linux build is
  scanned.
- Test and example inputs in `tests/` and `dakota/examples/`.

## Supply chain (lower priority)

Report issues in `build_backend/`, `scripts/` and the patch files that could
alter the built wheel, such as downloads without integrity checks.

## Report format

Please include a reproducer that uses the `dakota` executable or the Python
API, and keep proposed patches minimal and limited to the affected code.
