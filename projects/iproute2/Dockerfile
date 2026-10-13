# OSS Scanner build for iproute2.
#
# Two builds are produced. iproute2 only builds in-tree, so the second is a copy of the checkout:
#   /src        gcc, -O2 -g -fno-omit-frame-pointer. Mirrors a distribution build.
#               Tools: /src/ip/ip, /src/tc/tc, /src/misc/ss, /src/bridge/bridge,
#               /src/devlink/devlink, /src/rdma/rdma, /src/dcb/dcb, /src/vdpa/vdpa
#   /src-asan   clang, AddressSanitizer + UndefinedBehaviorSanitizer. Use it to confirm
#               memory-safety findings. Same layout: /src-asan/ip/ip and so on.
#
# The attack surface in threat_model.md is data arriving from the kernel: netlink replies and
# dumps, and procfs/sysfs content. /src/testsuite/tools/generate_nlmsg builds a netlink message
# that exercises the parsers without a kernel, and is the cheapest way to reach that surface.
#
# The upstream testsuite (make -C testsuite alltests) drives real interfaces and needs
# "sudo -E unshare -n", so it runs in the scanner's privileged container, not here.
FROM debian:trixie

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8

# Dependencies for every feature probed by ./configure. A probe that fails does not fail the
# build, it silently drops the feature, so the configure results are checked below.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential clang libclang-rt-dev pkg-config \
      bison flex gdb \
      libmnl-dev libelf-dev libbpf-dev libcap-dev libselinux1-dev \
      libxtables-dev libtirpc-dev libdb-dev libbsd-dev zlib1g-dev \
      iproute2 iputils-ping man-db groff-base locales jq xxd file \
    && rm -rf /var/lib/apt/lists/*

# man/Makefile runs its syntax check under LC_ALL=en_US.UTF-8, which has to exist or every page
# reports a locale warning instead of a real one.
RUN sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen && locale-gen en_US.UTF-8

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# 1) Reference build (distribution-like, with debug info and frame pointers).
#
# CCOPTS and LDFLAGS are passed in the environment, not on the make command line. ip/Makefile and
# tc/Makefile both do "LDFLAGS += -Wl,-export-dynamic", which tc's .so action modules need, and a
# command line assignment overrides that append instead of adding to it. An environment value is
# appended to as intended. For the same reason netem/Makefile can keep its own CCOPTS for the
# generator programs it compiles and runs during the build.
RUN cd /src && ./configure \
    && CCOPTS="-O2 -g -fno-omit-frame-pointer" make -j"$(nproc)"

# A missing libmnl drops devlink, rdma, tipc, dcb, vdpa, netshaper and dpll from SUBDIRS, and a
# missing libelf or xtables quietly reduces what tc can parse. Fail now rather than ship a build
# that is missing the subcommands the scanner is meant to examine.
RUN cd /src && for feature in HAVE_MNL HAVE_ELF HAVE_LIBBPF HAVE_CAP TC_CONFIG_XT; do \
      grep -q "^$feature:\?=y" config.mk || { echo "ERROR: configure did not enable $feature"; exit 1; }; \
    done

# 2) Sanitizer build. The sanitizer flags have to reach the link step as well as the compiles,
#    and tc links its action modules with LDFLAGS, so both variables carry them.
RUN cp -a /src /src-asan \
    && cd /src-asan && make clobber \
    && CC=clang ./configure \
    && grep -E '^(HAVE_|TC_CONFIG_)' /src/config.mk | sort > /tmp/feat-gcc \
    && grep -E '^(HAVE_|TC_CONFIG_)' /src-asan/config.mk | sort > /tmp/feat-clang \
    && diff /tmp/feat-gcc /tmp/feat-clang \
    && CCOPTS="-O1 -g -fno-omit-frame-pointer -fsanitize=address,undefined -fno-sanitize-recover=undefined" \
       LDFLAGS="-fsanitize=address,undefined" \
       make -j"$(nproc)" CC=clang

# Sanitizer defaults for anyone running the binaries in /src-asan. Leak detection is off: these
# are short-lived programs that exit without freeing, which threat_model.md puts out of scope.
ENV ASAN_OPTIONS=detect_leaks=0:abort_on_error=1:symbolize=1 \
    UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1

# Netlink message generator used by the testsuite, built while the network is still up.
RUN make -C /src/testsuite/tools -j"$(nproc)" \
    || echo "WARNING: generate_nlmsg failed to build"

# Smoke test: every binary runs, and the ASan build parses a netlink dump of the loopback
# device. Failures are reported but never abort the build.
RUN for tool in ip/ip tc/tc misc/ss bridge/bridge devlink/devlink rdma/rdma dcb/dcb vdpa/vdpa; do \
      /src/$tool -V >/dev/null 2>&1 || echo "WARNING: /src/$tool -V failed"; \
    done; \
    /src-asan/ip/ip -json addr show lo >/dev/null || echo "WARNING: ASan ip addr show reported errors"; \
    /src-asan/misc/ss -a >/dev/null || echo "WARNING: ASan ss reported errors"

# Man page syntax check: these document the command line that the parsers accept.
RUN make -C /src/man check || echo "WARNING: man page checks reported errors"
