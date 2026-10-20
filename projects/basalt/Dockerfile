# Scanner build recipe for Basalt, a Swift package. This file is the whole build: it installs nothing beyond
# the official toolchain image, fetches every dependency while network access is still available, then builds.
# Basalt declares .macOS(.v15) and imports Metal, Security, os, SwiftUI and AppKit in some targets, so this
# recipe is expected to fail on Linux until those imports are guarded. It is kept unchanged on purpose.
FROM swift:6.3.2-noble
ENV DEBIAN_FRONTEND=noninteractive

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Fetch every dependency now. Network access is removed after this image is built.
RUN swift package resolve

# Build the project the way its README does (swift build -c release), with debug info.
RUN swift build -c release -Xswiftc -g

# Build the tests so they can run offline. A failing test must not abort the build.
RUN swift build --build-tests || echo "WARNING: test build failed"
RUN swift test --skip-build || echo "WARNING: tests reported failures"
