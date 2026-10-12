# Threat model

A Decky Loader plugin on SteamOS. The Python backend runs as root and is called only by the local Decky UI. Treat share names, paths and mount options read from config files or systemd units as untrusted when they reach commands. Out of scope: anything needing root already. High: command injection or path traversal reachable from a crafted share or unit name.
