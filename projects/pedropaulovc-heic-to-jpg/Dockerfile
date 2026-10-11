FROM mcr.microsoft.com/dotnet/sdk:10.0

ENV DEBIAN_FRONTEND=noninteractive \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    DOTNET_NOLOGO=1 \
    NUGET_PACKAGES=/opt/nuget/packages

# The native Explorer COM DLL is cross-compiled, never loaded or registered.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git g++-mingw-w64-x86-64 binutils-mingw-w64-x86-64 \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# Preserve the WindowsDesktop targeting/reference packs and NuGet cache for
# offline re-builds. Match the repository's framework-dependent win-x64 publish.
RUN dotnet restore src/Heic2Jpg/Heic2Jpg.csproj \
      -r win-x64 -p:EnableWindowsTargeting=true -p:SelfContained=false \
    && dotnet publish src/Heic2Jpg/Heic2Jpg.csproj \
      -c Release -r win-x64 --self-contained false --no-restore \
      -p:EnableWindowsTargeting=true -p:DebugType=portable -p:DebugSymbols=true \
      -o /src/dist

# Compile the real shell-extension source and its resources for Windows x64.
# MinGW replaces the Windows-only MSVC build driver, not any application code.
RUN mkdir -p /src/obj/ShellExt \
    && x86_64-w64-mingw32-windres \
      -I /src/src/ShellExt /src/src/ShellExt/ShellExt.rc \
      -O coff -o /src/obj/ShellExt/ShellExt.res.o \
    && x86_64-w64-mingw32-g++ \
      -std=c++17 -O2 -g -fno-omit-frame-pointer \
      -DUNICODE -D_UNICODE -D_WIN32_WINNT=0x0A00 -DWINVER=0x0A00 \
      -shared -static -static-libgcc -static-libstdc++ \
      /src/src/ShellExt/dllmain.cpp /src/src/ShellExt/ShellExt.def \
      /src/obj/ShellExt/ShellExt.res.o \
      -o /src/dist/HeicToJpegShellExt.dll \
      -Wl,--out-implib,/src/obj/ShellExt/HeicToJpegShellExt.a \
      -lole32 -lshlwapi -luser32 -luuid

# There are no portable repository tests. WPF/WIC, the HEIF codec, Explorer,
# and MSIX tooling require Windows; do not replace them with a Linux converter.
CMD ["/bin/bash"]
