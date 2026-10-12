# Threat model

A macOS GUI for MacPorts with a privileged helper that runs port commands as root. The helper's XPC interface is the main attack surface: any local process may try to call it. High: any way for an unprivileged caller to run arbitrary commands or write arbitrary files as root through the helper. Out of scope: MacPorts itself.
