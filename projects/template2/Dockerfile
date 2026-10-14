FROM debian:bookworm

# Install prerequisites during the network-enabled build phase.
RUN apt-get update && apt-get install -y --no-install-recommends \
    perl build-essential libappconfig-perl libtest-leaktrace-perl \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src
# Match the upstream noninteractive configuration, including the XS stash.
RUN perl Makefile.PL TT_QUIET=n TT_ACCEPT=y && make -j2 && make test
