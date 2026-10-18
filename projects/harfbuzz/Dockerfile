# OSS Scanner build for HarfBuzz (https://github.com/harfbuzz/harfbuzz).
#
# HarfBuzz parses untrusted font files, so this builds the two configurations its own CI uses and
# google/oss-fuzz's projects/harfbuzz fuzzes:
#
#   /src/build       gcc, -O2 -g -fno-omit-frame-pointer, static library and CLI tools
#                    (hb-shape, hb-subset, hb-view, ...). Test binaries and standalone fuzzer
#                    harnesses live under /src/build; each harness reads the font files named on
#                    its command line and exits:
#                      /src/build/test/fuzzing/hb-{shape,subset,raster,vector,gpu,repacker}-fuzzer
#   /src/build-asan  clang, ASan + UBSan, static library, libFuzzer-linked fuzzers:
#                      /src/build-asan/test/fuzzing/hb-{shape,subset,raster,vector,gpu,repacker}-fuzzer
#
# The flags follow oss-fuzz's build.sh: experimental APIs enabled, -fno-sanitize=vptr because
# HarfBuzz builds with -fno-rtti, and -DHB_NO_VISIBILITY for the static build.
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential clang libclang-rt-18-dev lld llvm \
      meson ninja-build pkg-config ragel python3 \
      libfreetype-dev libglib2.0-dev libcairo2-dev libicu-dev libgraphite2-dev \
      libchafa-dev libpng-dev zlib1g-dev \
      gdb valgrind file xxd jq \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# 1) Reference build: upstream's linux CI configuration, with all optional backends enabled.
RUN CC=gcc CXX=g++ meson setup /src/build /src \
      --buildtype=debugoptimized \
      --default-library=static \
      --wrap-mode=nodownload \
      -Dglib=enabled \
      -Dgobject=enabled \
      -Dcairo=enabled \
      -Dchafa=enabled \
      -Dpng=enabled \
      -Dzlib=enabled \
      -Dicu=enabled \
      -Dfreetype=enabled \
      -Dgraphite=enabled \
      -Dgpu_demo=disabled \
      -Dexperimental_api=true \
      -Ddocs=disabled \
      -Dintrospection=disabled \
      -Dbenchmark=disabled \
      -Dc_args="-g -fno-omit-frame-pointer" \
      -Dcpp_args="-g -fno-omit-frame-pointer" \
    && meson compile -C /src/build -j "$(nproc)"

# 2) Sanitizer build: the fuzzers link libFuzzer. b_lundef=false is needed when linking with
#    -fsanitize=*, and -fno-sanitize-recover makes a UBSan report abort like an ASan one.
RUN CC=clang CXX=clang++ meson setup /src/build-asan /src \
      --buildtype=debugoptimized \
      --default-library=static \
      --wrap-mode=nodownload \
      -Dglib=enabled \
      -Dgobject=enabled \
      -Dcairo=enabled \
      -Dchafa=enabled \
      -Dpng=enabled \
      -Dzlib=enabled \
      -Dicu=enabled \
      -Dfreetype=enabled \
      -Dgraphite=enabled \
      -Dgpu_demo=disabled \
      -Dexperimental_api=true \
      -Ddocs=disabled \
      -Dintrospection=disabled \
      -Dbenchmark=disabled \
      -Db_sanitize=address,undefined \
      -Db_lundef=false \
      -Dfuzzer_ldflags="-fsanitize=fuzzer" \
      -Dc_args="-g -O1 -fno-omit-frame-pointer -fno-sanitize=vptr -fno-sanitize-recover=undefined -fsanitize=fuzzer-no-link -DHB_NO_VISIBILITY" \
      -Dcpp_args="-g -O1 -fno-omit-frame-pointer -fno-sanitize=vptr -fno-sanitize-recover=undefined -fsanitize=fuzzer-no-link -DHB_NO_VISIBILITY" \
    && meson compile -C /src/build-asan -j "$(nproc)"

# HarfBuzz's fuzzers can make its allocator fail on purpose, so ASan must let malloc return NULL.
ENV ASAN_OPTIONS=allocator_may_return_null=1:abort_on_error=1:detect_leaks=1:symbolize=1 \
    UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1

# Seed corpus: the font trees oss-fuzz zips for every font fuzzer, plus the repacker's graph corpus.
# For example: /src/build-asan/test/fuzzing/hb-shape-fuzzer /src/fuzzer_corpus/fonts
RUN set -e; mkdir -p /src/fuzzer_corpus/fonts /src/fuzzer_corpus/graphs \
    && for d in \
         test/shape/data/in-house/fonts \
         test/shape/data/aots/fonts \
         test/shape/data/text-rendering-tests/fonts \
         test/api/fonts \
         test/fuzzing/fonts \
         test/subset/data/fonts \
         perf/fonts; do \
         cp -a "$d"/. /src/fuzzer_corpus/fonts/; \
       done \
    && cp -a test/fuzzing/graphs/. /src/fuzzer_corpus/graphs/

# Smoke tests. The regular build runs the full upstream suite, including its standalone fuzzer
# harnesses. The ASan build skips the fuzzing suite (those binaries are libFuzzer targets, not the
# TAP drivers meson expects) and two checks that cannot pass in an oss-fuzz-style build: with
# fuzzer_ldflags set, failing-alloc.c joins HBSOURCES and check-includes rejects its first include,
# and HB_CUSTOM_MALLOC bypasses the allocation-failure injection test-cbdt-cache relies on.
# Failures are reported but never abort the image build.
RUN cd /src/build \
    && (timeout 20m meson test --no-rebuild --print-errorlogs --timeout-multiplier 10 \
        || echo "WARNING: harfbuzz tests reported failures or ran out of time")

RUN cd /src/build-asan \
    && tests=$(meson introspect . --tests \
                  | jq -r '.[] | select(((.suite // []) | map(. == "fuzzing" or endswith(":fuzzing")) | any) | not) | .name' \
                  | grep -v -x -e check-includes -e test-cbdt-cache | tr '\n' ' ') \
    && test -n "$tests" \
    && (ASAN_OPTIONS=allocator_may_return_null=1:abort_on_error=1:detect_leaks=0:symbolize=1 \
          timeout 20m meson test --no-rebuild --print-errorlogs \
          --timeout-multiplier 10 $tests \
        || echo "WARNING: harfbuzz ASan tests reported failures or ran out of time")

# One input per libFuzzer target, with a trailing byte that turns on HarfBuzz's failing allocator
# (AGENTS.md: fuzzers run with failing-malloc enabled), to show that each target starts, links and
# survives an out-of-memory path.
RUN set -e; for f in shape subset raster vector gpu repacker; do \
      corpus=/src/fuzzer_corpus/fonts; \
      if [ "$f" = repacker ]; then corpus=/src/fuzzer_corpus/graphs; fi; \
      seed=$(find "$corpus" -type f | head -n 1); \
      cp "$seed" /tmp/hb-fuzz-input; \
      printf '%b' '\200' >> /tmp/hb-fuzz-input; \
      ASAN_OPTIONS=allocator_may_return_null=1:abort_on_error=1:detect_leaks=0:symbolize=1 \
        timeout 2m "/src/build-asan/test/fuzzing/hb-$f-fuzzer" -runs=1 /tmp/hb-fuzz-input \
        >/dev/null 2>&1 || echo "WARNING: hb-$f-fuzzer failed its smoke run"; \
    done
