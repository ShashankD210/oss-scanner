FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_NOLOGO=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 bash dotnet-sdk-10.0 dotnet-sdk-8.0 \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# The test project builds the core library for .NET 8 and .NET 10 and the
# shared playground on .NET 10. Windows WPF applications are not Linux targets.
RUN dotnet restore CodeGlyphX.Tests/CodeGlyphX.Tests.csproj --disable-parallel \
    && dotnet build CodeGlyphX.Tests/CodeGlyphX.Tests.csproj \
      --configuration Release --no-restore --maxcpucount:2 \
    && dotnet build CodeGlyphX.Fuzz/CodeGlyphX.Fuzz.csproj \
      --configuration Release --maxcpucount:2

RUN dotnet test CodeGlyphX.Tests/CodeGlyphX.Tests.csproj \
      --configuration Release --framework net10.0 --no-build --no-restore \
      --filter 'FullyQualifiedName~CodecResourceBoundaryTests|FullyQualifiedName~ImageReaderLimitsTests|FullyQualifiedName~ImageReaderAnimationLimitTests|FullyQualifiedName~WebpSizeLimitTests|FullyQualifiedName~BarcodeBoundaryReviewTests|FullyQualifiedName~DecodeBudgetSnapshotTests' \
      --logger 'console;verbosity=minimal'
