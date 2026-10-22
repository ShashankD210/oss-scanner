# OSS Scanner build for itis-dakota (Python wheel around the Dakota C++ core).
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git curl make patch build-essential cmake ninja-build pkg-config perl \
      gfortran libboost-all-dev liblapack-dev libblas-dev libhdf5-dev \
      libgsl-dev python3 python3-dev python3-venv python3-pip \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Fetches the Dakota 6.24.0 source tarball into /src/dakota and patches it
RUN make get-dakota-src

ENV SETUPTOOLS_SCM_PRETEND_VERSION=0.0.1

RUN python3 -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir --upgrade pip \
    && /opt/venv/bin/pip install --no-cache-dir -v .

# test dependency, installed now because the audit has no network
RUN /opt/venv/bin/pip install --no-cache-dir pytest

ENV PATH=/opt/venv/bin:$PATH
