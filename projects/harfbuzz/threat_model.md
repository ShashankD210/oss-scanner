# HarfBuzz: threat model and severity guide

HarfBuzz is a C++ OpenType text shaping engine and font subsetting library. It is embedded in
browsers, text stacks, PDF viewers and operating-system font services, and font files are routinely
downloaded from the web and rendered without user interaction. Treat every byte of a font as
attacker-controlled.

## What this project does and where untrusted input enters

- **Font parsing** is the main surface. `hb_blob_create*()` and `hb_face_create()` accept arbitrary
  bytes, and the table parsers in `src/OT/` and `src/hb-ot-*.cc` interpret OpenType/TrueType tables,
  including AAT (`src/hb-aat-layout-*`), OpenType Layout (GSUB/GPOS/JSTF), variable fonts
  (`src/hb-ot-var-*`), color fonts (`src/OT/Color/`) and bitmap tables (`CBLC`/`CBDT`, `sbix`,
  `EBDT`/`EBLC`). `hb_sanitize_context_t` (`src/hb-sanitize.hh`) is the validation layer; a bug
  that reaches a parser before or without sanitization is much more serious than a rejected table.
- **Text and shaping input**: `hb_buffer_add_*()`, `hb_shape()`, feature and axis settings, and
  buffer deserialization (`hb_buffer_deserialize_glyphs()`, `hb_buffer_deserialize_unicode()`)
  process attacker-controlled text and parameters.
- **Subsetting and repacking**: `hb_subset_*()` in `src/hb-subset-*` processes an untrusted font plus
  caller-controlled Unicode/feature sets, and the repacker in `src/hb-repacker.hh` manipulates
  attacker-shaped object graphs.
- **Raster, vector and GPU output**: `hb_raster_*`, `hb_vector_*` and `hb_gpu_*` consume the same
  font data after shaping or subsetting.

HarfBuzz does not execute font data as code. Code execution requires memory corruption in the host
process, or a logic bug that changes security-relevant output.

Before substantive analysis, read the checkout's `AGENTS.md`, `README.md`, `BUILD.md`,
`TESTING.md`, `CONFIG.md` and `RELEASING.md`. `AGENTS.md` defines the contributor rules that any
proposed patch is expected to follow.

## Components that matter most / least

- Most important: the core blob/face/font/buffer machinery (`src/hb-*.hh`, `src/hb-*.cc`), the
  sanitizer and serializers (`hb-sanitize.hh`, `hb-serialize.hh`), OpenType table parsing
  (`src/OT/`), the shapers, the subsetter (including the graph helpers in `src/graph/`) and the
  repacker.
