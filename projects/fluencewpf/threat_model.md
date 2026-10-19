# Threat model

Read the upstream `SECURITY.md` first. Fluence.Wpf is a Windows WPF control and
theming library, with an optional PowerShell module. It runs inside the consuming
application or PowerShell host, with that process's privileges. It is not a web
service or an authentication boundary. The current upstream security policy
covers both the library and module, with fixes considered for `main` and the
latest tagged release.

## What this project does and where untrusted input enters

- A legitimate application may bind data of lower trust to controls. In
  particular, `HyperlinkButton.NavigateUri` reaches shell navigation on click
  (`Fluence.Wpf/Controls/HyperlinkButton.cs`). Assess whether a malicious URI
  supplied through ordinary application data can start an unsafe handler or
  execute a program under the host's privileges. State the binding and user
  interaction required; the property is also intentionally settable by trusted
  application code.
- User-entered text, selections, paths, and image bytes can reach WPF controls
  and PowerShell prompts. The PowerShell module accepts file and `pack:` image
  sources, rejects other URI schemes in `Resolve-FluenceImageSource.ps1`, and
  hands an accepted source to WPF image loading in `Set-FluenceWindowIcon.ps1`.
  Follow the actual input path before attributing a decoder issue to Fluence.
  `Image.Source` and `PersonPicture.ProfilePicture` are WPF `ImageSource`
  properties, not custom Fluence image or SVG parsers.
- Windows messages, monitor/taskbar state, DWM results, and per-user theme
  registry values influence window layout and rendering. `FluenceWindow` calls
  `NativeMethods` for Win32/DWM operations; `RegistryHelper` reads `HKCU`
  theme/accent state. Investigate malformed or adversarial state only where it
  crosses into unsafe interop, an unexpected privilege boundary, or a material
  availability failure. Most desktop state is under the same user's control.
- Module import dot-sources scripts under its installed `Private/` and `Public/`
  folders and loads DLLs from its staged `lib/<target framework>/` folder
  (`Fluence.Wpf.PowerShell.psm1`, `Get-FluenceLibraryPath.ps1`,
  `Import-FluenceLibrary.ps1`). This is a package integrity boundary: assess
  dependency confusion, unexpected assembly resolution, and load order only
  with a realistic attacker who can affect the package or its deployment while
  the host still treats it as trusted. Merely replacing an already writable
  module script or DLL does not establish a new vulnerability.
- `Show-FluenceWindow` accepts `-Xaml`, `-XamlPath`, `-Content`, and `-Initialize`.
  Its own documentation says XAML and callbacks are trusted code, and
  `Set-FluenceWindowContent.ps1` uses `XamlReader.Parse` or `XamlReader.Load`.
  A caller that deliberately executes attacker-controlled XAML or a PowerShell
  scriptblock has crossed the caller's trust boundary before reaching this API;
  there is no promised sandbox. Still inspect bugs where lower-trust data in a
  legitimate trusted window crosses into code, secrets, files, or process
  execution unexpectedly.

## Components that matter most / least

- Prioritize `Fluence.Wpf/Native/`, `Fluence.Wpf/Controls/FluenceWindow.cs`,
  and related window policy code for handle lifetime, pointer/structure size,
  integer conversion, message handling, and Win32/DWM API contracts. The
  handwritten native imports in `NativeMethods.cs` specify
  `DefaultDllImportSearchPaths(System32)`; check generated CsWin32 calls and
  actual call sites as well before alleging DLL search-path exposure.
- Prioritize `Fluence.Wpf/Controls/HyperlinkButton.cs` when data-provided links
  reach the shell, and the PowerShell module's `Public/` entry points and
  `Private/` helpers for data-to-script, path-to-file, image-loading, runspace,
  and assembly-loading boundaries. Password prompts normally return
  `SecureString`; `-AsPlainText` and plaintext defaults are explicit options
  (`New-FluencePrompt.ps1`, `Get-FluenceInput.ps1`). Check a claimed secret leak
  against those documented choices.
- Theme XAML/resources and ordinary controls matter when they provide a
  demonstrable route from lower-trust data to security impact. Demo apps,
  examples, tests, generated API docs, and screenshot scripts are mainly
  reproducer material; report a defect there when it affects the shipped
  library/module, its package, or a documented security property.
- The project targets .NET Framework 4.7.2, .NET 8 for Windows, and .NET 10 for
  Windows. Check target-specific behavior and whether a proposed patch works
  across the supported targets. Distinguish a Fluence bug from a flaw in WPF,
  Windows, PowerShell, or a third-party package.

## How to exercise it

- Scanner source is at `/src`. The Linux image can restore/build the managed
  Windows-targeted library for static analysis, but it cannot run WPF UI,
  Windows PowerShell 5.1, Win32/DWM, or desktop interaction. A successful Linux
  build is not runtime validation of those paths. Use the source and tests there
  for code tracing and focused non-UI checks where feasible.
- Reproduce UI, shell-navigation, native interop, and PowerShell module
  findings on Windows with the affected .NET target or PowerShell host. The
  canonical commands are in upstream `CONTRIBUTING.md`. The module is outside
  the solution: build/stage the library with
  `Fluence.Wpf.PowerShell.Module/build/Build-Module.ps1`, then run
  `Test-Module.ps1`. UI tests need an interactive desktop; PowerShell 5.1 tests
  need `powershell.exe -STA`. Keep test files, image inputs, XAML, and temporary
  module copies inside a disposable test environment.
- A report should identify the source revision, affected framework/host and
  Windows version, attacker-controlled input and prerequisite, precise trust
  boundary, minimal safe reproducer, expected versus observed behavior, and
  impact. For a crash, include exception or process-termination evidence. For
  native memory corruption, separate observed corruption from a claim of code
  execution. Include a regression test and a patch that preserves the public
  API and supported targets when possible.

## How you rate severity

- Critical: demonstrated arbitrary code execution or equivalent compromise of
  a higher-trust host from lower-trust data with realistic, common deployment
  prerequisites; or reliably exploitable native corruption with the same
  consequence. Explain any click or host behavior required.
- High: a credible cross-privilege file/secret compromise, unsafe process
  execution through a legitimate data path, or reachable native memory safety
  defect with plausible exploitation. Do not automatically cap native memory
  corruption; rate it from reachability, control, mitigations, and evidence.
- Medium: meaningful disclosure or integrity impact confined to the current
  user's process, or repeatable process crash/resource exhaustion caused by
  lower-trust data in a realistic application workflow.
- Low or non-security: weak impact, unusual prerequisites, same-user
  self-inflicted failures, or rendering/usability defects. Severity depends on
  the actual boundary and deployment, not on a dangerous API name alone.

## Anything to leave alone

- Do not present deliberate loading of untrusted XAML/scriptblocks as a
  sandbox escape: the API documents these as trusted code. Do not exclude
  attacker-controlled ordinary data processed by a trusted UI merely because
  the surrounding XAML or callback was authored by the application.
- Do not report registry reads, System32 native imports, `Assembly.LoadFrom`,
  `XamlReader`, `Process.Start`, `SecureString`, or `BitmapFrame` by pattern
  match alone. Demonstrate a reachable attack path and meaningful impact, and
  account for the code's explicit checks and the caller's chosen permissions.
- Do not treat a Linux-only failure to execute WPF or a generated analyzer
  warning as a runtime vulnerability. Reproduce Windows-specific claims on
  Windows before making definitive impact claims.
- Coordinate sensitive findings through the upstream `SECURITY.md` private
  reporting route. Do not put exploit details or secrets in a public issue.
