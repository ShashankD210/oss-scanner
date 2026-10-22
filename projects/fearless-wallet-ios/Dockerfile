# syntax=docker/dockerfile:1
# Linux build of the app-owned portable wallet codecs and original tests.
# UIKit/Keychain/App Attest/CloudKit need macOS and are source-review surfaces here.
FROM swift:6.1-jammy
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates git python3 curl \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
COPY <<'SWIFT' /opt/scanner/Package.swift
// swift-tools-version: 5.9
import PackageDescription
let package = Package(
    name: "FearlessScannerCore",
    dependencies: [.package(url: "https://github.com/apple/swift-crypto.git", exact: "3.9.1")],
    targets: [
        .target(name: "fearless"),
        .testTarget(name: "FearlessScannerCoreTests", dependencies: ["fearless", .product(name: "Crypto", package: "swift-crypto")])
    ]
)
SWIFT
COPY --chmod=0755 <<'SH' /usr/local/bin/scanner-core-build
#!/bin/sh
set -eu
mkdir -p /opt/scanner/Sources/fearless /opt/scanner/Tests/FearlessScannerCoreTests
for name in IOSPortableWalletMaterialEnvelope IOSPortableWalletSemanticMaterial IOSPortableWalletSemanticMaterialValidation IOSPortableWalletAssetRowPresentation IOSPortableAndroidSourceProof; do
    ln -sf "/src/fearless/Common/Model/$name.swift" "/opt/scanner/Sources/fearless/$name.swift"
done
# Separate test copies adapt six fixture concatenations for Swift 6.1's type checker
# and use Apple's Linux Crypto module for the CryptoKit SHA256 API. Every original
# byte fragment, test and assertion is retained; application sources stay byte-identical.
python3 - <<'PY'
from pathlib import Path
import re

tests = Path("/src/fearlessTests")
harness = Path("/opt/scanner/Tests/FearlessScannerCoreTests")
proof_name = "IOSPortableAndroidSourceProofTests.swift"
source = (tests / proof_name).read_text()
fixture = re.compile(
    r"(?m)^(?P<indent> +)let (?P<name>raw|valid|legacyRaw|wrongType)"
    r"(?:: \[UInt8\])? = (?P<expression>[^\n]*\+\n"
    r"(?:[ \t]+[^\n]*\+\n)*[ \t]+[^\n]+)"
)

def typed_chunks(match):
    indent, name = match.group("indent", "name")
    chunks = re.split(r"\s+\+\s*", match.group("expression"))
    assert name + "Chunks" not in source, "Fixture adapter variable already exists"
    rows = "\n".join(indent + "    " + chunk.strip() + "," for chunk in chunks)
    return (f"{indent}let {name}Chunks: [[UInt8]] = [\n{rows}\n{indent}]\n"
            f"{indent}let {name}: [UInt8] = {name}Chunks.flatMap {{ $0 }}")

proof, count = fixture.subn(typed_chunks, source)
assert count == 6, f"Expected six Swift 6.1 fixture adapters, found {count}"
semantic_name = "IOSPortableWalletSemanticMaterialTests.swift"
semantic = (tests / semantic_name).read_text()
semantic, count = re.subn(r"(?m)^import CryptoKit$", "import Crypto", semantic)
assert count == 1, f"Expected one CryptoKit test import, found {count}"
for name, adapted in [(proof_name, proof), (semantic_name, semantic)]:
    destination = harness / name
    if destination.is_symlink():
        destination.unlink()
    destination.write_text(adapted)
PY
cd /opt/scanner
swift test --jobs 2
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/scanner-core-test
#!/bin/sh
set -eu
cd /opt/scanner
exec swift test --skip-build --skip-update
SH
RUN scanner-core-build
