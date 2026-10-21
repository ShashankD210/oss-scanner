# jsse: a JavaScript engine written in Rust (parser, bytecode compiler and
# interpreter, garbage collector, built-ins including RegExp, Intl and
# Temporal). The scanner builds this with the repository checkout as the
# build context.
#
# The shipped release profile is fat LTO with one codegen unit, which
# compiles the whole crate on a single core. For the audit it is overridden:
# no LTO, parallel codegen, and line tables kept so crashes come with readable
# backtraces. The binary still lands at target/release/jsse, where the
# project's scripts look for it.
FROM rust:1-bookworm
ENV CARGO_TERM_COLOR=never \
    CARGO_BUILD_JOBS=4 \
    CARGO_PROFILE_RELEASE_LTO=false \
    CARGO_PROFILE_RELEASE_CODEGEN_UNITS=16 \
    CARGO_PROFILE_RELEASE_DEBUG=line-tables-only \
    RUSTFLAGS="-C force-frame-pointers=yes"

# The rust image already ships git, python3, gcc and make, which is all the
# build (including jemalloc's C sources) and the test scripts need.

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# The scanner clones without submodules. test262 is the engine's conformance
# suite and the main way to exercise it; the spec submodule is not needed.
RUN git submodule update --init --depth 1 test262

# Fetch every dependency now; the audit runs offline. The fuzz crate is
# outside the workspace, so it is fetched on its own (without --locked: its
# lock file is not kept in step with the engine's).
RUN cargo fetch --locked && (cd fuzz && cargo fetch)
ENV CARGO_NET_OFFLINE=true

# The `jsse` command-line engine: `target/release/jsse file.js` runs a script.
RUN cargo build --frozen --release

# In the image, `python3 scripts/run-test262.py --jsse target/release/jsse
# --test262 ./test262 <path>` runs test262 (or a directory of it), `python3
# scripts/run-custom-tests.py tests/` the project's own JS tests, and
# `cargo test --profile release-nolto` the Rust tests (it recompiles the crate
# in test mode).

CMD ["bash"]
