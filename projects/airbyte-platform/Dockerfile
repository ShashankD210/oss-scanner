# OSS Scanner build for airbytehq/airbyte-platform.
# Network access is available during this build and removed afterwards, so everything Gradle, pnpm and helm need at
# scan time is fetched here.
FROM debian:bookworm

ARG NODE_VERSION=20.19.0
ARG PNPM_VERSION=8.6.12
ARG HELM_VERSION=v3.16.4

ENV DEBIAN_FRONTEND=noninteractive \
    JAVA_HOME=/usr/lib/jvm/temurin-21 \
    GRADLE_USER_HOME=/opt/gradle-home \
    GRADLE_OPTS="-Dorg.gradle.daemon=false -Dorg.gradle.vfs.watch=false" \
    PATH=/opt/node/bin:$PATH

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl gnupg git jq python3 unzip xz-utils procps postgresql-15 xxd \
    && curl -fsSL https://packages.adoptium.net/artifactory/api/gpg/key/public \
      | gpg --dearmor -o /usr/share/keyrings/adoptium.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb bookworm main" \
      > /etc/apt/sources.list.d/adoptium.list \
    && apt-get update && apt-get install -y --no-install-recommends temurin-21-jdk \
    && ln -s "/usr/lib/jvm/temurin-21-jdk-$(dpkg --print-architecture)" /usr/lib/jvm/temurin-21 \
    && rm -rf /var/lib/apt/lists/*

RUN set -eux; \
    case "$(dpkg --print-architecture)" in \
      amd64) NODE_ARCH=x64;   HELM_ARCH=amd64 ;; \
      arm64) NODE_ARCH=arm64; HELM_ARCH=arm64 ;; \
      *) echo "unsupported architecture"; exit 1 ;; \
    esac; \
    mkdir -p /opt/node; \
    curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-${NODE_ARCH}.tar.xz" \
      | tar -xJ -C /opt/node --strip-components=1; \
    npm install -g "pnpm@${PNPM_VERSION}"; \
    curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-${HELM_ARCH}.tar.gz" \
      | tar -xz -C /usr/local/bin --strip-components=1 "linux-${HELM_ARCH}/helm"

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# jOOQ code generation normally starts Postgres with Testcontainers, which needs Docker. Docker is not available in
# this build or at scan time, so patch the generator to use the local Postgres when AIRBYTE_JOOQ_CODEGEN_PG_URL is
# set (a fresh database per run). The Dockerfile also applies scanner-only Gradle build adjustments below.
RUN python3 - <<'PY'
from pathlib import Path

path = Path("airbyte-db/db-lib/src/main/kotlin/io/airbyte/db/instance/FlywayMigrationDatabase.kt")
src = path.read_text()
old = """    val container =
      PostgreSQLContainer(dockerImage)
        .withDatabaseName("jooq_airbyte_configs")
        .withUsername("jooq_generator")
        .withPassword("jooq_generator")
    container.start()

    dataSource = create(container.username, container.password, container.driverClassName, container.jdbcUrl)
"""
new = """    val localPgUrl = System.getenv("AIRBYTE_JOOQ_CODEGEN_PG_URL")
    dataSource =
      if (localPgUrl.isNullOrBlank()) {
        val container =
          PostgreSQLContainer(dockerImage)
            .withDatabaseName("jooq_airbyte_configs")
            .withUsername("jooq_generator")
            .withPassword("jooq_generator")
        container.start()
        create(container.username, container.password, container.driverClassName, container.jdbcUrl)
      } else {
        val dbName = "jooq_" + java.util.UUID.randomUUID().toString().replace("-", "")
        java.sql.DriverManager.getConnection("$localPgUrl/postgres", "jooq_generator", "jooq_generator").use {
          it.createStatement().execute("CREATE DATABASE $dbName")
        }
        create("jooq_generator", "jooq_generator", "org.postgresql.Driver", "$localPgUrl/$dbName")
      }
"""
if old not in src:
    raise SystemExit("FlywayMigrationDatabase.kt changed upstream; update the jOOQ codegen patch in the Dockerfile")
path.write_text(src.replace(old, new))

path = Path("airbyte-api/manifest-server-api/build.gradle.kts")
src = path.read_text()
replacements = {
    "import java.net.URL": "import java.net.URI",
    "val url = URL(remoteSpecUrl)": "val url = URI(remoteSpecUrl).toURL()",
}
for old, new in replacements.items():
    if src.count(old) != 1:
        raise SystemExit(f"Unexpected manifest-server-api build.gradle.kts source for {old!r}")
    src = src.replace(old, new)
path.write_text(src)
PY

# Local Postgres for jOOQ codegen, also usable at scan time: `pg_ctlcluster 15 main start`.
ENV AIRBYTE_JOOQ_CODEGEN_PG_URL=jdbc:postgresql://localhost:5432
RUN pg_ctlcluster 15 main start \
    && su postgres -c "psql -c \"CREATE ROLE jooq_generator LOGIN SUPERUSER PASSWORD 'jooq_generator'\"" \
    && pg_ctlcluster 15 main stop

# Gradle: compile main and test sources for every module, then resolve every resolvable configuration so later
# `./gradlew --offline ...` runs (including tests) find everything in the cache.
RUN mkdir -p "$GRADLE_USER_HOME/init.d" && cat > "$GRADLE_USER_HOME/init.d/resolve-all.gradle" <<'GRADLE'
import org.gradle.api.artifacts.repositories.MavenArtifactRepository
import java.net.URI

