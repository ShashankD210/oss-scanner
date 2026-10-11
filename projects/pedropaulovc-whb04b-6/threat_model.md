# Threat model

## Project and scope

This is Pedro's public MIT reverse-engineered WHB04B-6 USB pendant protocol/reference application. Authenticated GitHub metadata identifies a non-fork repository with default branch `main`; audited head on 2026-10-11 is `472ab010acda3507fe6052d88965d71bd74b0cf8`. Reports go to Pedro, not the hardware vendor or maintainers of HidSharp/CNCjs. This small reference project makes no claim of broad adoption, vendor endorsement, or upstream maintainer representation.

The sole application project is `src/WHB04BClient.csproj`: a portable `net8.0` console executable using HidSharp 2.1.0 and Microsoft.Extensions logging/DI 9.0.7. There are no Node projects, solution files, test projects, test frameworks, or committed NuGet lockfiles in the inspected tree. Relevant owned code includes `src/HidCommunication.cs`, `src/HidCommunicationException.cs`, `src/PendantInputData.cs`, `src/PendantDisplayData.cs`, `src/WHB04BClient.cs`, and `src/Program.cs`.

Vendor DLLs, documentation, and split archives in `vendor/` are documented as copyrighted by Chengdu Xinhecheng Technology Co.,Ltd., retained for backup. The repository's MIT source license does not relicense them. The application uses HidSharp directly and does not link the vendor PHB04B DLL. Keep the complete checkout for review, but do not execute vendor binaries, install vendor drivers, or flash firmware. Documentation/images and release workflows are lower priority than the live protocol code.

## Untrusted input and boundaries

USB/HID data must be treated as untrusted even when a device matches the pendant VID/PID. The main surfaces are HID input buffers, report/header recognition, lengths, key/dial enum values, signed jog conversion, malformed/repeated reports, device reconnect/disconnect, timeout/error handling, and output LCD packet serialization. Device discovery does not authenticate packet contents.

`HidCommunication.ReadInput()` reads an eight-byte report with a 100 ms timeout. `PendantInputData` requires at least eight bytes and interprets the jog byte as `sbyte`. `WHB04BClient.HasDataChanged()` checks header `0x04` and conditionally filters invalid dial packets; inspect that state/filter logic and whether shorter reads or unexpected values propagate. Test malformed lengths, unknown enum values, signed extremes, reconnect transitions, and coordinate/display serialization using synthetic buffers against the real parser classes. Do not presume a checksum is validated merely because these are HID reports.

HidSharp/native OS HID APIs are a separate dependency/OS boundary; trace memory-corruption allegations to the exact reachable layer. The application opens physical devices and writes a 21-byte LCD payload as three report blocks. Constructor initialization and polling are not safe inert test setup. Device permissions, exclusive access, native library availability, and hardware timing are outside a no-device parser test's proof.

This standalone reference application does not authenticate CNCjs, store a CNCjs session token, or issue CNC motion commands. Its input events can be reused by a downstream motion controller, but the CNCjs credential-to-actuation mapping lives in the separate `pedropaulovc/cncjs-pendant-whb04b-6` enrollment. Do not invent a CNC authorization layer or assign that widget's behavior to this executable. USB output still has real device effects; LCD writes and mode updates must not be performed on hardware during scanning. Reports about a potential downstream physical-actuation impact must explain the additional integration assumptions and distinguish them from a demonstrated local parser issue.

## Offline environment and safe exercise

The Dockerfile copies the complete checkout to `/src`, installs .NET 8 SDK/runtime and Linux HID native dependencies, restores all NuGet packages online into `/opt/nuget/packages`, and builds Debug output at `/opt/whb04b/linux`. It also produces a framework-dependent Windows x64 review artifact at `/opt/whb04b/win-x64` and restores the ordinary project's assets again for offline `dotnet build src/WHB04BClient.csproj --no-restore -c Debug`. No real credentials, devices, serial interfaces, or machine endpoints are supplied.

There is no existing synthetic HID/protocol test suite to run. Construct hardware-free in-process tests directly around `PendantInputData` and `PendantDisplayData` and pure serialization/parsing logic, without constructing `WHB04BClient` or `HidCommunication` where initialization opens a device. A temporary test driver can reuse the preloaded SDK and NuGet cache; do not fetch new packages offline. Capture any generated output bytes in memory only. This is parser/serializer exercise, not emulated hardware integration.

Do not execute the console application: `Program.cs` initializes the pendant, clears its LCD, and writes demonstration display data periodically. Do not expose `/dev/hidraw`, USB devices, serial ports, host credentials, or real machine network access. Never send actual HID output or CNC commands. Source review and compilation do not prove HID integration, OS device permissions, reconnect timing, or physical safety.

The project targets ordinary .NET 8 rather than a Windows-only TFM; Linux runtime supports hardware-free managed parser testing. Windows x64 publishing is cross-compilation/review evidence only, with no Windows runtime/device verification. Linux HID integration is also deliberately untested. The release workflow publishes Linux and Windows artifacts, but neither platform claim should be inferred from a container build alone.

## Severity guidance

Untrusted HID parsing that permits controlled memory corruption/code execution is high or critical according to demonstrated reach/control. Crashes, unbounded resource use, polling failures, or persistent inability to read/display are generally medium. Wrong dial/key/coordinate interpretation can have downstream safety relevance, but escalate severity only with a supported path into an actual control integration; this application itself is a display/input reference, not a CNC safety controller. Treat intended local device access, authorized display updates, and ordinary HID permissions as design behavior unless an additional exploitable boundary violation is shown.

## Leave alone

No hardware attachment, real tokens, vendor tools, driver installation, firmware updates, CNCjs production sessions, or physical actuation. Report owned-source flaws with offline synthetic reproductions and explicit platform/evidence limits. Clearly attribute dependency/vendor findings instead of presenting Pedro as their upstream maintainer.
