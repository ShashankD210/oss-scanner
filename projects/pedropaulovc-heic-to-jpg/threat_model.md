# Threat model: heic-to-jpg

## Purpose and trust boundaries

This is a Windows 11 HEIC/HEIF-to-JPEG command-line converter with an Explorer
IExplorerCommand COM extension. The managed application targets
`net10.0-windows` and uses WPF imaging backed by Windows Imaging Component
(WIC); the separately installed Microsoft HEIF Image Extensions supplies the
HEIF decoder. It does not contain a Linux converter or a bundled image codec.
There is no server, authentication service, or production endpoint to contact.

Treat image bytes, embedded metadata, file names, directory names, and Explorer
selections as untrusted. Users intentionally choose local inputs and may choose
`--outdir`; this is a desktop application operating with the caller's filesystem
permissions, not a sandbox or a privileged conversion service. Review all
tracked source, including the native shell extension and packaging/install
scripts, even where Linux cannot execute it.

## Important code and behaviors to audit

- `src/Heic2Jpg/Program.cs`: command-line parsing, quality range 1–100,
  `--outdir` handling, unknown arguments interpreted as input paths, existence
  checks, error reporting, and per-input processing. Existing files are passed
  to the decoder without extension/magic-byte or size validation. The default
  output directory is the input's full-path parent; an explicit output directory
  is created and is not restricted to a sandbox.
- `src/Heic2Jpg/Converter.cs`: `BitmapDecoder.Create` consumes untrusted bytes
  with `BitmapCacheOption.OnLoad`, then uses the first frame, converts to Bgr24,
  and JPEG-encodes it. Audit image dimensions/resource exhaustion, metadata
  query/rational handling, malformed metadata, stream/file lifetime, exceptions,
  and temporary-file cleanup. Selected metadata is copied best-effort; unsupported
  fields need not be preserved.
- `src/Heic2Jpg/NamingPolicy.cs` and `Converter.Claim`: existing JPEGs must never
  be overwritten, including under concurrent conversions. Candidates run from
  `<basename>.jpg` through `<basename> (1000).jpg`. A non-overwriting `File.Move`
  claims the final name atomically; collision exhaustion must fail. Encoding
  first uses a randomized `.partial` name in the output directory, opened with
  `File.Create` (not `CreateNew`). Audit preexisting temporary paths, reparse
  points/symlinks, permissions, racing directory changes, and cleanup separately
  from the final-name claim. Timestamp copying is also in scope. Ordinary writes
  to the user-requested directory are intended, but a crafted image should not
  redirect writes outside it or overwrite unrelated existing files.
- `src/Heic2Jpg/Log.cs`: best-effort append logging to the caller's LocalAppData
  `heic-to-jpg/convert.log`; paths and diagnostics may be sensitive or attacker
  controlled. Logging failure should not abort conversion.
- `src/ShellExt/dllmain.cpp`: native COM lifetime/refcounting, selection extraction,
  executable discovery next to the DLL, quoting/escaping paths for
  `CreateProcessW`, and batching near 30,000 command-line characters. The DLL
  launches asynchronously and does not decode images. Audit argument injection,
  path truncation, and which executable can be launched. It should not block
  Explorer waiting for conversion.
- `package/AppxManifest.xml`, resources, and `scripts/*.ps1`: review package
  permissions, signing/trust assumptions, external binary location, and quoting.
  Do not install packages, create/trust signing certificates, modify registry or
  Explorer integration, or restart Explorer during this audit.

## Build and offline exercise

The Dockerfile copies the real checkout into `/src`, restores the actual .NET 10
project for `win-x64` with `EnableWindowsTargeting=true`, and cross-publishes its
framework-dependent Windows executable and portable PDBs to `/src/dist`. It also
cross-compiles the real native COM DLL and icon/version resources with MinGW-w64
for Windows x64, retaining debug information and frame pointers. This is an
analysis build, not a claim of identical MSVC code generation or a signed MSIX
release. Source, SDK, targeting packs, `/opt/nuget/packages`, native cross-tools,
and build outputs remain available after networking is removed.

Offline managed rebuild (run inside the built image):

```sh
dotnet publish src/Heic2Jpg/Heic2Jpg.csproj -c Release -r win-x64 \
  --self-contained false --no-restore -p:EnableWindowsTargeting=true \
  -p:DebugType=portable -p:DebugSymbols=true -o /src/dist
```

Offline native rebuild (also inside the image):

```sh
x86_64-w64-mingw32-windres -I /src/src/ShellExt \
  /src/src/ShellExt/ShellExt.rc -O coff -o /src/obj/ShellExt/ShellExt.res.o
x86_64-w64-mingw32-g++ -std=c++17 -O2 -g -fno-omit-frame-pointer \
  -DUNICODE -D_UNICODE -D_WIN32_WINNT=0x0A00 -DWINVER=0x0A00 \
  -shared -static -static-libgcc -static-libstdc++ \
  /src/src/ShellExt/dllmain.cpp /src/src/ShellExt/ShellExt.def \
  /src/obj/ShellExt/ShellExt.res.o -o /src/dist/HeicToJpegShellExt.dll \
  -Wl,--out-implib,/src/obj/ShellExt/HeicToJpegShellExt.a \
  -lole32 -lshlwapi -luser32 -luuid
```

There is no portable test project or repository Linux test suite. In particular,
Linux cannot execute the WPF/WIC pipeline, load this Windows DLL, obtain the
Windows Store HEIF codec, or exercise Explorer. Wine or a substituted decoder
would not establish correctness of the intended runtime. Do not call a Linux
`dotnet` execution failure a conversion test, and do not omit managed/native
source audit because execution is unavailable. MSIX packing with Windows SDK
`makeappx.exe` is not performed in this container.

The real existing application test is `scripts/test-concurrent-naming.ps1`.
On a separate disposable Windows 11 22H2+ host with .NET Desktop Runtime 10,
Microsoft HEIF Image Extensions, the checkout with cross-published `dist`, and
an explicitly provided local HEIC fixture, run from the checkout root:

```powershell
.\scripts\test-concurrent-naming.ps1 -SourceHeic C:\fixtures\sample.heic -Racers 4
```

It launches real conversions into a unique temporary directory and checks
four distinct, decodable JPEGs and no leftover partial files. Supplying the
fixture explicitly avoids its fallback search through the user's Pictures.
No installer or Explorer registration is needed for this test. No HEIC fixture
is committed in the repository, so runtime coverage requires a separately
provided non-sensitive fixture and the Windows dependencies. Repeated manual
conversion into a scratch directory can also exercise quality, metadata, and
non-overwrite behavior with real input; retain unrelated files as sentinels.

## Severity and exclusions

Code execution, native memory corruption, unintended executable launch, or
arbitrary overwrite/deletion beyond the intended output directory is high or
critical depending on exploitability. Loss of preexisting files violates the
explicit no-clobber goal. Assess crashes, disk/memory/CPU exhaustion and hangs
based on realistic user-triggered local exposure; this is not a network-facing
service. Expected rejection of unsupported or corrupt images is not a bug.

Framework/WIC/HEIF codecs and MinGW/.NET build dependencies retain their own
licenses; the project's original-code MIT grant does not relicense them. No
vendored third-party codec/source or existing third-party source notice was
identified. Binary icon/logo provenance is not documented in the checkout;
do not infer an external artwork grant. Findings should distinguish an
application flaw from a platform decoder defect and identify the affected
platform prerequisite. Do not contact live services, copy private credentials,
run the installer/uninstaller, or perform certificate/package/Explorer changes.
