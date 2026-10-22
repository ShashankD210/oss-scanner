# syntax=docker/dockerfile:1
# Build the Android app's portable wallet-material codecs and their original JVM tests.
# Full APK/device builds need the Android SDK/NDK and platform services; see threat_model.md.
FROM eclipse-temurin:21-jdk-jammy
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl git unzip python3 \
    && rm -rf /var/lib/apt/lists/*
ARG KOTLIN_VERSION=2.1.10
ARG KOTLIN_SHA256=c6e9e2636889828e19c8811d5ab890862538c89dc2a3101956dfee3c2a8ba6b1
RUN curl -fsSL -o /tmp/kotlin.zip "https://github.com/JetBrains/kotlin/releases/download/v${KOTLIN_VERSION}/kotlin-compiler-${KOTLIN_VERSION}.zip" \
    && echo "${KOTLIN_SHA256}  /tmp/kotlin.zip" | sha256sum -c - \
    && unzip -q /tmp/kotlin.zip -d /opt && rm /tmp/kotlin.zip
ENV PATH=/opt/kotlinc/bin:$PATH
RUN mkdir -p /opt/scanner/lib \
    && curl -fsSL -o /opt/scanner/lib/junit.jar https://repo.maven.apache.org/maven2/junit/junit/4.13.2/junit-4.13.2.jar \
    && echo "8e495b634469d64fb8acfa3495a065cbacc8a0fff55ce1e31007be4c16dc57d3  /opt/scanner/lib/junit.jar" | sha256sum -c - \
    && curl -fsSL -o /opt/scanner/lib/hamcrest.jar https://repo.maven.apache.org/maven2/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3.jar \
    && echo "66fdef91e9739348df7a096aa384a5685f4e875584cce89386a7a47251c4d8e9  /opt/scanner/lib/hamcrest.jar" | sha256sum -c -
COPY . /src
WORKDIR /src
COPY --chmod=0755 <<'SH' /usr/local/bin/scanner-core-build
#!/bin/sh
set -eu
cd /src
main=feature-account-impl/src/main/java/jp/co/soramitsu/account/impl/data/repository
tests=feature-account-impl/src/test/java/jp/co/soramitsu/account/impl/data/repository
kotlinc -jvm-target 17 -cp /opt/scanner/lib/junit.jar:/opt/scanner/lib/hamcrest.jar \
  "$main/PortableWalletSemanticMaterial.kt" "$main/PortableWalletAssetRowPresentation.kt" \
  "$tests/PortableWalletSemanticMaterialTest.kt" "$tests/PortableWalletSemanticIOSAuxiliaryTest.kt" \
  "$tests/PortableWalletAssetRowPresentationTest.kt" \
  -include-runtime -d /opt/scanner/wallet-core-tests.jar
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/scanner-core-test
#!/bin/sh
set -eu
cd /src
exec java -cp '/opt/scanner/wallet-core-tests.jar:/opt/scanner/lib/*' org.junit.runner.JUnitCore \
 jp.co.soramitsu.account.impl.data.repository.PortableWalletSemanticMaterialTest \
 jp.co.soramitsu.account.impl.data.repository.PortableWalletSemanticIOSAuxiliaryTest \
 jp.co.soramitsu.account.impl.data.repository.PortableWalletAssetRowPresentationTest
SH
RUN scanner-core-build && scanner-core-test
# Compiler, test dependencies and test binaries remain available for offline reproducers.
