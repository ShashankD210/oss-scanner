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
framework-dependent Windows executable and portable PDBs to `/src/dist`. For the
native DLL, it runs genuine Microsoft `rc.exe`, `cl.exe`, and `link.exe` under
64-bit Wine. Wine is only a build-tool host: neither the converter nor the
extension is executed, loaded, or registered.

The source includes Microsoft's `shobjidl_core.h`, which the previous MinGW-w64
environment did not provide. Substituting headers or omitting the native build
would not audit the intended SDK/COM semantics. Clang's `/W4` also enables
additional diagnostics for the unchanged source (including its partial
`STARTUPINFOW` initializer), so using genuine MSVC preserves the upstream
`scripts/build.ps1` flags `/W4 /WX /EHsc /O2 /MT /DUNICODE /D_UNICODE /LD`
without disabling diagnostics or changing application source. The static CRT
choice `/MT`, original `.def` exports, and original icon/version resources are
preserved. The SDK's `uuid.lib` supplies the COM IID definitions.

The native analysis build adds `/Z7` and `/DEBUG:FULL` to retain CodeView object
information and `/src/dist/HeicToJpegShellExt.pdb`; this is not a signed MSIX
release or a runtime correctness claim. Native PDB source paths use Wine's
`Z:\src\...`, which maps to the actual retained `/src/...` checkout. Source,
Microsoft compiler/SDK/CRT files, their original unpacked contents, package
cache, Wine prefix, .NET targeting packs, `/opt/nuget/packages`, and build outputs
remain available after networking is removed. No `install.sh` header rewriting,
compiler patching, header shim, or replacement native implementation is used.

### Online downloads and third-party terms

All downloads and dependency installation happen during the online Docker
build, not in the offline rebuild:

- The existing Microsoft .NET 10 SDK image, WindowsDesktop reference packs, and
  NuGet restore retain their own terms.
- Ubuntu supplies `wine64`, `python3`, and `msitools` (with its cabinet/MSI
  dependencies), along with `curl`, CA certificates, and Git. The Wine package's
  license/copyright notices remain under `/usr/share/doc`.
- The ISC-licensed `vsdownload.py` from
  [mstorsjo/msvc-wine](https://github.com/mstorsjo/msvc-wine/tree/514f8ea34842cd6d831804d0e9658d3a32870ae1)
  is pinned to commit `514f8ea34842cd6d831804d0e9658d3a32870ae1` and verified
  against SHA-256
  `278429ffd7ec3aa0080ed0438fe241bb73795555949970cc569db68a60cc386b`.
  Its copyright/license remains in `/opt/msvc-wine/vsdownload.py`.
- An immutable
  [Microsoft VS 2022 package manifest](https://download.visualstudio.microsoft.com/download/pr/bc92e2cb-33de-4a0c-995d-efa817f16b16/6e470016e4324c84c255ffd0beb3767d17ec89cc8561e9409ee3e1f6d29400f5/VisualStudio.vsman)
  is verified against SHA-256
  `6e470016e4324c84c255ffd0beb3767d17ec89cc8561e9409ee3e1f6d29400f5`.
  It pins the servicing package URLs and hashes for MSVC 17.14/14.44 and Windows
  SDK `10.0.26100` (installed SDK layout `10.0.26100.0`, compiler layout
  `14.44.35207`). Original Microsoft VSIX/MSI/CAB downloads and notices are kept
  in `/opt/msvc-cache` and `/opt/msvc`; the manifest remains in `/opt/msvc-wine`.
- `--accept-license` accepts the
  [Microsoft Build Tools license](https://go.microsoft.com/fwlink/?LinkId=2179911);
  the downloaded Windows SDK and CRT also retain their applicable Microsoft
  license/EULA and redistribution restrictions. The original-code MIT grant
  does not relicense any of these files. **Do not publish or redistribute this
  analysis image or its SDK/toolchain cache.** Distribution of compiled runtime
  components must separately comply with Microsoft's redistributable terms.

Offline managed rebuild (run inside the built image):

```sh
dotnet publish src/Heic2Jpg/Heic2Jpg.csproj -c Release -r win-x64 \
  --self-contained false --no-restore -p:EnableWindowsTargeting=true \
  -p:DebugType=portable -p:DebugSymbols=true -o /src/dist
```

Offline native rebuild (also inside the image):

```sh
mkdir -p /src/obj/ShellExt /src/dist
cd /src/src/ShellExt
/usr/lib/wine/wine64 "$SDK_BIN/rc.exe" \
  /nologo /fo Z:/src/obj/ShellExt/ShellExt.res ShellExt.rc
/usr/lib/wine/wine64 "$MSVC_BIN/cl.exe" \
  /nologo /W4 /WX /EHsc /O2 /MT /DUNICODE /D_UNICODE /LD /Z7 \
  dllmain.cpp Z:/src/obj/ShellExt/ShellExt.res /FoZ:/src/obj/ShellExt/ \
  /link /DEF:ShellExt.def /OUT:Z:/src/dist/HeicToJpegShellExt.dll \
  /IMPLIB:Z:/src/obj/ShellExt/ShellExt.lib \
  /DEBUG:FULL /PDB:Z:/src/dist/HeicToJpegShellExt.pdb \
  /INCREMENTAL:NO /OPT:REF /OPT:ICF \
  ole32.lib shlwapi.lib user32.lib uuid.lib
```

There is no portable test project or repository Linux test suite. In particular,
Linux cannot execute the WPF/WIC pipeline, load this Windows DLL, obtain the
Windows Store HEIF codec, or exercise Explorer. Running the application under
Wine or a substituted decoder would not establish correctness of the intended
runtime. Do not call a Linux `dotnet` execution failure a conversion test, and do
not omit managed/native source audit because execution is unavailable. MSIX
packing with Windows SDK `makeappx.exe` is not performed in this container.

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

Framework/WIC/HEIF codecs, Microsoft SDK/CRT/compiler packages, Wine, and .NET
build dependencies retain their own licenses; the project's original-code MIT
grant does not relicense them. No vendored third-party codec/source or existing
third-party source notice was identified. Binary icon/logo provenance is not
documented in the checkout;
do not infer an external artwork grant. Findings should distinguish an
application flaw from a platform decoder defect and identify the affected
platform prerequisite. Do not contact live services, copy private credentials,
run the installer/uninstaller, or perform certificate/package/Explorer changes.
