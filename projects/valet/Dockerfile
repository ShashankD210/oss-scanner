# Valet (https://github.com/standardsats/valet), an Android Bitcoin and Lightning wallet in Scala.
#
# The toolchain follows the project's Containerfile and F-Droid's builder: Gradle 8.13 and AGP 8.12 run on
# JDK 21, and app/build.gradle forks the Scala 2.11.12 compiler onto a JDK 11 toolchain, because that compiler
# does not run on 21. JDK 11 comes from the Temurin image here, not from the Debian bullseye repository that
# the Containerfile uses, so that the build does not break when bullseye moves to archive.debian.org.
FROM eclipse-temurin:11-jdk-noble AS jdk11

FROM debian:trixie
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    TZ=Etc/UTC
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git unzip openjdk-21-jdk-headless \
    && rm -rf /var/lib/apt/lists/*
COPY --from=jdk11 /opt/java/openjdk /usr/lib/jvm/temurin-11-jdk

# With the license accepted, AGP installs the SDK platform and build tools that app/build.gradle names on first
# use, as with the Containerfile.
ENV ANDROID_HOME=/opt/android-sdk \
    ANDROID_SDK_ROOT=/opt/android-sdk
RUN mkdir -p "$ANDROID_HOME/licenses" \
    && printf '\n24333f8a63b6825ea9c5514f83c2829b004d1fee' > "$ANDROID_HOME/licenses/android-sdk-license"

# Gradle finds the JDK 11 toolchain for ScalaCompile at this path.
RUN mkdir -p /root/.gradle && printf '%s\n' \
      'org.gradle.java.installations.paths=/usr/lib/jvm/temurin-11-jdk' \
      'org.gradle.java.installations.auto-download=false' \
      > /root/.gradle/gradle.properties

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# The other flavors (tnet3, tnet4, regtest) change only the chain and the resources, so mainnet debug is enough.
RUN ./gradlew --no-daemon --console=plain :app:assembleMainnetDebug

# A throwaway JUnit test caches the unit-test classpath and the stub android.jar, so that tests written later
# compile and run offline, also when the repository has no unit tests of its own. A failing test does not abort
# the build.
RUN mkdir -p app/src/test/java/ossscanner \
    && printf '%s\n' \
      'package ossscanner' \
      'import fr.acinq.bitcoin.Satoshi' \
      'import fr.acinq.eclair.MilliSatoshi' \
      'import org.junit.Assert.assertEquals' \
      'import org.junit.Test' \
      'class SmokeSpec {' \
      '  @Test def truncatesToSatoshi(): Unit = assertEquals(Satoshi(1L), MilliSatoshi(1999L).truncateToSatoshi)' \
      '}' \
      > app/src/test/java/ossscanner/SmokeSpec.scala \
    && (timeout 900 ./gradlew --no-daemon --console=plain :app:testMainnetDebugUnitTest \
        || echo "WARNING: unit tests reported failures"); \
    rm -rf app/src/test/java/ossscanner; rmdir -p app/src/test/java 2>/dev/null; true

# gradle.properties gives Gradle an 8 GB heap, which fills the scanner's 8 GB agent machine. This user-level value
# overrides it for later builds.
RUN echo 'org.gradle.jvmargs=-Xmx4g -Dfile.encoding=UTF-8' >> /root/.gradle/gradle.properties
