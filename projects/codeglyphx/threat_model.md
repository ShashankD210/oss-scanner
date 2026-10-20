# CodeGlyphX security audit

## Project and trust boundaries

CodeGlyphX is an Apache-2.0-licensed, pure-managed .NET toolkit for QR codes,
barcodes, structured payloads, image decoding, and rendering. It can be embedded
in applications that decode externally supplied images and payloads. Assume
that image bytes, symbol contents, dimensions, metadata, and structured payload
text are attacker controlled. Caller-configured limits are security boundaries.

## Priorities

- Image detection and decoding in CodeGlyphX/Rendering and public ImageReader
  entry points: dimensions, stride and size arithmetic, offsets, compression,
  frames/pages, nesting, allocations, and malformed or truncated input.
- Barcode and QR recognition: malformed symbol data, error-correction lengths,
  bit/byte arithmetic, hostile candidate counts, and bounded recognition work.
- ImageDecodeOptions and decode budgets: enforce configured input, dimension,
  frame/page, memory, cancellation, and recognition-time limits consistently.
- Payload parsing and output escaping: hostile text must not bypass validation
  or become unintended active content in generated HTML, SVG, links, or documents.
- Public convenience APIs and multi-symbol scanning: check that wrappers retain
  core validation and limits, including repeated or crowded candidate images.

All first-party code in this repository is in scope. The Linux image builds the
core library, tests, shared playground, and existing fuzz driver. Windows WPF
applications may be inspected, but cannot be executed in this image.

## Environment and useful entry points

The checkout is /src. .NET 8 and .NET 10 SDKs/runtimes, restored NuGet packages,
built tests, the fuzz executable, and the checked-in image corpus are retained.
The Dockerfile runs a focused decoder/resource-boundary smoke suite on .NET 10.
Use --no-restore during the offline audit.

```sh
# Run correctness coverage without host-sensitive corpus timing measurements.
dotnet test CodeGlyphX.Tests/CodeGlyphX.Tests.csproj -c Release -f net10.0 --no-build --no-restore --filter 'Category!=CorpusTiming'
dotnet test CodeGlyphX.Tests/CodeGlyphX.Tests.csproj -c Release -f net8.0 --no-build --no-restore --filter 'Category!=CorpusTiming'

# The existing managed fuzz driver accepts a file path or bytes on stdin.
dotnet CodeGlyphX.Fuzz/bin/Release/net8.0/CodeGlyphX.Fuzz.dll /tmp/synthetic-input.bin
```

CodeGlyphX.Fuzz/Program.cs exercises format detection, image readers, animation,
individual codecs, and other public decoding paths. Read the driver before
interpreting its results: it bounds inputs and tolerates expected parse errors.
It exposes CODEGLYPHX_FUZZ_TIMEOUT_MS, CODEGLYPHX_FUZZ_MAX_MB,
CODEGLYPHX_FUZZ_MAX_INPUT_MB, and CODEGLYPHX_FUZZ_LOG settings. A successful
driver exit alone does not establish that an input decoded safely or correctly.

Useful regression suites include CodecResourceBoundaryTests,
ImageReaderLimitsTests, ImageReaderAnimationLimitTests, WebpSizeLimitTests,
BarcodeBoundaryReviewTests, and DecodeBudgetSnapshotTests. Retain and reuse
their fixtures and public API patterns when minimizing a finding.

## Intended behavior and scope limits

- Decoding a URL, command-like text, or other arbitrary string as symbol data
  does not authorize navigation or execution. Assess whether CodeGlyphX itself
  crosses that boundary or emits unsafe active content in an output format.
- A normal parse rejection, failed Try method, or documented exception for
  malformed input is not by itself a vulnerability.
- Distinguish unavoidable work on proportionately large caller-authorized input
  from a small input causing disproportionate resource use or bypassing limits.
- Third-party reference implementations, comparison tooling, fixtures, and
  historical benchmark results are not separate audit targets. Attribute any
  finding to a reachable defect in the enrolled first-party code.
- Do not scan production websites or devices, contact external services, or
  include real credentials or private data in a reproducer.

## Findings and severity

Provide a minimal synthetic input, affected public API, exact command, configured
options, and observed impact. For resource findings include input size, decoded
dimensions/frame counts, time, memory, and the limit that should have stopped
processing. Demonstrate the production entry point as well as the root cause.

Use severity proportional to realistic impact. Unauthorized execution, arbitrary
file access/write, secret exposure, or a demonstrated memory-safety exploit may
be high or critical. Bounded-input denial of service or a meaningful configured
limit bypass deserves a report with a concrete impact explanation. Avoid
classifying every managed exception or decoder mismatch as high severity.
Group shared root causes and identify affected sibling codecs or wrappers.
Prefer a minimal corrective patch and a regression test reproducing the defect.
