# Threat model

## What jsse is and where untrusted input enters

jsse is a JavaScript engine written in Rust: parser, tree-walking interpreter,
bytecode compiler and VM, garbage collector, and the built-ins (RegExp, Intl,
Temporal, TypedArrays, Atomics, Proxy, ShadowRealm). It is meant to run
JavaScript its embedder does not trust, so treat all script as attacker-controlled.

The build in this image is the `jsse` CLI at `/src/target/release/jsse`. The
attacker controls the source of the main script or module, `eval()`,
`new Function` and dynamic `import()`; every value a script can build (strings,
BigInts, RegExp patterns, buffer sizes, `Proxy` handlers, and getters or
`valueOf` callbacks that run in the middle of a built-in); and `Intl` and
`Temporal` arguments such as locales, time zones and calendars.

The default global environment (which includes the `$262` harness object) is in
scope. The `--node` host floor (`__host_*` globals) and `--can-block` are for
trusted use, so a bug that needs either is out of scope.

## Components that matter most

Most important, because they hold the `unsafe` code and the engine's memory
management:

* `src/types.rs` (tagged values, string and BigInt refcounts via raw `Arc` pointers);
* `src/interpreter/object_arena.rs`, `gc.rs`, `root_stack.rs`;
* `src/interpreter/bytecode/` and the inline caches (`ic.rs`, `ic_store.rs`);
* `builtins/typedarray.rs` and `atomics.rs` (raw buffer pointers, resizable and
  detached `ArrayBuffer`s, shared memory);
* `proxy.rs`, `shadow_realm.rs`, `promise.rs` and generators (user code runs
  while a built-in holds references into engine state).

Also in scope, mostly for DoS and panics: the lexer, the parser, RegExp, and the
`Intl` and `Temporal` built-ins.

Out of scope:

* `fuzz/`, `benches/`, `benchmarks/`, `scripts/`, `tests/`, `research/`, `docs/`,
  and the `test262` and `spec` checkouts;
* `src/cli.rs`, `src/main.rs`, `src/thp.rs`, and the Node compatibility layer
  (`node_host.rs`);
* third-party crates (ICU4X, `jiff`, `chrono`, `jemalloc`, ...) unless script
  reaches a bug in them; if so, name the crate;
* Spectre-style and timing side channels;
* spec-conformance gaps and wrong output with no memory-safety or availability impact.

## How to exercise it

* `target/release/jsse test.js` (`-m` for a module, `-e` for inline source,
  `--no-bytecode` for the tree-walker; the two engines should agree).
* The build has line tables and frame pointers but no AddressSanitizer, and the
  image has stable Rust only. jemalloc hides heap errors from valgrind, so for
  memory errors build the system-allocator variant with
  `cargo build --frozen --release --no-default-features --target-dir /tmp/jsse-sysalloc`
  and run it under `valgrind`.
* `python3 scripts/run-test262.py --jsse target/release/jsse --test262 ./test262 <path>`
  runs conformance tests, `python3 scripts/run-custom-tests.py tests/` the
  project's JS tests, and `cargo test --profile release-nolto` the Rust tests.
* `fuzz/fuzz_targets/` has `parse_roundtrip` and `differential` harnesses.

## How we rate severity

* **Critical:** memory corruption reachable from plain script with a controlled
  read or write (use-after-free, out-of-bounds, type confusion, bad `Arc`
  refcount in `unsafe` code), shown by a reproducer with a controlled effect.
* **High:** the same bug classes without a demonstrated controlled effect; data
  races or undefined behaviour in `unsafe` code; a crash that plainly has a
  stronger form.
* **Medium:** uncontrolled resource exhaustion from a small script (unbounded
  memory, uninterruptible loops, catastrophic regex backtracking), native stack
  overflow that aborts instead of throwing `RangeError`, and `panic!`/`unwrap`
  aborts reachable from script.
* **Low:** wrong results with no path to memory corruption, and leaks of freed
  or uninitialised memory that script cannot read back.

A bug in safe Rust that breaks no memory or availability guarantee is a
correctness bug, not a security finding.

## Anything to leave alone

* test262 failures and other spec deviations (the project tracks these);
* slow scripts, large inputs that legitimately use a lot of memory, and
  deep-recursion `RangeError`s that are thrown and caught correctly;
* `unwrap` and `expect` on values the engine itself just built, and code for
  non-default features or targets (`perf-counters`, Windows, wasm).

## How we would like reports and patches to look

* A minimal `.js` or `.mjs` file, the exact command line, the commit tested, and
  the panic, backtrace or valgrind output.
* A minimal patch against `main` of https://github.com/pmatos/jsse, with a
  regression test as a `.js` file under `tests/` (it passes when it exits 0).