- Also in scope: the public API's handling of caller-supplied sizes, counts and lifetimes.
- Lower priority: `perf/` benchmarks, `util/` command-line tools (useful for reproduction, but
  issues confined to the CLI's own argument handling are low), test code and `docs/`.
- Third-party dependencies (freetype, GLib, ICU, Graphite2, Cairo, Chafa, zlib) are separate
  projects. Report a HarfBuzz bug in how it calls them, not bugs inside them.
- The `src/rust/` HarfRust and fontations backends are disabled in these builds, so findings that
  only affect them are out of scope unless a patch enables them.
- The `subprojects/` directory contains Meson wrap files for dependencies; it is out of scope for
  this scan.

## How to exercise it

Two builds are prepared in the image:

- `/src/build`: gcc, debugoptimized, static library, CLI tools and tests. Run the upstream suite
  with `meson test -C /src/build --print-errorlogs`.
- `/src/build-asan`: clang, AddressSanitizer + UndefinedBehaviorSanitizer, with libFuzzer-linked
  fuzzers. Run the non-fuzzing tests with
  `meson test -C /src/build-asan --no-suite=fuzzing`. Run the fuzzers against the prepared corpus:

  ```
  /src/build-asan/test/fuzzing/hb-shape-fuzzer /src/fuzzer_corpus/fonts
  ```

  `hb-subset-fuzzer`, `hb-raster-fuzzer`, `hb-vector-fuzzer` and `hb-gpu-fuzzer` use the same font
  directory; `hb-repacker-fuzzer` uses `/src/fuzzer_corpus/graphs`. The standalone harnesses in
  `/src/build/test/fuzzing/` read the files named on the command line and exit.
- Reproduce from the CLI with the tools in `/src/build/util/` (`hb-shape`, `hb-subset`,
  `hb-view`, `hb-info`, `hb-vector`, `hb-raster`). `test/fuzzing/hb_fuzzing_tools.py` and
  `record-test.py` help reduce a crashing input to a regression test.

For a focused check, upstream's curated suites can be run separately:

```
meson test -C /src/build --suite shape
meson test -C /src/build --suite subset
meson test -C /src/build --suite api
```

`/src/build-asan` sets `fuzzer_ldflags`, so `failing-alloc.c` is part of `HBSOURCES` and
`HB_CUSTOM_MALLOC` is active. `check-includes` and `test-cbdt-cache` are therefore expected to fail
in that build; both pass in `/src/build`, and they should not be "fixed" by changing the tests.

If a patch touches a compile-time feature guard, also build with the optional backends disabled:

```
meson setup /src/build-min /src -Dglib=disabled -Dgobject=disabled -Dcairo=disabled \
  -Dchafa=disabled -Dpng=disabled -Dzlib=disabled -Dicu=disabled -Dfreetype=disabled \
  -Dgraphite=disabled -Dgpu_demo=disabled -Ddocs=disabled -Dintrospection=disabled
meson test -C /src/build-min
```

No network access is needed for any of these commands.

## How we rate severity

- **Critical**: memory corruption reachable from a font or text in a default build, with a
  plausible path to code execution or sandbox escape in a typical embedder (browser, PDF viewer,
  OS font service). Sanitizer bypasses that let attacker-controlled values reach a later out-of-
  bounds access are in this class.
- **High**: memory corruption (out-of-bounds read/write, use-after-free, double free,
  uninitialized use) reachable from untrusted input where exploitability is unclear; an
  integer overflow leading to a short allocation; or an abort in library code that survives
  `NDEBUG` and that a normal embedder can trigger.
- **Medium**: denial of service with small inputs (quadratic or exponential shaping/subsetting,
  excessive memory use), or memory-safety bugs that require non-default API use, a locally supplied
  font, or a debug-only configuration.
- **Low**: `assert()` failures that a release build (`NDEBUG`) compiles out and that do not
  precede memory corruption; issues only in `util/`, tests, benchmarks or documentation; and
  hardening gaps with no demonstrated impact.

A malformed font that HarfBuzz rejects through `hb_sanitize_context_t`, or for which it falls back
to `.notdef` or zero metrics, is intended behavior rather than a finding.

Findings that only reproduce in a supported reduced-feature or 32-bit configuration still count as
real bugs, per `AGENTS.md`; state the configuration and how to reproduce it.

## Allocation failures

The sanitizer build defines `HB_CUSTOM_MALLOC` and links `src/failing-alloc.c`, whose
`hb_malloc_impl()`, `hb_calloc_impl()` and `hb_realloc_impl()` return NULL for roughly one in
sixteen allocations once the fuzzer turns the injector on. The fuzzer enables it when the last byte
of the input is `>= 0x80` (`_fuzzing_alloc_state()` in `test/fuzzing/hb-fuzzer.hh`), and the image
sets `ASAN_OPTIONS=allocator_may_return_null=1` so ASan permits those NULL returns.

This is deliberate OOM simulation, not a source of false positives. An allocation failure that
HarfBuzz does not handle is a valid finding: an unchecked NULL dereference, a crash or assertion,
a leak, corrupted state, or a wrong result a caller could observe. We rate it as at least a denial
of service under the guidance above, and as memory corruption if the unhandled failure leads to an
out-of-bounds access, use-after-free or similar. Please do not dismiss a finding just because the
failed allocation came from `failing-alloc.c`; what matters is whether the failure is handled
safely.

The reference build in `/src/build` does not inject allocation failures, so reproduce these
findings with `/src/build-asan`.

This matches the project's own `AGENTS.md`: fuzzing runs with failing-malloc enabled, so OOM
handling is part of the exercised surface. For the experimental raster, vector and GPU libraries,
best-effort early failure is acceptable; the bar is no crash, no out-of-bounds access and no memory
corruption. Do not propose complex rollback logic just to preserve partial work after OOM, and
follow the allocation and error-handling patterns in the surrounding code.

## Please avoid reporting

- Missing checks for values that `hb_sanitize_context_t` has already validated.
- `assert()` failures that disappear with `-DNDEBUG` and do not precede memory corruption.
- Findings in `perf/`, `util/`, `test/`, `docs/` or the `subprojects/` Meson wrap directory.
- Convention-only deviations from `AGENTS.md` (STL containers in library code, raw `memcpy`,
  missing `likely`/`unlikely`, naming or formatting) that have no security impact.
- Style issues, missing tests, missing documentation, commit-message prefixes or other process
  issues, and build warnings.

## Patches and reproducers

Reproduce the issue before proposing a fix and explain the root cause. Prefer a minimal patch plus a
regression test under `test/`; `test/fuzzing/fonts` and `test/subset/data/fonts` are good homes for
a minimized input. Include the exact harness command, the full ASan/UBSan report and the tests you
ran in the finding.

`AGENTS.md` makes the public API and ABI hard constraints. A proposed patch should:

- Keep the diff minimal and scoped: no unrelated renames, reordering, reformatting or refactors,
  and prefer a targeted fix over simplifying code that may be handling a real edge case.
- Not add, remove or change public API/ABI unless the finding explicitly requires it; if it does,
  follow the `NEWS`, public-header and `docs/` steps in `AGENTS.md`.
- Preserve optional-feature behavior, reduced-feature builds and compile-time feature guards.
- Avoid STL containers in library code, and use HarfBuzz's own containers, `likely`/`unlikely`,
  nil-object patterns, and `hb_memcpy`/`hb_memcmp`/`hb_memset` from `hb-algs.hh` where the
  surrounding code does.
- Fail safely on allocation failure and follow the existing allocation and error-handling patterns.
- Add or update the nearest regression test, fuzz seed or fuzz target when hardening a parser, and
  inspect `test/fuzzing/` and the existing guardrail code before changing validation.
- Run the full suite (`meson test -C /src/build`) and the relevant targeted suites before proposing
  a patch, and report any failures.
- Rebuild the touched targets and check for new warnings; for `util/` changes, exercise the rebuilt
  tool on a small sample.
- Update generators rather than hand-editing generated tables, and keep Meson and CMake in sync
  when build wiring changes without silently changing Meson or feature defaults.
- Leave unrelated local fonts, generated outputs and untracked artifacts alone, and never disable
  or weaken tests to get a green run.
