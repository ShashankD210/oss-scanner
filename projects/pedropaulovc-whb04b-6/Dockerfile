FROM mcr.microsoft.com/dotnet/sdk:8.0-bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    NUGET_PACKAGES=/opt/nuget/packages
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git python3 libudev1 libusb-1.0-0 \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
# Restore online once and keep the full NuGet cache for offline source work.
# net8.0 is a portable console target, not a Windows-only TFM.
RUN dotnet restore src/WHB04BClient.csproj \
    && dotnet build src/WHB04BClient.csproj --no-restore \
       -c Debug -o /opt/whb04b/linux
# Windows-targeted compilation is review evidence only, not HID runtime proof.
RUN dotnet restore src/WHB04BClient.csproj -r win-x64 \
    && dotnet publish src/WHB04BClient.csproj --no-restore \
       -r win-x64 --self-contained false -c Debug -o /opt/whb04b/win-x64 \
    && dotnet restore src/WHB04BClient.csproj
# The application is a hardware demo, not an isolated test runner.
# Do not execute it during build or attach devices at audit time.
CMD ["/bin/bash"]
