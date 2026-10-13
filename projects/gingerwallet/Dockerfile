FROM mcr.microsoft.com/dotnet/sdk:8.0-bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    DOTNET_NOLOGO=1 \
    NUGET_PACKAGES=/root/.nuget/packages

# Keep the SDK, native wallet dependencies and a virtual display available to
# the scanner. GUI tests can then run without an interactive desktop.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 curl bash procps xvfb xauth \
      libx11-6 libice6 libsm6 libfontconfig1 libglib2.0-0 \
      libxext6 libxrender1 libxi6 libxrandr2 libxcursor1 libxfixes3 \
      libgl1 libegl1 libusb-1.0-0 libudev1 libevent-2.1-7 \
      fonts-dejavu-core fonts-liberation \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# Fetch all NuGet dependencies while the build has network access. Preserve
# source, build output and the package cache for subsequent offline analysis.
RUN dotnet restore WalletWasabi.sln --locked-mode
RUN dotnet build WalletWasabi.sln --no-restore --configuration Release \
      -p:DebugType=portable -p:DebugSymbols=true
RUN xvfb-run -a dotnet test WalletWasabi.Tests/WalletWasabi.Tests.csproj \
      --no-restore --no-build --configuration Release --filter "UnitTests" \
      --logger "console;verbosity=minimal" \
      --blame-hang --blame-hang-timeout 5m

CMD ["bash"]
