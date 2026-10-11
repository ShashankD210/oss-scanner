FROM mcr.microsoft.com/dotnet/sdk:10.0

ENV DEBIAN_FRONTEND=noninteractive \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    DOTNET_NOLOGO=1 \
    NUGET_PACKAGES=/opt/nuget/packages

# Run genuine Microsoft x64 build tools under Wine, never the application/DLL.
# MinGW lacks shobjidl_core.h; clang-cl /W4 /WX enables additional diagnostics
# that reject this unchanged MSVC source. Keep the upstream compiler semantics.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 msitools wine64 \
    && rm -rf /var/lib/apt/lists/*

# Download/extract the original vendor packages without header fixes or binary
# patches. The immutable VS manifest pins every package URL and SHA-256.
# --accept-license accepts Microsoft's terms, not the project's MIT license.
# This image contains non-redistributable SDK/toolchain files: do not publish it.
RUN mkdir -p /opt/msvc-wine /opt/msvc-cache \
    && curl --fail --location --silent --show-error \
      https://raw.githubusercontent.com/mstorsjo/msvc-wine/514f8ea34842cd6d831804d0e9658d3a32870ae1/vsdownload.py \
      --output /opt/msvc-wine/vsdownload.py \
    && printf '%s  %s\n' \
      278429ffd7ec3aa0080ed0438fe241bb73795555949970cc569db68a60cc386b \
      /opt/msvc-wine/vsdownload.py | sha256sum --check - \
    && curl --fail --location --silent --show-error \
      https://download.visualstudio.microsoft.com/download/pr/bc92e2cb-33de-4a0c-995d-efa817f16b16/6e470016e4324c84c255ffd0beb3767d17ec89cc8561e9409ee3e1f6d29400f5/VisualStudio.vsman \
      --output /opt/msvc-wine/VisualStudio.vsman \
    && printf '%s  %s\n' \
      6e470016e4324c84c255ffd0beb3767d17ec89cc8561e9409ee3e1f6d29400f5 \
      /opt/msvc-wine/VisualStudio.vsman | sha256sum --check - \
    && python3 /opt/msvc-wine/vsdownload.py \
      --accept-license --major 17 --msvc-version 17.14 --sdk-version 10.0.26100 \
      --manifest /opt/msvc-wine/VisualStudio.vsman \
      --host-arch x64 --architecture x64 \
      --with-default no --with-msvc yes --with-sdk yes \
      --with-asan no --with-atl no \
      --only-unpack --skip-patch --cache /opt/msvc-cache --dest /opt/msvc

# Wine's Z: drive maps /, including the retained /src source paths in the PDB.
# Explicit paths also avoid relying on vcvars, wrappers, or SDK header rewrites.
ENV WINEARCH=win64 \
    WINEPREFIX=/opt/wineprefix \
    WINESERVER=/usr/lib/wine/wineserver64 \
    WINEPATH=Z:/opt/msvc/VC/Tools/MSVC/14.44.35207/bin/Hostx64/x64 \
    MSVC_BIN=/opt/msvc/VC/Tools/MSVC/14.44.35207/bin/Hostx64/x64 \
    SDK_BIN="/opt/msvc/Windows Kits/10/bin/10.0.26100.0/x64" \
    INCLUDE="Z:/opt/msvc/VC/Tools/MSVC/14.44.35207/include;Z:/opt/msvc/Windows Kits/10/Include/10.0.26100.0/ucrt;Z:/opt/msvc/Windows Kits/10/Include/10.0.26100.0/shared;Z:/opt/msvc/Windows Kits/10/Include/10.0.26100.0/um;Z:/opt/msvc/Windows Kits/10/Include/10.0.26100.0/winrt" \
    LIB="Z:/opt/msvc/VC/Tools/MSVC/14.44.35207/lib/x64;Z:/opt/msvc/Windows Kits/10/Lib/10.0.26100.0/ucrt/x64;Z:/opt/msvc/Windows Kits/10/Lib/10.0.26100.0/um/x64"

RUN /usr/lib/wine/wine64 wineboot --init \
    && /usr/lib/wine/wineserver64 -w

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

# Compile the unchanged source/resources with scripts/build.ps1's release flags.
# /Z7 adds embedded CodeView information; /DEBUG:FULL emits real native PDBs
# without requiring a compiler PDB server. uuid.lib supplies the SDK COM IIDs.
RUN mkdir -p /src/obj/ShellExt \
    && cd /src/src/ShellExt \
    && /usr/lib/wine/wine64 "$SDK_BIN/rc.exe" \
      /nologo /fo Z:/src/obj/ShellExt/ShellExt.res ShellExt.rc \
    && /usr/lib/wine/wine64 "$MSVC_BIN/cl.exe" \
      /nologo /W4 /WX /EHsc /O2 /MT /DUNICODE /D_UNICODE /LD /Z7 \
      dllmain.cpp Z:/src/obj/ShellExt/ShellExt.res \
      /FoZ:/src/obj/ShellExt/ \
      /link /DEF:ShellExt.def /OUT:Z:/src/dist/HeicToJpegShellExt.dll \
      /IMPLIB:Z:/src/obj/ShellExt/ShellExt.lib \
      /DEBUG:FULL /PDB:Z:/src/dist/HeicToJpegShellExt.pdb \
      /INCREMENTAL:NO /OPT:REF /OPT:ICF \
      ole32.lib shlwapi.lib user32.lib uuid.lib

# There are no portable repository tests. WPF/WIC, the HEIF codec, Explorer,
# and MSIX tooling require Windows; do not replace them with a Linux converter.
CMD ["/bin/bash"]
