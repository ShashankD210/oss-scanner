# HASS.Agent .NET10: a Windows app (C#, .NET 10, WinForms tray app and the same exe as a Windows service).
# It targets net10.0-windows, so on Linux it can be compiled (EnableWindowsTargeting) but not run; the unit tests
# need Windows to run as well. The build gives the scanner the compiled assemblies and a checked project graph.
FROM mcr.microsoft.com/dotnet/sdk:10.0
ENV DEBIAN_FRONTEND=noninteractive \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_NOLOGO=1 \
    NUGET_XMLDOC_MODE=skip
RUN apt-get update && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Every NuGet package (the app and the tests) is fetched now; the audit runs offline.
RUN dotnet restore src/HASS.Agent.NET10/HASS.Agent.NET10.csproj -p:EnableWindowsTargeting=true \
    && dotnet restore tests/HASS.Agent.NET10.Tests/HASS.Agent.NET10.Tests.csproj -p:EnableWindowsTargeting=true

# The app, as its maintainer builds it (warnings are errors in this project), with full debug info.
RUN dotnet build src/HASS.Agent.NET10/HASS.Agent.NET10.csproj -c Debug --no-restore -p:EnableWindowsTargeting=true

# The xUnit tests compile here but only run on Windows; a failure to build them does not stop the image.
RUN dotnet build tests/HASS.Agent.NET10.Tests/HASS.Agent.NET10.Tests.csproj -c Debug --no-restore -p:EnableWindowsTargeting=true \
    || echo "WARNING: the test project did not build"
