# fossid_compliance_updater: a Java 8-target Maven CLI (picocli + gson) that reads a FossID scan, looks up
# licenses/copyrights at GitHub and package registries, asks Gemini or Ollama for anything missing, and writes
# the answers back to FossID. Scanner contract: the checkout lives at /src; everything is fetched here, online.
FROM maven:3.9-eclipse-temurin-11
ENV DEBIAN_FRONTEND=noninteractive MAVEN_OPTS="-Dmaven.repo.local=/root/.m2/repository"
RUN apt-get update && apt-get install -y --no-install-recommends git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# Build the fat JAR as the README does, with debug info (javac -g is Maven's default). Also pull every plugin and
# dependency the build needs so the agent can rebuild after editing sources, with the network gone.
RUN mvn -B -q clean package \
    && mvn -B -q dependency:go-offline \
    && mvn -B -q -o clean package \
    && ls -l target/fossidComplianceUpdater.jar

# Smoke test: no test suite exists, so just prove the JAR starts. A failing run must not abort the build.
RUN java -jar target/fossidComplianceUpdater.jar --help || echo "WARNING: --help exited non-zero"
