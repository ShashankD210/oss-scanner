FROM mcr.microsoft.com/dotnet/sdk:8.0-bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    NUGET_PACKAGES=/opt/nuget/packages \
    npm_config_cache=/opt/npm-cache
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git nodejs npm python3 chromium libudev1 libusb-1.0-0 \
    && rm -rf /var/lib/apt/lists/*
COPY . /src
WORKDIR /src
# The current root package is a static ES-module WebHID app, with no npm
# dependencies or compilation step. Keep a populated npm cache if dependencies
# are added later; do not start the server, request HID devices, or connect CNCjs.
RUN npm install --include=dev --ignore-scripts --no-audit --no-fund
# The widget normally gets its Socket.IO client from the external CNCjs host.
# Cache its expected 2.x client for isolated analysis, not a CNCjs server
# substitute. The source does not pin the host's exact Socket.IO version.
RUN npm install --prefix /opt/socketio-client-v2 --ignore-scripts --no-audit --no-fund socket.io-client@2.5.0
# The checkout also contains the complete net8.0 C# reference application.
# Fetch all NuGet dependencies online; retain the package cache for offline work.
RUN dotnet restore whb04b-6/src/WHB04BClient.csproj \
    && dotnet build whb04b-6/src/WHB04BClient.csproj --no-restore \
       -c Debug -o /opt/whb04b-reference/linux
# Produce a Windows-targeted artifact for source/binary review only. This is not
# evidence that Windows HID integration works in a Linux scanner VM.
RUN dotnet restore whb04b-6/src/WHB04BClient.csproj -r win-x64 \
    && dotnet publish whb04b-6/src/WHB04BClient.csproj --no-restore \
       -r win-x64 --self-contained false -c Debug -o /opt/whb04b-reference/win-x64 \
    && dotnet restore whb04b-6/src/WHB04BClient.csproj
# No automated hardware-free suite is declared. Never run Program.cs: it opens
# a real pendant and emits display traffic. Leave parser sources and outputs for
# hardware-free in-process inspection; no devices or credentials are provided.
CMD ["/bin/bash"]