beforeSettings { settings ->
  settings.pluginManagement.repositories.maven {
    name = "googleMavenCentralMirror"
    url = URI.create("https://maven-central.storage-download.googleapis.com/maven2")
  }
  settings.pluginManagement.resolutionStrategy.eachPlugin { plugin ->
    if (plugin.requested.id.id.startsWith("io.airbyte.gradle.")) {
      plugin.useVersion("0.78.0")
    } else if (plugin.requested.id.id == "com.dorongold.task-tree") {
      plugin.useVersion("2.1.1")
    } else if (plugin.requested.id.id == "com.github.eirnym.js2p") {
      plugin.useVersion("1.0")
    } else if (plugin.requested.id.id == "org.openapi.generator") {
      plugin.useVersion("7.10.0")
    }
  }
}
beforeProject { project ->
  project.buildscript.repositories.maven {
    name = "googleMavenCentralMirror"
    url = URI.create("https://maven-central.storage-download.googleapis.com/maven2")
  }
  project.buildscript.repositories.maven {
    name = "airbytePublicMaven"
    url = URI.create("https://airbyte.mycloudrepo.io/public/repositories/airbyte-public-jars/")
  }
  project.repositories.whenObjectAdded { repository ->
    if (repository instanceof MavenArtifactRepository && repository.url.toString().contains("repo.maven.apache.org")) {
      repository.url = URI.create("https://maven-central.storage-download.googleapis.com/maven2")
    }
  }
}
allprojects {
  tasks.register("resolveAllDependencies") {
    doLast {
      configurations.findAll { it.canBeResolved }.each { conf ->
        try { conf.resolve() } catch (Exception e) { logger.warn("Could not resolve ${project.path}:${conf.name}: ${e.message}") }
      }
    }
  }
}
GRADLE
# Cap Gradle and Kotlin memory and parallelism. The repository's gradle.properties asks for an 8 GB heap with parallel
# builds, which together with the Kotlin daemon exhausts memory in a container. Settings in GRADLE_USER_HOME take
# precedence over the project's gradle.properties.
RUN printf '%s\n' \
      'org.gradle.jvmargs=-Xmx4g -Xss4m -XX:MaxMetaspaceSize=1g' \
      'org.gradle.parallel=false' \
      'org.gradle.workers.max=2' \
      'org.gradle.caching=true' \
      'kotlin.daemon.jvmargs=-Xmx2g' \
    > "$GRADLE_USER_HOME/gradle.properties"

# The webapp production bundle is skipped here; the webapp step below installs dependencies and generates the client,
# which is what the scanner needs, without running a full vite build alongside the Kotlin compile.
RUN pg_ctlcluster 15 main start \
    && ./gradlew classes testClasses -x :oss:airbyte-webapp:pnpmBuild \
    && ./gradlew resolveAllDependencies -x :oss:airbyte-webapp:pnpmBuild \
    ; status=$?; pg_ctlcluster 15 main stop; exit $status

# Keep the downloaded manifest schema usable by later offline Gradle runs.
RUN python3 - <<'PY'
from pathlib import Path

path = Path("airbyte-api/manifest-server-api/build.gradle.kts")
src = path.read_text()
old = """      val url = URI(remoteSpecUrl).toURL()
      url.openStream().use { input: InputStream ->
        specFile.outputStream().use { output: OutputStream ->
          input.copyTo(output)
        }
      }
      println("Downloaded OpenAPI spec from $remoteSpecUrl to $specFile")
"""
new = """      if (!gradle.startParameter.isOffline || !specFile.exists()) {
        val url = URI(remoteSpecUrl).toURL()
        url.openStream().use { input: InputStream ->
          specFile.outputStream().use { output: OutputStream ->
            input.copyTo(output)
          }
        }
        println("Downloaded OpenAPI spec from $remoteSpecUrl to $specFile")
      } else {
        println("Using cached OpenAPI spec at $specFile")
      }
"""
if src.count(old) != 1:
    raise SystemExit("Unexpected manifest-server-api spec download task source")
path.write_text(src.replace(old, new))
PY

# Webapp: install dependencies and generate the API client and Connector Builder schema while online. After this,
# `pnpm test:ci` regenerates the client from local files only.
ENV AGENTS_SUPPORTED_DEFINITIONS_SKIP_FETCH=true
RUN cd airbyte-webapp \
    && pnpm install --frozen-lockfile \
    && AGENTS_SUPPORTED_DEFINITIONS_SKIP_FETCH=false pnpm run generate-client

# Rendered Helm chart V2 defaults, referenced by threat_model.md.
RUN mkdir -p /src/build/oss-scanner \
    && helm template airbyte charts/v2/airbyte > /src/build/oss-scanner/chart-v2-defaults.yaml

# Cheap tests so you know the image works; failures do not abort the build.
RUN pg_ctlcluster 15 main start; \
    ./gradlew --offline :oss:airbyte-commons-auth:test :oss:airbyte-workload-launcher:test -x :oss:airbyte-webapp:pnpmBuild \
      || echo "WARNING: gradle tests reported failures"; \
    pg_ctlcluster 15 main stop || true

RUN ln -s /opt/node/bin/node /usr/local/bin/node \
    && ln -s /opt/node/bin/npm /usr/local/bin/npm \
    && ln -s /opt/node/bin/npx /usr/local/bin/npx \
    && ln -s /opt/node/bin/pnpm /usr/local/bin/pnpm \
    && chmod -R a+rwX "$GRADLE_USER_HOME" /src