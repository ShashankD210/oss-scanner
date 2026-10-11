# Hometown XR

This Python pipeline extracts and curates multilingual narratives from archived
web records. It reads Common Crawl WARC/WET/ARC data, compressed files, remote
catalogs, and local exports, maintains resumable state, and offers local review
and annotation interfaces.

## Inputs and boundaries

Treat archived documents, archive metadata, content URLs, compressed streams,
filenames, imported JSON, and HTTP requests to local workbenches as untrusted.
Prioritize decompression/resource exhaustion, archive parsing, path traversal,
unsafe deserialization, URL and local-file access, injection into exported HTML
or review pages, and unauthorized changes through a local HTTP interface.
Inspect `downloader.py`, `processor.py`, `output.py`, `annotation_workbench.py`,
`story_workbench.py`, checkpoints, database recovery, and export paths.

Differentiate operator-selected local paths and model configuration from paths
or commands that hostile downloaded data can select. Trusted model checkpoints
are not permission for untrusted archive data to load arbitrary code.

## Offline checks

The checkout is `/src`; the image installs the project's Python dependencies and
CPU PyTorch. Run:

```sh
python -m pytest
python main.py --help
```

The test suite uses synthetic records and mocks for heavyweight inference. Model
weights, a live crawl, authenticated object storage, CUDA, Windows PowerShell,
and workstation-specific data are not included. Do not start production crawls,
download model weights during the offline scan, or edit preserved datasets.
Use small local WARC/compression fixtures, temporary databases, and loopback
servers. Full embedding and language-model quality/performance are outside this
Linux source/test environment; their surrounding input-handling code is in scope.

`python dependency_profiles.py --check` is a separate security/release-policy
diagnostic. At preparation time it reports an expired dependency review deadline
of 2026-08-31. The image preserves that policy and the declared model stack;
passing build/tests does not renew the exception or establish dependency safety.
Review the documented dependency risks and operational restrictions in scope.
Large Git LFS datasets are not needed by the synthetic test suite (source CI also
checks out with LFS disabled); production datasets are not validation fixtures.

## Severity and reporting

Prioritize attacker-controlled code execution, arbitrary file reads/writes,
credential leakage, and compromise of the reviewer's browser or local service.
For denial of service, show a small input with disproportionate cost and identify
existing limits. Explain required CLI options and network exposure. Provide an
offline reproducer, affected code path, and minimal patch with a regression test.
Keep unpatched findings private and never include real archive subjects' data.
