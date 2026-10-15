FROM debian:bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CC=gcc \
    CFLAGS="-g -O2 -fno-omit-frame-pointer"

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential \
      ca-certificates \
      git \
      iproute2 \
      liburing-dev \
      linux-libc-dev \
      python3 \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN make clean && make -j"$(nproc)"
RUN make test || echo "WARNING: make test reported failures"
