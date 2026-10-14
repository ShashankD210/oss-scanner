FROM debian:bookworm

# Fetch build and test dependencies while network access is available.
RUN apt-get update && apt-get install -y --no-install-recommends \
    perl build-essential libb-cow-perl \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src
RUN perl Makefile.PL && make -j2 && make test
