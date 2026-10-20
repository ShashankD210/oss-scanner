# syntax=docker/dockerfile:1
# The native app requires macOS frameworks and cannot be built on Linux.
# This image builds unchanged, portable Swift policy/event code, runs the
# protocol tests offline, and retains the complete app source for review.
# See threat_model.md for the precise coverage and macOS-only limitations.
FROM swift:6.2-noble

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates git ripgrep python3 python3-jsonschema python3-yaml \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# No source rewriting or macOS framework stubs: these files use Foundation.
# Keep debug information and the Swift compiler for offline reproducers.
RUN mkdir -p /src/build/oss-scanner \
    && swiftc -swift-version 5 -g -Onone -parse-as-library \
        -emit-library -emit-module -module-name DeskMindAuditCore \
        -emit-module-path /src/build/oss-scanner/DeskMindAuditCore.swiftmodule \
        -o /src/build/oss-scanner/libDeskMindAuditCore.so \
        app/Shared/FileMention.swift \
        app/Shared/FolderPolicy.swift \
        app/Shared/Routing.swift \
        app/Shared/DownloadSource.swift \
        app/Shared/RecordingCapture.swift \
        app/Shared/HarnessEvents.swift

RUN python3 -m compileall -q protocol/tools
RUN --network=none python3 -m unittest discover -s protocol/tests -v \
    && python3 protocol/tools/check.py \
        protocol/examples/agent-request.json \
        protocol/examples/agent-request.list.json

CMD ["/bin/bash"]
